# System Architecture Overview

## Service Architecture

DirectFile is a distributed system composed of multiple microservices, each with a specific responsibility. The services communicate via REST APIs and AWS SQS message queues.

```
                                    ┌─────────────────┐
                                    │   CSP Simulator  │
                                    │  (Auth Proxy)    │
                                    │  Port: 5000      │
                                    └────────┬─────────┘
                                             │
                        ┌────────────────────┼────────────────────┐
                        │                    │                    │
                        ▼                    ▼                    ▼
              ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
              │   Screener      │  │   df-client      │  │   Backend API   │
              │  (Astro SSG)    │  │  (React SPA)     │  │  (Spring Boot)  │
              │  Port: 3500     │  │  Port: 3000      │  │  Port: 8080     │
              └─────────────────┘  └────────┬─────────┘  └───┬──────┬──────┘
                                            │                │      │
                                            │          ┌─────┘      └──────┐
                                            │          ▼                   ▼
                                            │  ┌──────────────┐   ┌──────────────┐
                                            │  │  PostgreSQL   │   │  LocalStack   │
                                            │  │  (directfile) │   │  (S3/SQS/KMS)│
                                            │  │  Port: 5435   │   │  Port: 4566   │
                                            │  └──────────────┘   └──────┬────────┘
                                            │                            │
                                            │                     ┌──────┴────────┐
                                            │                     │               │
                                            │              ┌──────▼──────┐ ┌──────▼──────┐
                                            │              │ MeF Submit  │ │ MeF Status  │
                                            │              │ (Spring)    │ │ (Spring)    │
                                            │              │ Port: 8083  │ │ Port: 8082  │
                                            │              └─────────────┘ └─────────────┘
                                            │
                                   ┌────────▼─────────┐
                                   │   State API      │
                                   │  (Spring Boot)   │
                                   │  Port: 8081      │
                                   └──────────────────┘

              Other Services:
              ┌──────────────┐  ┌──────────────┐
              │ Email Service│  │    Redis      │
              │ (Spring Boot)│  │  Port: 6379   │
              └──────────────┘  └──────────────┘
```

## Service Descriptions

### 1. CSP Simulator (Authentication Proxy)
- **Technology**: Python/Flask
- **Purpose**: In production, DirectFile integrates with a real Credential Service Provider (CSP) for identity verification (IAL2-level). For local development, a Flask-based simulator provides fake authentication. It acts as a reverse proxy, routing requests to either the frontend or backend based on URL path.
- **Why it exists**: The IRS requires identity-proofed authentication. DirectFile delegates this entirely to an external provider rather than building its own auth system. This avoids the burden of IAL2 identity proofing within the application itself.
- **Design decision rationale**: By not storing authentication credentials, DirectFile reduces its attack surface and compliance burden. The system links users via an external ID rather than PII.

### 2. df-client (Frontend)
- **Technology**: React 18, TypeScript, Vite, SASS, USWDS (US Web Design System)
- **Purpose**: The taxpayer-facing interview UI. This is a Single Page Application (SPA) that walks users through tax questions, collects their answers, and displays computed tax information.
- **Key sub-packages**:
  - `df-client-app/` — The main React application
  - `df-static-site/` — The screener (Astro-based static site for pre-login eligibility questions)
  - `js-factgraph-scala/` — The transpiled Scala fact graph library for use in JavaScript
  - `packages/` — Shared frontend packages (eslint plugins, etc.)
- **Why SPA**: The fact graph runs client-side, performing tax calculations in real-time as users enter data. This enables instant feedback without round-trips to the server, which is critical for a good mobile experience.

### 3. Backend API
- **Technology**: Java 21, Spring Boot, Maven
- **Purpose**: The central backend service. Handles:
  - User session management and token generation
  - Storing and retrieving fact graph data (encrypted in PostgreSQL)
  - Converting fact graph data to MeF XML format
  - Generating PDF representations of tax returns
  - Managing the submission lifecycle (submit, resubmit, track status)
  - Dispatching messages to SQS queues
  - Integrating with the Data Import system
- **Database**: PostgreSQL with Liquibase migrations
- **Why Java/Spring**: Java is the IRS's primary backend language. Spring Boot provides mature REST API tooling, SOAP/WSDL support (needed for MeF), and a large ecosystem of enterprise libraries. The hybrid Java+Scala approach uses Java for APIs and Scala for the graph/computation engine.

### 4. MeF Submit Service
- **Technology**: Java, Spring Boot
- **Purpose**: Reads tax return XML from the dispatch queue, submits it to the IRS Modernized e-File (MeF) system, and reports the result.
- **Why separate**: MeF submission involves SOAP/WSDL communication with on-premises IRS infrastructure. Isolating this into its own service keeps the main backend simple and allows independent scaling and security controls. This service is NOT included in the default docker-compose because it requires real MeF credentials.

### 5. MeF Status Service
- **Technology**: Java, Spring Boot
- **Purpose**: Polls MeF for acknowledgements (accepted/rejected status) of submitted returns. When a status change is detected, it writes to SQS queues to notify the backend.
- **Why separate**: Polling MeF is a background process that runs on a schedule. Separating it prevents the polling load from affecting the user-facing API. It has a single pod and was identified as a scaling bottleneck that the team planned to address (see Status Service 2.0 RFC).

