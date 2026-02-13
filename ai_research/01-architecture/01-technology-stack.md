# Technology Stack Deep Dive

## Why Each Technology Was Chosen

This document explains not just _what_ technologies DirectFile uses, but _why_ each was selected. Understanding these rationale is critical for making informed choices in a replacement system.

---

## Backend: Java 21 + Scala 3 (Hybrid)

### The Decision
The team chose a hybrid Java + Scala approach for the backend, where:
- **Java** handles API layers, Spring Boot services, SOAP/WSDL communication with MeF
- **Scala** implements the Fact Graph (the core tax logic engine)

### Why Scala for Tax Logic
1. **Graph-native**: Functional languages like Scala have natural support for graph data structures. Tax logic is fundamentally a dependency graph where computed values depend on other values. Scala's immutability, pattern matching, and recursion make graph traversal idiomatic.
2. **Cross-platform**: Via Scala.js, the same Scala code compiles to both JVM bytecode (for the backend) and JavaScript (for the frontend). This is critical — it means the exact same tax computation code runs on both client and server, eliminating the risk of calculation discrepancies.
3. **Domain modeling**: Scala's type system allows precise modeling of tax concepts (e.g., a fact can be complete or incomplete, writable or derived, with specific type constraints).
4. **Correctness**: The functional paradigm and lack of nulls helps prevent entire categories of bugs that would be dangerous in a tax calculation context.

### Why Java for APIs
1. **IRS familiarity**: Java is the IRS's primary backend language. All IRS infrastructure, support systems, and operational staff are built around Java.
2. **Spring Boot ecosystem**: Provides mature REST API tooling, security frameworks, database integration, and monitoring.
3. **SOAP/WSDL support**: MeF uses SOAP-based web services described by WSDL. Java has the best tooling for auto-generating client code from WSDL definitions.
4. **JVM interoperability**: Java and Scala run on the same JVM and can call each other's code directly, making the hybrid approach seamless.

### Why NOT Python, Ruby, or Node.js
The team explicitly rejected these languages for the backend, citing:
- **Package management nightmares**: Ruby, Python, and Node.js ecosystems have histories of abandoned packages with security vulnerabilities. In a government tax system handling PII/FTI, supply chain security is paramount.
- **Dependency on abandoned projects**: Nested dependency chains in npm/pip/gem can introduce versioning conflicts and security holes.
- **Government compliance**: The IRS has existing processes and tooling for Java systems. Using a different ecosystem would require building new support infrastructure.

### Implications for a Replacement System
If building a Python backend (as suggested), consider:
- The Fact Graph equivalent must be reimplemented. Python's dynamic typing requires extra care to prevent type-related bugs in tax calculations.
- There is no direct equivalent of Scala.js for Python → JavaScript transpilation. The tax logic engine will either need to be duplicated in both languages, or run exclusively on one side (server-only with API calls, or client-only with a JS implementation).
- Supply chain security must be carefully managed (pinned dependencies, security audits, etc.)

---

## Frontend: React 18 + TypeScript + Vite

### The Decision
React 18 with TypeScript, using the Vite build tool.

