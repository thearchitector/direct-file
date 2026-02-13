# State Tax Integration (State API)

## Purpose

Although DirectFile only files **federal** tax returns, US taxpayers also have state and local filing obligations. Rather than forcing users to re-enter all their federal information in a separate state filing tool, DirectFile enables an optional data transfer. After federal filing, the user can authorize a state tax tool to pull their federal return data through the State API.

This is a critical feature because:
1. It dramatically reduces the burden on taxpayers who need to file state returns
2. It prevents data entry errors from re-typing federal data
3. It enables a smooth handoff to partner state filing tools
4. It transfers not just raw XML but also enriched JSON with additional data elements useful for state calculations

## Architecture

The State API is a separate Spring Boot service with its own PostgreSQL database. It does NOT share a database with the main Backend API.

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  DirectFile  │     │   State API  │     │  State Tax   │
│  Backend     │────▶│   Service    │◀────│  Tool (3rd   │
│              │     │              │     │  party)      │
└──────────────┘     └──────────────┘     └──────────────┘
       │                    │
       ▼                    ▼
 ┌──────────┐        ┌──────────┐
 │ directfile│        │ stateapi │
 │    DB     │        │    DB    │
 └──────────┘        └──────────┘
```

## Data Transfer Flow

### Step 1: Federal Return Accepted
After the user's federal return is accepted by MeF, the State API handoff becomes available.

### Step 2: User Authorizes Transfer
The user is shown an "Authorize State" screen where they can opt to share their federal return data with a specific state filing tool. The authorization process:
1. User selects their state
2. User confirms they want to share their data
3. The Backend generates an **authorization code** (UUID)
4. The authorization code is stored in the State API database along with:
   - Tax return UUID
   - TIN (taxpayer identification number)
   - Tax year
   - State code
   - MeF submission ID

### Step 3: State Tool Requests Data
The state filing tool (operated by a state revenue agency or authorized vendor) calls the State API to retrieve the data:

#### V1 Endpoint: `POST /state-api/authorization-code`
Saves the authorization code to the database.

#### V2 Endpoint: `POST /state-api/v2/authorization-token`
Generates a JWT token containing encrypted metadata for time-limited access.

#### Export Endpoint: `GET /state-api/export-return`
Retrieves the actual federal return data. The request includes:
- A JWT Bearer token signed with the state's private key
- The JWT contains: issuer (state account ID), subject (authorization code), issued-at timestamp

### Step 4: Authentication and Verification
The State API verifies the request:
1. **Bearer token present**: The JWT must be in the Authorization header
2. **JWT verification**: Signature verified using the state's registered public certificate
3. **Certificate validation**: The state's certificate must be registered and not expired
4. **Authorization code validation**: The code must exist and not be expired
5. **Account ID validation**: The issuer must match a registered state account

### Step 5: Data Returned
If all checks pass, the State API returns:

```json
{
  "status": "success",
  "taxReturn": "<encrypted-data>"
}
```

The encrypted `taxReturn` field contains:
```json
{
  "status": "accepted",
  "submissionId": "123456",
  "xml": "<MeF XML return data>",
  "directFileData": "<enriched JSON with additional data elements>"
}
```

The `directFileData` JSON includes data elements beyond what's in the standard MeF XML that were identified as useful to state revenue agencies for streamlining the state filing experience.

## Error Codes

The State API returns structured error responses:

| Error Code | Meaning |
|-----------|---------|
| `E_BEARER_TOKEN_MISSING` | No JWT bearer token in request |
| `E_AUTHORIZATION_CODE_NOT_EXIST` | Authorization code not found in database |
| `E_AUTHORIZATION_CODE_EXPIRED` | Authorization code has expired |
| `E_AUTHORIZATION_CODE_INVALID_FORMAT` | Malformed authorization code |
| `E_ACCOUNT_ID_NOT_EXIST` | State account ID not registered |
| `E_JWT_VERIFICATION_FAILED` | JWT signature verification failed |
| `E_CERTIFICATE_NOT_FOUND` | State's public certificate not registered |
| `E_CERTIFICATE_EXPIRED` | State's certificate has expired |
| `E_INTERNAL_SERVER_ERROR` | Unexpected server error |

## Security Model

### State Authentication
Each state is registered with:
- An **account ID** (identifies the state)
- A **public certificate** (used to verify JWT signatures)
- A **state profile** in the `state_profiles` database table

States sign their JWT tokens with a private key corresponding to their registered certificate. The State API verifies the signature using the public certificate.

### Encryption
- Data returned by the export endpoint is encrypted
- The State API has its own LocalStack/S3 integration for accessing stored artifacts
- The State API uses the same envelope encryption pattern as the main backend for data at rest

### Time-Limited Access
Authorization codes and JWT tokens have expiration windows. This limits the window during which a compromised code could be used to access data.

## Database Schema

The State API has its own PostgreSQL database (`stateapi`) with key tables:

- **`state_profiles`**: Registered state accounts, certificates, and configuration
- **`authorization_codes`**: Active authorization codes linking tax returns to state data requests

## Why a Separate Service?

The State API is a separate service (not part of the Backend API) because:

1. **Different trust boundaries**: State tax tools are third-party systems. The State API acts as a controlled gateway with its own authentication model.
2. **Different scaling requirements**: State data requests are bursty (concentrated after federal acceptance) and independent of the main filing flow.
3. **Security isolation**: A vulnerability in the State API should not directly compromise the main filing system.
4. **Independent deployment**: State API changes (new states, certificate rotations) shouldn't require redeploying the main backend.

## Supported States

DirectFile was designed to work with multiple state tax systems. Each state has a profile in the `state_profiles` table. The system was designed for expansion — adding a new state primarily involves:
1. Registering the state's account and certificate
2. Configuring the state's profile
3. Ensuring the enriched JSON includes data elements the state needs

## Design Lessons for a Replacement System

1. **Separate the state handoff from the main filing flow**: It has different authentication, different trust requirements, and different scaling patterns.
2. **Provide enriched data beyond raw XML**: States identified additional data elements they needed beyond what MeF XML contains. Including these in a structured JSON format alongside the XML significantly improves the state filing experience.
3. **Use certificate-based authentication**: JWT + certificate pairs provide a robust authentication model for machine-to-machine API access.
4. **Time-limit all authorization tokens**: Minimize the window for data exposure.
5. **Plan for the enriched data format from the start**: The `directFileData` JSON was added based on state feedback. A new system should anticipate what additional data states might need and design the export format accordingly.