### 6. Email Service
- **Technology**: Java, Spring Boot
- **Purpose**: SMTP relay for sending emails to taxpayers. Triggered by system events (submission confirmation, status changes, etc.) via SQS messages.
- **Profiles**: `blackhole` (logs instead of sending), `send-email` (actually sends)

### 7. State API
- **Technology**: Java, Spring Boot
- **Purpose**: Handles the transfer of federal return data to state tax filing tools. After a taxpayer authorizes the transfer, the state tool can pull the federal return data (both MeF XML and enriched JSON) through authenticated REST endpoints.
- **Database**: Separate PostgreSQL instance for authorization codes and state profiles
- **Security**: Uses JWT bearer tokens signed with state-specific certificates for authentication

### 8. Redis
- **Purpose**: Caching layer used for session data and status response caching. Reduces load on the database and status service for frequently polled data.

### 9. LocalStack
- **Purpose**: Mocks AWS services (S3, SQS, KMS, SNS, IAM, Lambda) for local development. In production, real AWS services are used.
- **Services mocked**: S3 (artifact storage), SQS (message queues), KMS (encryption key management)

## Data Flow: From User Input to Filed Return

### Step 1: User Authentication
1. User navigates to the screener, answers eligibility questions
2. If eligible, user is directed to the CSP for identity verification
3. CSP returns a verified identity token to the backend
4. Backend creates or retrieves the user record, generates a session token

### Step 2: Interview / Data Collection
1. Frontend initializes a fact graph from the JSON-serialized fact dictionary
2. User answers questions; each answer is written to the client-side fact graph
3. Fact graph automatically computes all derived facts in real-time
4. On each "Save and continue", writable facts are sent to the backend API
5. Backend encrypts and persists the fact graph blob in PostgreSQL

### Step 3: Tax Computation
1. The fact graph computes all tax values (AGI, deductions, credits, tax owed/refund)
2. Computation happens both client-side (for UI responsiveness) and server-side (for authoritative values)
3. The same Scala code runs in both environments (transpiled to JS for frontend, JVM for backend)

### Step 4: Submission
1. User reviews their return summary and signs electronically
2. Frontend calls POST `/taxreturn/{id}/submit`
3. Backend converts the fact graph to MeF-compliant XML
4. XML is validated against the MeF schema
5. XML and PDF artifacts are encrypted and stored in S3
6. A message is placed on the dispatch queue (SQS)
7. The Submit service reads from the queue and submits to MeF
8. MeF returns a receipt ID

### Step 5: Status Tracking
1. Submit service notifies the Status service and Backend via SQS
2. Status service polls MeF for acknowledgements
3. When accepted/rejected, status is propagated back to the backend via SQS
4. Backend updates the database and caches the status in Redis
5. Frontend polls the backend for status updates

### Step 6: State Tax Handoff (Optional)
1. After federal acceptance, user can authorize state data transfer
2. Backend generates an authorization code and stores it in State API
3. State tax tool presents the code and pulls data via State API endpoints
4. State API returns encrypted federal return data (XML + enriched JSON)

## Message Queue Architecture

DirectFile uses SQS extensively for async communication between services:

| Queue | Producer | Consumer | Purpose |
|-------|----------|----------|---------|
| `dispatch-queue` | Backend API | Submit Service | Tax return ready for MeF submission |
| `submission-confirmation-queue` | Submit Service | Backend API | Confirmation that submission was sent to MeF |
| `pending-submission-queue` | Submit Service | Status Service | Submission needs status polling |
| `status-change-queue` | Status Service | Backend API | MeF status changed (accepted/rejected) |
| Various email queues | Backend API | Email Service | Trigger email notifications |
| Dead letter queues (dlq-*) | Any | Ops/debugging | Failed messages for investigation |

## Database Architecture

DirectFile uses separate PostgreSQL databases for different services to enforce data isolation:

| Database | Service | Key Tables |
|----------|---------|------------|
| `directfile` | Backend API | `users`, `taxreturns`, `taxreturn_owners`, `taxreturn_submissions`, `submission_events` |
| `stateapi` | State API | `authorization_codes`, `state_profiles` |
| `email` | Email Service | Email templates, send history |

### Key Backend Database Tables

- **`users`**: Links external auth provider IDs to internal user IDs. Stores minimal user info.
- **`taxreturns`**: The primary table. Contains the encrypted fact graph blob, tax year, and return metadata. One row per user per tax year.
- **`taxreturn_owners`**: M:N relationship between users and tax returns (supports joint filing).
- **`taxreturn_submissions`**: One row per submission attempt. Contains a snapshot of the facts at submission time, return headers, and submission metadata. Immutable after creation.
- **`submission_events`**: Append-only audit log of status changes for each submission. Functions as a write-ahead log for replaying submission history.

### Why This Schema Design

The schema deliberately separates mutable data (the active tax return being edited) from immutable data (submission snapshots and events). This design:
1. Enables full reconstruction of a taxpayer's journey through the system
2. Supports resubmission after rejection without losing history
3. Provides an audit trail for compliance and debugging
4. Attributes time spent to specific sessions (original vs. correction)
