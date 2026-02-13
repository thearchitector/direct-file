# Data Import System

## Purpose

The Data Import (DI) system is responsible for pre-populating a user's tax return with information the IRS already has from internal systems. When a taxpayer starts DirectFile, the DI system can automatically retrieve their W-2 data, 1099 data, and other previously-reported information, saving them from manually re-entering it.

This is a high-value feature because:
1. Reduces data entry burden (particularly for filers with multiple W-2s or complex income)
2. Reduces errors from mistyping numbers
3. Builds trust — the user sees that the system already "knows" their information
4. Speeds up the filing process significantly

## Architecture: Spoke-Hub Model

The DI system uses a **spoke-hub architecture** within a single application (NOT a microservice mesh):

```
                    ┌─────────────────────────────┐
                    │      Backend API             │
                    │  (sends TIN + TaxYear +      │
                    │   ReturnId to input queue)    │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │        Input Queue            │
                    │  (message: TIN, TaxYear,      │
                    │   ReturnId)                   │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │        Hub (Spooler)          │
                    │  Distributes queries to       │
                    │  spokes, manages timeouts,    │
                    │  merges results               │
                    └──┬────────┬────────┬─────────┘
                       │        │        │
                ┌──────▼──┐ ┌──▼─────┐ ┌▼────────┐
                │ Spoke A  │ │ Spoke B│ │ Spoke C  │
                │ (W-2s)   │ │(1099s) │ │(SSA-1099)│
                │          │ │        │ │          │
                │ Queries  │ │Queries │ │ Queries  │
                │ IRS      │ │ IRS    │ │ SSA      │
                │ System A │ │System B│ │ System   │
                └──────┬───┘ └───┬────┘ └────┬─────┘
                       │         │           │
                       └─────────┼───────────┘
                                 ▼
                    ┌──────────────────────────────┐
                    │     Merged UserData Object    │
                    │  (all retrieved info combined) │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │      Output Queue             │
                    │  (sends merged data back to   │
                    │   Backend API)                │
                    └──────────────────────────────┘
```

### How It Works

1. **Request**: When a user initiates a return or requests data import, the Backend sends a message to the DI input queue containing the user's TIN, tax year, and return ID.

2. **Hub Distribution**: The hub (spooler) receives the message and dispatches parallel queries to each registered spoke.

3. **Spoke Processing**: Each spoke connects to a specific IRS internal system:
   - Spoke for W-2 data
   - Spoke for 1099-INT data
   - Spoke for 1099-G data
   - Spoke for SSA-1099 data
   - Spoke for 1099-R data
   - Additional spokes for other data sources

4. **ETL**: Each spoke performs ETL (Extract-Transform-Load) on the data it retrieves:
   - **Extract**: Query the IRS system for the taxpayer's records
   - **Transform**: Convert from the source system's format to DirectFile's UserData schema
   - **Load**: Categorize the data into the appropriate structure

5. **Timeout Management**: The hub enforces a configurable timeout (default 5 seconds). If a spoke doesn't respond in time, the hub proceeds with whatever data has been collected. Late-arriving data can be sent as an update.

6. **Merge**: The hub merges UserData objects from all responding spokes into a single unified result.

7. **Response**: The merged result is sent to the Backend via the output queue. The Backend populates the user's fact graph with the imported data.

### Why a Single App (Not Microservices)

The team explicitly rejected a microservice architecture for DI because:
- **Scale**: Fewer than 20 data source integrations don't justify microservice overhead
- **Failure points**: Each microservice adds network hops, message queues, and failure modes
- **Complexity**: A single app with modular code is easier to test, deploy, and debug
- **Efficiency**: Shared connection pools, shared configuration, single deployment unit

The ADR acknowledges that if the system scales to dozens of data sources, a microservice approach might become necessary, but for the foreseeable scale, a monolith is correct.

### Why NOT Have the Backend Do DI Directly

The team also rejected having the Backend API handle data import inline:
- **Complexity isolation**: The Backend is already complex enough with auth, persistence, submission, and PDF/XML generation
- **Latency**: DI queries to external systems are slow (seconds). Processing them synchronously in the API would block user requests.
- **Scaling independence**: The DI system has different scaling characteristics than the API (bursty at the start of filing season)

## Mock Data Import Service

For local development, a mock DI service is used instead of connecting to real IRS systems. The mock service:
- Is activated by the `mock` Spring Profile
- Returns predefined data profiles from JSON files in `backend/src/main/resources/dataimportservice/mocks/`
- Includes profiles like `marge.json` and `homer.json` with sample tax data
- Developers can add custom profiles by adding JSON files to the mocks directory
- The profile to use is selected via the `x-data-import-profile` request header

## Data Import Gating

The DI system includes a "gating" mechanism that controls which data sources are available for which users. This is configured in `data-import-gating-sample.json` and allows:
- Gradual rollout of new data source integrations
- Per-user or per-group enabling/disabling of specific data sources
- Feature-flagging of data import capabilities

## Imported Data in the Fact Graph

Imported data is stored in the fact graph alongside user-entered data, but with metadata that distinguishes imported facts from manually entered facts:
- The `imported.xml` module (25 KB) defines facts related to data import
- Imported data is presented to the user for verification — it is NEVER auto-submitted
- If the user modifies an imported value, the modified value takes precedence
- The flow tracks which items were imported vs. manually entered (used for analytics and user experience)

## Implications for a Replacement System

1. **The spoke-hub pattern is clean and extensible**: Even if the new system doesn't connect to IRS internal systems, this pattern works for any external data source (employer portals, financial institution APIs, etc.)

2. **Always let users verify imported data**: Pre-populated data should be editable and require explicit confirmation. Never auto-submit based on imported values alone.

3. **Design for partial results**: Not all data sources will respond in time. The system must work gracefully with incomplete imported data.

4. **Use mock services for development**: Real data sources are often unavailable or restricted during development. A mock service with predefined profiles is essential.

5. **Separate DI from the main API**: Data import has different latency, reliability, and scaling characteristics. Keep it as a separate concern.
