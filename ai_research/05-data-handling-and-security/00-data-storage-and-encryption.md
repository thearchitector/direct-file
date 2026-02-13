# Data Storage and Encryption

## Data Categories

DirectFile handles several categories of sensitive data, each with distinct storage and security requirements:

### 1. User Information
- **What**: Links between external authentication provider IDs and internal DirectFile user IDs, plus basic user preferences
- **Where stored**: PostgreSQL `users` table
- **Sensitivity**: Contains the mapping between an anonymous external ID and the user's tax records
- **Key design choice**: DirectFile asks users for personal information directly rather than pulling it from the auth provider. This avoids burdening users with IAL2 identity proofing requirements just to retrieve info the auth provider might have. The trade-off is that users must enter info twice (once in the auth system, once in DirectFile), which requires careful UX messaging.

### 2. Unfiled Tax Information (Active Fact Graph)
- **What**: The user's in-progress tax return data — the fact graph JSON blob containing all writable and derived facts
- **Where stored**: Encrypted JSON blob in the PostgreSQL `taxreturns` table
- **Sensitivity**: Contains PII (names, SSNs, addresses) and FTI (income amounts, deductions, credits)
- **Why a blob**: See detailed rationale below

### 3. Filed Tax Information (Artifacts)
- **What**: PDFs of completed returns, XML bundles submitted to MeF, fact graph snapshots at submission time
- **Where stored**: AWS S3, encrypted using the S3 Encryption Client
- **Sensitivity**: Contains the authoritative record of what was filed — PII and FTI
- **Why S3**: These are read-only documents after filing. Storing them separately from the active database protects against accidental modification and provides a clear boundary between "working data" and "filed records"