### Why React
1. **Government standard**: React is widely used across federal agencies. USDS (US Digital Service) and 18F both use React extensively.
2. **react-uswds**: TrussWorks maintains an official React component library for the US Web Design System (USWDS). This provides pre-built, accessible, 508-compliant UI components.
3. **Talent pool**: React developers are plentiful, which matters for a government project that relies on contractors.
4. **SPA capability**: DirectFile needs the fact graph running client-side for real-time tax computation. A SPA architecture allows edge processing (computation in the user's browser) without server round-trips.

### Why TypeScript
- Adds static typing to JavaScript, catching bugs at compile time
- Critical for a codebase where type mismatches (e.g., treating a dollar amount as a string) could cause tax calculation errors
- Enables static analysis of fact graph paths, ensuring screens reference facts that actually exist

### Why Vite (not Create React App)
- Faster development builds with hot module replacement
- Better ES module support
- More modern build tooling than the older CRA

### CSS: SASS + SASS Modules + PostCSS
- **SASS**: Required for interoperability with USWDS, which exports SASS variables
- **SASS Modules**: Scopes CSS to individual components, preventing global style conflicts
- **PostCSS**: Compiles modern CSS down to browser-compatible output via browserslist
- **Stylelint**: CSS linting with accessibility plugin for automated a11y checks

### i18n: i18next + react-i18next
- Industry-standard internationalization library
- Supports YAML translation files (chosen over JSON for human readability)
- Supports string interpolation from fact graph values (e.g., `{{/taxYear}}`)
- Supports nested translation references (`$t(path./to-value)`)
- Translation files: `en.yaml` (English), `es.yaml` (Spanish)

### State Management: Redux + Fact Graph (singleton)
- **Redux**: Used for non-tax application state (session info, UI state)
- **Fact Graph**: The primary state container for all tax data. Initialized as a global singleton on the frontend. React components read from and write to this graph directly, rather than going through Redux for tax data.

---

## Database: PostgreSQL 15

### Why PostgreSQL
1. **Reliability**: Battle-tested, open-source relational database
2. **JSON support**: Tax return data is stored as an encrypted JSON blob, and PostgreSQL has excellent JSON querying capabilities
3. **Liquibase migrations**: Schema changes are managed declaratively with rollback support
4. **Government familiarity**: Widely used in federal systems

### Key Design Choice: Blob Storage for Tax Data
The fact graph (the user's tax return data) is stored as an **encrypted JSON blob** in a single column, NOT as normalized relational data. This was a deliberate choice:
- **Flexibility**: The schema of tax data changes frequently as tax law changes. A blob avoids constant schema migrations.
- **Atomicity**: The entire fact graph is read/written as a unit, avoiding partial-update bugs
- **Encryption simplicity**: One blob = one encryption operation, using per-user envelope encryption
- **No snapshot vs. delta lock-in**: The blob approach works whether you overwrite the whole thing or apply patches

---

## Message Queues: AWS SQS

### Why SQS
- Managed service, no operational overhead for the team
- Reliable delivery with dead-letter queue support
- Integrates natively with other AWS services
- LocalStack provides a faithful local mock for development

### Queue Design
DirectFile uses multiple purpose-specific queues rather than a single shared queue:
- `dispatch-queue`: Tax returns ready for MeF submission
- `submission-confirmation-queue`: Confirms submission was sent
- `pending-submission-queue`: Returns needing status polling
- `status-change-queue`: MeF accepted/rejected notifications
- Various email notification queues
- Dead letter queues (prefixed `dlq-`) for failed messages

---

## Object Storage: AWS S3

### Why S3
- Filed tax return artifacts (PDFs, XML) are documents that should be stored as documents
- Read-only after filing (immutable), which S3 is optimized for
- Separation from the main database provides additional protection against accidental modification
- AWS S3 Encryption Client provides envelope encryption before storage

### What's Stored
- MeF XML submissions
- PDF renderings of completed tax returns
- Fact graph snapshots at submission time

---

## Encryption: AWS KMS

### Why KMS
- Manages encryption keys without the application needing to handle raw key material
- Supports per-user data encryption keys (envelope encryption)
- Auditable key usage via CloudTrail
- For local development, a `LOCAL_WRAPPING_KEY` environment variable provides a stand-in

### Encryption Approach
- **Server-side envelope encryption**: A per-user symmetric data key encrypts the fact graph. The data key is itself encrypted by a KMS root key.
- **The encrypted data key is stored alongside the encrypted fact graph** in the database
- **S3 artifacts**: Encrypted using the S3 Encryption Client with a separate KMS wrapping key

---

## Monitoring: OpenTelemetry + Prometheus + Grafana

- **OpenTelemetry**: Instrumentation standard for collecting metrics, traces, and logs
- **Prometheus**: Time-series metrics storage and querying
- **Grafana**: Dashboarding and visualization
- Enabled via Docker Compose profile (`--profile monitoring`)

---

## Build Tools

| Tool | Purpose |
|------|---------|
| Maven (mvnw) | Java/Scala backend builds, dependency management |
| SBT | Scala compilation and Scala.js transpilation |
| npm workspaces | Frontend multi-package management |
| Vite | Frontend build and dev server |
| Docker Compose | Local development orchestration |
| Liquibase | Database schema migrations |
| Spotless | Java code formatting |
| Prettier | Frontend code formatting |
| SpotBugs + PMD | Java static analysis |
| ESLint + Stylelint | Frontend linting |
| Jacoco | Java code coverage |
| Vitest | Frontend testing framework |
| pre-commit | Git hooks for formatting and linting |
