# Recommendations for Building a Replacement Tax Filing Application

## Overview

This document synthesizes all findings from the DirectFile analysis into actionable recommendations for designing and building a new, open-source web-based tax assistance application. Recommendations are organized by priority and topic area.

---

## Priority 1: Core Architecture Decisions

### 1.1 Build a Declarative Tax Rules Engine (Fact Graph Equivalent)

**This is the single most critical component of the new system.**

DirectFile's Fact Graph is not just a data structure — it simultaneously serves as:
- The data model (stores all user-entered and computed data)
- The computation engine (derives tax calculations)
- The flow controller (determines which screens to show via boolean derived facts)
- The validation engine (tracks completeness of every field)
- The form generator input (populates PDF and XML outputs)

**Recommendation**: Build a Python-based declarative rules engine that:
- Represents tax rules as a dependency graph of facts
- Supports writable facts (user input) and derived facts (computed values)
- Tracks completeness (every fact is either complete or incomplete)
- Supports collections (multiple W-2s, multiple dependents)
- Organizes facts into modules with explicit export boundaries
- Can serialize/deserialize to JSON for storage and transport

**Implementation approach**:
- Define the rules in a declarative format (YAML, JSON, or a Python DSL — NOT embedded in imperative code)
- The engine should be a pure computation layer with no side effects
- All tax-specific logic should be in the rule definitions, not in the engine code
- The engine should be thoroughly tested independently of the UI

### 1.2 Solve the Isomorphic Execution Problem

DirectFile runs the same Scala tax engine on both client (JavaScript) and server (JVM). This guarantees identical calculations. A Python backend cannot directly share code with a JavaScript frontend.

**Recommended approach**: Write the tax rules engine in TypeScript and run it isomorphically:
- The TypeScript engine runs in the browser for instant feedback
- The Python backend calls the engine via a Node.js subprocess, embedded JS runtime (e.g., PyMiniRacer), or a lightweight Node.js sidecar service
- This gives you: client-side speed + server-side authority + single source of truth for tax logic

**Alternative approach** (simpler but with trade-offs): Server-only computation:
- All tax math happens on the Python server
- The client sends data via API calls and receives computed results
- Trade-off: every input requires a network round-trip, degrading the mobile experience
- Mitigation: optimistic UI updates, debounced API calls, websocket-based streaming

### 1.3 Design the Interview Flow as Declarative Configuration

The interview flow (which screens to show, in what order, under what conditions) should be defined as data, not code.

**Recommendation**: Store the flow definition in YAML or JSON with this hierarchy:
```
flow:
  categories:
    - route: "you-and-your-family"
      subcategories:
        - route: "about-you"
          complete_when: "/aboutYouComplete"
          screens:
            - route: "intro"
              condition: null  # always shown
              fields:
                - path: "/filers/*/firstName"
                  type: "string"
                  required: true
                - path: "/filers/*/lastName"
                  type: "string"
                  required: true
            - route: "ssn"
              condition: {operator: "isComplete", fact: "/filers/*/firstName"}
              fields:
                - path: "/filers/*/ssn"
                  type: "tin"
                  required: true
```

**Key design rules** (carried forward from DirectFile):
1. All flow conditions must be simple boolean checks against the rules engine — push complex logic into the engine as derived boolean facts
2. The flow configuration is static at startup — all possible paths exist in the configuration
3. Multiple components (screen renderer, checklist, data view, navigation) independently read the same flow config

### 1.4 Use a Single JSON Document for Active Tax Data

**Recommendation**: Store the user's in-progress tax return as an encrypted JSON blob in PostgreSQL, not as normalized relational tables.