### 4. Tax Software Schema (Fact Dictionary)
- **What**: The XML configuration files that define the fact graph schema, computation rules, and flow
- **Where stored**: Version-controlled in the git repository; loaded at application startup
- **Sensitivity**: Not sensitive (it's configuration, not user data), but changes must be tracked with who/what/when/why

### 5. Permissions
- **What**: Which users can access which tax returns (supports joint filing where two users share a return)
- **Where stored**: PostgreSQL `taxreturn_owners` table
- **Sensitivity**: Access control data — must be in the same system as the data it protects

## Why Store the Fact Graph as a JSON Blob

This is one of the most important data storage decisions in DirectFile. The active tax return data is stored as a single encrypted JSON blob in a PostgreSQL column, NOT as normalized relational tables. The reasons are:

### 1. Flexibility — No Lock-in on Update Strategy
A blob can be completely overwritten (snapshot approach) or have fields patched (delta approach). DirectFile doesn't commit to either strategy, giving the team flexibility to evolve.

### 2. Atomicity
Database operations are transactional. If a write fails, it rolls back entirely. With a document store (e.g., S3) for active data, you could end up with the database believing an update happened that the document store didn't record, creating inconsistency.

### 3. Efficiency — Single Read
Reading a user's tax data requires one database query, not dozens of JOINs across normalized tables. This is especially important because the fact graph is read on every page load.

### 4. Data Size Is Manageable
A complete fact graph JSON is not enormous (probably tens of KB). No performance concerns with storing this in a PostgreSQL column.

### 5. Single Point of Failure Is Manageable
The database is already a critical dependency. Adding a second data store (e.g., a separate document database for active data) doubles the failure points without proportional benefit.

### 6. Schema Evolution
Tax law changes annually. The fact dictionary (schema) changes with it. If the tax data were normalized across dozens of tables, every schema change would require database migrations, data backfills, and careful backwards-compatibility handling. With a blob, the schema lives in the application code, and old data can be migrated in-memory when loaded.

## Encryption Architecture

### Server-Side Envelope Encryption (Fact Graph)
DirectFile uses **envelope encryption** to protect fact graphs at rest:

```
                    ┌─────────────┐
                    │   AWS KMS   │
                    │  (Root Key) │
                    └──────┬──────┘
                           │ encrypts/decrypts
                    ┌──────▼──────┐
                    │  Data Key   │
                    │ (per-user)  │
                    └──────┬──────┘
                           │ encrypts/decrypts
                    ┌──────▼──────┐
                    │  Fact Graph │
                    │  (JSON blob)│
                    └─────────────┘

Stored in database:
┌──────────────────────────────────────────────┐
│ taxreturns table                              │
│ ┌──────────────┐  ┌────────────────────────┐ │
│ │ encrypted_key │  │ encrypted_fact_graph   │ │
│ └──────────────┘  └────────────────────────┘ │
└──────────────────────────────────────────────┘
```

The process:
1. A unique symmetric **data encryption key** is generated for each user
2. The fact graph JSON is encrypted with this data key (AES)
3. The data key itself is encrypted using a **root key** managed by AWS KMS
4. Both the encrypted fact graph and the encrypted data key are stored together in the database

To read the data:
1. Retrieve the encrypted data key and encrypted fact graph from the database
2. Call KMS to decrypt the data key using the root key
3. Use the decrypted data key to decrypt the fact graph
4. Parse the decrypted JSON into a fact graph instance

### Why Server-Side (Not Client-Side) Encryption
The team considered client-side encryption (encrypting in the browser before sending to the server) but chose server-side for these reasons:
- **Reduced implementation complexity**: Server-side encryption using KMS is well-understood and auditable
- **Meets security posture**: Protects against database breaches and unauthorized database access
- **Enables server-side operations**: The backend needs to read fact graphs for XML/PDF generation, validation, etc.
- **Timeline**: Simpler to implement before pilot launch

The trade-off is that plaintext fact graphs are visible to the API gateway and supporting services between the browser and the backend. Future layers of protection (e.g., message-level encryption) can be added as the threat model matures.

### S3 Artifact Encryption
Filed tax return artifacts (PDFs, XML) in S3 use the **Amazon S3 Encryption Client**:
- Uses a separate KMS wrapping key (distinct from the fact graph key)
- Envelope encryption: each artifact gets a unique data key
- The wrapping key is used only for artifact storage, making encryption/decryption operations independently auditable
- S3 also provides its own server-side encryption, so artifacts have two layers of protection

### Local Development Encryption
For local development, a `LOCAL_WRAPPING_KEY` environment variable provides a stand-in for KMS. This is a base64-encoded AES key generated by the `local-setup.sh` script. It's stored in the developer's shell configuration and used by the local Docker containers.

## Privacy and PII Protections

### What Constitutes PII/FTI
- **PII** (Personally Identifiable Information): Names, SSNs, addresses, dates of birth, email addresses
- **FTI** (Federal Tax Information): Income amounts, deductions, credits, tax liability, refund amounts, W-2 data, 1099 data

### Protections Implemented
1. **Encryption at rest**: All PII/FTI encrypted in the database and S3
2. **Logging safeguards**: Plaintext fact graph data must not appear in logs. Additional mitigations were implemented to prevent information disclosure through logging.
3. **No PII in URLs**: Tax data is never included in URLs or query parameters
4. **Session management**: Automatic logout after inactivity
5. **Shared computer safety**: The application is designed to be safe on shared/public computers (libraries, etc.)
6. **CORS and CSP**: Content Security Policy and Cross-Origin Resource Sharing headers prevent unauthorized access

### Exempted Code in the Open-Source Release
The public repository excludes:
- Code or data considered PII or FTI
- Sensitive But Unclassified (SBU) information
- Source code for National Security Systems
- Some functionality was removed or rewritten for the public release

## Authentication Architecture

### External Identity Provider
DirectFile does NOT handle authentication directly. Instead, it delegates to an external **Credential Service Provider (CSP)** that performs identity verification.

In production, this was an external government identity service. For local development, a **CSP Simulator** (Python/Flask application) provides fake authentication.

### Authentication Flow
1. User navigates to the CSP simulator (port 5000 locally)
2. CSP simulator presents a login form (any email + IAL2 selection in dev)
3. CSP verifies identity and returns a token to the backend
4. Backend creates/retrieves the user record linked by the external ID
5. Backend generates a session token for the user
6. Session is maintained via cookies/Redis

### Why External Authentication
- **Avoids IAL2 burden**: Identity proofing (verifying someone is who they claim to be) is complex and regulated. Delegating to a specialized provider avoids building this capability.
- **Vendor independence**: The internal user ID system means DirectFile isn't locked to a specific auth provider. The external ID is just a link.
- **PII minimization**: DirectFile only receives what it needs from the auth provider, not the full identity proofing data.

### User Identity Linking
- The `users` table stores: internal ID, external provider ID, provider type
- Tax returns are linked to users via `taxreturn_owners` (M:N to support joint filing)
- Users are NOT identified by PII (like SSN) — this protects against PII changes (SSN reassignment, etc.) and makes user records harder to identify if the database is compromised

## Data Retention Considerations

- **Active returns**: Stored as long as the user might need to edit (current tax season)
- **Filed artifacts**: Must be retained for the period required by the IRS (typically 3-7 years)
- **Version tracking**: Filed artifacts include version information for all software and schemas used, enabling replication of the filing system if needed for audit