**Reasons** (validated by DirectFile's experience):
- Tax schemas change annually — blob avoids constant migrations
- Atomic reads/writes prevent partial-update bugs
- Single read per page load (no JOINs)
- Encryption is simpler for a single blob
- Schema evolution happens in application code, not database migrations

---

## Priority 2: Tax Domain Implementation

### 2.1 Start with the Supported Tax Scope

Based on DirectFile's scope, the initial supported tax situations should be:

**Income types** (in order of prevalence):
1. W-2 wage income (covers ~70% of filers)
2. 1099-INT interest income
3. 1099-G unemployment compensation
4. SSA-1099 Social Security benefits
5. 1099-R retirement distributions
6. Alaska Permanent Fund dividends

**Filing statuses**: All five (Single, MFJ, MFS, HoH, QSS)

**Credits** (in order of complexity):
1. Child Tax Credit / Other Dependent Credit
2. Earned Income Tax Credit (most complex, most impactful)
3. Standard Deduction (with age/blindness additional amounts)
4. Educator Expense Adjustment
5. Student Loan Interest Deduction

**Explicit knockouts**: Itemized deductions, self-employment, capital gains, rental income, foreign income, multiple state residency, non-resident aliens

### 2.2 Implement Knockouts Early

The knockout system (detecting unsupported situations and directing users elsewhere) should be one of the first things built. Reasons:
- Prevents users from wasting time on a return that can't be completed
- Simplifies the initial scope by clearly defining boundaries
- Can be implemented as boolean derived facts in the rules engine

### 2.3 Model Tax Forms as Output, Not Input

DirectFile does NOT present blank tax forms for users to fill in. Instead, it asks plain-language questions and generates the forms from the answers. This is a fundamental design philosophy:

- Users should never need to know what "Schedule 1, Line 8g" means
- Questions should be answerable from personal knowledge without consulting IRS publications
- The system translates plain-language answers into form data behind the scenes

### 2.4 Handle Dependent Qualification Rigorously

The dependent/qualifying person determination is one of the most complex areas of tax law and was a significant source of bugs in DirectFile. The rules involve:
- Relationship tests (child, sibling, parent, etc.)
- Age tests (varies by credit)
- Residency tests (lived with taxpayer for how long)
- Support tests (who paid for more than half of support)
- Joint return test
- Citizenship/residency test

Each credit has its own definition of "qualifying child" or "qualifying person." A single dependent might qualify for CTC but not EITC, or vice versa.

---

## Priority 3: Frontend Architecture

### 3.1 Use React with a Government Design System

**Recommendation**: React + TypeScript + US Web Design System (USWDS)

- React is the standard for government SPAs
- TypeScript catches type errors at compile time (critical for tax data)
- USWDS provides accessible, 508-compliant components out of the box
- @trussworks/react-uswds provides React bindings

### 3.2 Design for Mobile-First

DirectFile was explicitly designed to work on mobile phones. Key considerations:
- Touch-friendly input fields (especially for dollar amounts, dates, SSNs)
- Minimal data entry per screen (one concept per page)
- Progress tracking via checklist
- Resume-anywhere capability (users may complete their return over multiple sessions)

### 3.3 Build for Bilingual Support from Day One

Do NOT add internationalization later — build it in from the start:
- Use i18next or equivalent for all user-facing text
- Store translations in YAML files (more readable than JSON for content editors)
- Structure HTML content as YAML objects (avoids embedding raw HTML in translation strings)
- Support fact value interpolation in translation strings
- Plan the translation workflow: export to spreadsheet → translate → import back

### 3.4 Target WCAG 2.2 AA Accessibility

This should be a non-negotiable requirement:
- Use semantic HTML and proper heading hierarchy
- All functionality must be keyboard-accessible
- Test with screen readers (NVDA on Windows, VoiceOver on macOS)
- Automated accessibility checks in CI/CD
- Use the USWDS design system for accessible base components

---

## Priority 4: Backend Architecture

### 4.1 Python Backend with Django or FastAPI

**Recommendation**: Use Python with either Django (for full-featured ORM, admin, and ecosystem) or FastAPI (for modern async API design).

Key backend responsibilities:
- User session management
- Encrypted storage/retrieval of tax data
- Tax rules engine execution (authoritative server-side computation)
- PDF generation from tax data
- XML generation for e-filing (MeF or equivalent)
- Email notifications
- State data transfer API

### 4.2 PostgreSQL with Alembic Migrations

- PostgreSQL for the primary database (proven, reliable, good JSON support)
- Alembic (if using SQLAlchemy) or Django migrations for schema management
- Separate databases per service (if using microservices)

### 4.3 Message Queue for Async Processing

Use a message queue (Redis Streams, RabbitMQ, or AWS SQS) for:
- Submission processing (don't block the user waiting for e-filing)
- Email notifications
- Status polling
- Data import from external sources

### 4.4 Submission Pipeline

Design the submission pipeline as a separate concern:
1. Backend validates completeness and generates XML
2. XML is stored immutably (S3 or equivalent)
3. Submission message goes to queue
4. Submission worker sends to MeF (or future equivalent)
5. Status worker polls for acceptance/rejection
6. Status updates flow back to the user via the backend

### 4.5 Resubmission-Aware Schema

Use DirectFile's proven three-table pattern:
- **TaxReturn**: Mutable working copy (encrypted JSON blob)
- **Submission**: Immutable snapshot at submission time
- **SubmissionEvent**: Append-only audit log of status changes

---

## Priority 5: Security and Compliance

### 5.1 Encrypt All Taxpayer Data at Rest

Use envelope encryption:
- Per-user encryption key for the tax data blob
- Wrap key managed by a key management service
- Separate key for S3/file storage artifacts
- For local development, use a local key (not a cloud KMS)

### 5.2 Delegate Authentication to an External Provider

Do NOT build authentication into the application:
- Use an external identity provider (Auth0, Keycloak, Login.gov, etc.)
- Link users via external ID, not PII
- This avoids the burden of identity proofing and credential management
- Build a simulator for local development

### 5.3 Minimize PII Exposure

- Never log plaintext tax data
- Never include tax data in URLs
- Implement Content Security Policy headers
- Design for shared computer safety (public library usage)
- Automatic session timeout after inactivity

---

## Priority 6: Testing Strategy

### 6.1 Four Types of Tests (From DirectFile)

1. **Completeness tests**: For each section of the interview, verify that all required facts are set regardless of which path the user takes
2. **Correctness tests**: Unit tests that verify tax calculations produce correct results for known inputs
3. **Flow navigation tests**: Verify the correct next screen is shown for every combination of current screen + user answer
4. **Snapshot tests**: For each predefined scenario, verify the full screen sequence hasn't changed unexpectedly

### 6.2 Build a Scenario Library

Create hundreds of predefined "scenarios" — complete tax situations as JSON files:
- Each scenario represents a specific taxpayer profile
- Scenarios serve as test inputs for all four test types
- Include edge cases and boundary conditions
- Add a migration script for batch-updating scenarios when the schema changes

### 6.3 Involve Tax Experts

Automated tests verify that code behaves as coded. Only human tax experts can verify that the coding matches the law. Plan for:
- Manual testing by tax subject matter experts
- Review of test scenarios for correctness
- Bug reports that trigger new test cases

---

## Priority 7: Data Import (Optional but High-Value)

### 7.1 Pre-Populate from Available Data Sources

If access to external data sources is available (employer W-2 databases, financial institutions, prior-year returns), implement a data import system:
- Spoke-hub architecture: central hub dispatches to parallel data source modules
- Configurable timeout (5 seconds per source)
- Merge results from multiple sources into unified format
- Present imported data for user verification (never auto-submit imported data)

### 7.2 Design for Graceful Degradation

Data import should be optional and non-blocking:
- If a data source is unavailable, the user can still manually enter data
- Partial results are better than no results
- Imported data must always be user-verified before use

---

## Summary: Minimum Viable Architecture

For an initial working system, the absolute minimum components are:

```
┌─────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  React Frontend │◄───▶│  Python Backend   │◄───▶│   PostgreSQL     │
│  + TypeScript   │     │  (Django/FastAPI) │     │   (encrypted     │
│  Tax Engine     │     │  + Tax Engine     │     │    JSON blobs)   │
│  + USWDS        │     │  + PDF Generator  │     │                  │
│  + i18next      │     │  + XML Generator  │     │                  │
└─────────────────┘     └──────────────────┘     └──────────────────┘
```

1. **Tax Rules Engine** (TypeScript, shared client/server): The core computation
2. **Flow Configuration** (YAML/JSON): Defines the interview structure
3. **React Frontend**: Renders screens, collects input, shows computed values
4. **Python Backend**: API, authentication, storage, submission
5. **PostgreSQL**: Encrypted tax data storage
6. **Translation Files** (YAML): English + Spanish content

Everything else (message queues, S3, state API, email service, monitoring) can be added incrementally as the system matures.

---

## Anti-Patterns to Avoid

1. **Don't normalize tax data into relational tables** — Use a document/blob approach
2. **Don't embed tax logic in imperative code** — Keep it declarative and separate from the engine
3. **Don't build authentication** — Delegate to an external provider
4. **Don't present blank tax forms** — Ask plain-language questions
5. **Don't skip accessibility** — Build it in from day one, don't bolt it on later
6. **Don't skip internationalization** — Same reasoning
7. **Don't create a monolithic fact dictionary** — Use modules with export boundaries from the start
8. **Don't duplicate tax logic across client and server** — Find a way to share it
9. **Don't block the user during submission** — Use async processing with status polling
10. **Don't store submission snapshots in the same mutable record** — Separate mutable working data from immutable submission records
