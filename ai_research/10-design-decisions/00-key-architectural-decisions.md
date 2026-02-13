# Key Architectural Decisions and Their Rationale

## Overview

This document consolidates the most important design decisions made during DirectFile's development, explaining not just what was decided but WHY. These decisions were documented in Architecture Decision Records (ADRs) and RFCs found in the `docs/adr/` and `docs/rfc/` directories.

Understanding the rationale behind each decision is critical for a replacement system — some decisions should be carried forward, others might be revisited given different constraints.

---

## Decision 1: Fact Graph as Central Knowledge Representation

**Date**: Early 2022 (foundational)

**Decision**: All tax logic is expressed as a declarative graph of interconnected facts, using a Scala-based engine that compiles to both JVM and JavaScript.

**Why**:
- Tax law IS a dependency graph. Every computed value depends on other values in complex, sometimes circular ways.
- A graph representation makes relationships explicit and traversable, unlike imperative code where dependencies are implicit in execution order.
- The graph naturally handles incomplete information (partially-filled returns) without crashing.
- Declarative rules can potentially be reviewed by tax experts who aren't programmers.
- Academic backing: https://arxiv.org/pdf/2009.06103

**Trade-offs accepted**:
- Higher learning curve for developers unfamiliar with graph-based computation
- Scala adds language complexity beyond what Java alone would require
- The fact dictionary XML format is verbose and can be hard to read for complex derivations

**Verdict for replacement**: This is the single most important decision to carry forward. Any replacement MUST have a central, declarative representation of tax rules that can serve as data model, computation engine, and flow controller simultaneously.

---

## Decision 2: Hybrid Java + Scala Backend

**Date**: December 14, 2022

**Decision**: Use Scala for the Fact Graph engine and Java/Spring Boot for API layers, running together on the JVM.

**Why Scala**:
- Functional programming naturally supports graph concepts (immutability, recursion, pattern matching)
- Scala.js enables transpilation to JavaScript for client-side execution
- Strong type system helps prevent bugs in tax calculations
- Used in financial domain at scale (Twitter, banks)

**Why Java for APIs**:
- IRS's primary backend language — all infrastructure and ops tooling supports Java
- Spring Boot provides mature REST API, security, and SOAP/WSDL tooling
- SOAP/WSDL support is critical for MeF integration
- Seamless JVM interoperability with Scala

**Why NOT Python/Ruby/Node**:
- Supply chain security concerns (abandoned packages, nested dependencies with vulnerabilities)
- Government compliance processes are built around Java
- Package management "nightmares" in production

**Verdict for replacement**: The Java/Scala-specific choice doesn't need to be carried forward, but the REASONS for the choice do:
1. The tax engine must run isomorphically (client + server)
2. MeF requires SOAP/WSDL support (though a wrapper service could handle this)
3. Supply chain security must be rigorously managed regardless of language

---

## Decision 3: React SPA with Client-Side Fact Graph

**Date**: January 3, 2023 (updated March 8, 2023)

**Decision**: Build a React 18 SPA with TypeScript, where the transpiled Scala fact graph runs client-side as the primary state and computation engine.

**Technology choices**:
- React 18 with Hooks (standard government SPA framework)
- TypeScript (catches bugs at compile time)
- i18next (internationalization)
- Redux (non-tax application state)
- react-uswds (accessible USWDS components from TrussWorks)
- Vitest (testing, replaced Jest)

**Why SPA over MPA**:
- The fact graph running client-side enables instant tax computation feedback without server round-trips
- Critical for mobile experience where network latency would make server-side-only computation feel sluggish
- Edge processing: the user's device does the computation, reducing server load

**Why NOT SSR/SSG for the interview**:
- The interview is inherently dynamic and interactive, not suitable for static generation
- However, the screener (eligibility check) IS static and uses Astro SSG

**Verdict for replacement**: React is a strong choice for the frontend. The key architectural property to preserve is client-side tax computation for instant feedback.

---

## Decision 4: Flow Configuration as Serializable Code

**Date**: May 10, 2023

**Decision**: The interview flow is defined as JSX/TSX components that are serializable configuration, not runtime rendering logic.

**Key constraints**:
1. The fact graph is THE source of state — not React state, not Redux
2. Screens are driven by static configuration mapping facts to fields
3. Conditionals in the flow must be simple boolean checks against the fact graph
4. Complex logic belongs in the fact graph as derived boolean facts
5. The flow configuration is a global constant — all possible paths exist at startup

**Why**:
- Separation of concerns: tax logic in the fact graph, presentation logic in the flow
- The flow could eventually become configuration editable by tax experts
- Static analysis can verify flow correctness at build time
- Multiple components (BaseScreen, Checklist, DataView, getNextScreen) all read the same flow config independently

**Verdict for replacement**: The separation between tax logic (fact graph equivalent) and presentation logic (flow configuration) is essential. Whether the flow is expressed as JSX, JSON, YAML, or a database is an implementation detail — the important thing is that it's declarative and statically analyzable.

---

## Decision 5: Fact Dictionary Modules with Export Boundaries

**Date**: November 27, 2023

**Decision**: Split the monolithic fact dictionary into modules (separate XML files) with private-by-default visibility. Facts must be explicitly exported to be used across modules.

**Why**:
- The monolithic dictionary had ~800 facts with any-to-any dependencies — impossible to reason about or test
- Bugs in one section (e.g., spouse) would manifest in unrelated areas (e.g., deductions, credits)
- Developers couldn't know which facts were "safe" to depend on from other sections
- Testing "everything, everywhere, all of the time" was intractable

**The solution**:
- ~37 modules, each in its own XML file
- ~140 "culminating facts" exported across module boundaries
- Build-time enforcement: referencing a non-exported fact from another module fails the build
- Test strategy: test each module independently, then test cross-module boundaries

**Verdict for replacement**: Module boundaries are essential for maintainability. Any replacement must have a way to organize tax rules into independent, testable modules with well-defined interfaces.

---

## Decision 6: Server-Side Envelope Encryption for Taxpayer Data

**Date**: July 31, 2023 (fact graphs), August 21, 2023 (S3 artifacts)

**Decision**: Use AWS KMS-based envelope encryption for all taxpayer data at rest.

**Fact graph encryption**: Per-user symmetric data key, encrypted by KMS root key. Both stored in the database.

**S3 artifact encryption**: Amazon S3 Encryption Client with a separate KMS wrapping key.

**Why server-side (not client-side)**:
- Simpler implementation with industry-standard tools
- The server needs access to plaintext data for XML/PDF generation
- Meets security requirements without adding complexity that could delay launch
- Client-side encryption can be added later as an additional layer

**Verdict for replacement**: Envelope encryption is the right pattern. The specific provider (AWS KMS) can change, but the per-user encryption key approach should be preserved.

---

## Decision 7: JSON Blob Storage for Active Tax Data

**Date**: December 21, 2022

**Decision**: Store the active tax return (fact graph) as an encrypted JSON blob in PostgreSQL, NOT as normalized relational tables.

**Why**:
- Flexibility: works with both snapshot and delta update approaches
- Atomicity: single-row transactions prevent partial updates
- Efficiency: one read, not dozens of JOINs
- Schema evolution: tax law changes annually, blob avoids constant schema migrations
- Single failure point: no additional systems beyond the existing database

**Verdict for replacement**: This is a pragmatic and correct decision. A replacement system should also store the active tax data as a blob (JSON document) rather than trying to normalize tax data into relational tables.

---

## Decision 8: YAML for Translation Files

**Date**: September 6, 2023

**Decision**: Use YAML (not JSON, not Markdown) for i18n translation files, with structured HTML representation.

**Why**:
- YAML is more human-readable than JSON for content editors and translators
- Structured YAML representation of HTML avoids embedding raw HTML tags in translation strings
- YAML is a superset of JSON, so conversion is straightforward
- Translation strings can be programmatically split into meaningful snippets for translators

**Trade-offs**:
- YAML indentation sensitivity can cause errors
- react-i18next doesn't support YAML natively (requires a conversion step)
- Requires converting existing JSON translation files

**Verdict for replacement**: YAML is a good choice for translation files. The structured HTML representation (using YAML arrays and objects instead of raw HTML strings) significantly reduces translator errors.

---

## Decision 9: Separate Screener as Static Site (Astro SSG)

**Date**: August 8, 2023

**Decision**: Build the eligibility screener as a separate static site using Astro, not as part of the React SPA.

**Why**:
- The screener is mostly static content with simple yes/no questions
- Screener pages are unprotected (not behind authentication)
- Keeping unprotected routes separate from protected routes simplifies security
- Static pages are faster to load and easier to deploy
- Content changes can be deployed independently from the main application

**Why Astro specifically**:
- Focuses on shipping minimal JavaScript
- Supports React components if needed
- Good i18next integration
- Content-focused with multi-page architecture

**Verdict for replacement**: Separating the screener from the main application is good practice. The specific SSG framework matters less than the separation of concerns.

---

## Decision 10: WCAG 2.2 AA Accessibility Target

**Date**: October 2023

**Decision**: Target WCAG 2.2 AA compliance, exceeding the baseline Section 508 (WCAG 2.0 A+AA) requirement.

**Why**:
- Tax filing is a critical task for ALL US residents — accessibility is essential
- WCAG 2.2 is the latest guidance, setting the application up for years to come
- WCAG 2.2 AA doesn't require more work than 2.1 AA with the current design
- Exceeding 508 aligns with the mission of serving taxpayers with diverse abilities

**Implementation**:
- react-uswds provides accessible base components
- Automated a11y testing via stylelint plugins
- Manual testing with NVDA (Windows) and VoiceOver (macOS)
- APCA (WCAG 3.0 draft) contrast checks for forward-thinking color choices

**Verdict for replacement**: WCAG 2.2 AA is the right target. Using a design system with pre-built accessible components (like USWDS) dramatically reduces the accessibility burden.

---

## Decision 11: Resubmission-Aware Database Schema

**Date**: December 2023

**Decision**: Restructure the database to cleanly separate mutable data (active return) from immutable data (submission snapshots and events).

**Schema**:
- `TaxReturn`: Mutable working copy of the return
- `Submission`: Immutable snapshot of facts at submission time (1:M with TaxReturn)
- `SubmissionEvent`: Append-only audit log of status changes (1:M with Submission)

**Why**:
- Original schema couldn't reliably reconstruct a taxpayer's journey through resubmissions
- Need to attribute time to specific sessions (20 min original + 5 min correction, not just "25 min")
- Compliance requirement: must be able to replay submission history from database alone
- Making schema changes after production launch is orders of magnitude more expensive

**Verdict for replacement**: This three-table pattern (mutable working data → immutable submission snapshots → append-only event log) is a robust design that should be carried forward.

---

## Decision 12: Data Import System (Spoke-Hub Architecture)

**Date**: July 3, 2024

**Decision**: Build a single Data Import application with a spoke-hub architecture for fetching taxpayer data from IRS internal systems.

**How it works**:
- The "hub" (spooler) receives a request with TIN and tax year
- It dispatches to multiple "spokes" (Data Import Modules), each connecting to a different IRS system
- Spokes query in parallel with a configurable timeout (5 seconds)
- Results are merged into a unified UserData structure
- If the timeout fires before all systems respond, partial data is sent; updates follow

**Why single app (not microservices)**:
- At the current scale (<20 integrations), microservices add failure points without proportional benefit
- Centralized management makes error cases easier to track and test
- Fewer messages = fewer failure points
- Simpler deployment

**Verdict for replacement**: The spoke-hub pattern is a clean way to aggregate data from multiple sources. For a new system that doesn't integrate with IRS internal systems, this specific architecture may not be needed, but the pattern is useful if integrating with any external data sources (e.g., employer W-2 databases, financial institutions).

---

## Summary: Decisions to Carry Forward vs. Revisit

### Carry Forward
1. Central declarative tax logic engine (Fact Graph equivalent)
2. Isomorphic execution (same logic on client and server)
3. Module boundaries with export rules
4. Blob storage for active tax data
5. Envelope encryption with per-user keys
6. Flow configuration as declarative, serializable data
7. Resubmission-aware schema (mutable + immutable + event log)
8. WCAG 2.2 AA accessibility target
9. Separate screener from main application
10. YAML-based structured translation files

### Revisit for New Constraints
1. Java/Scala → Python backend (need new isomorphic execution strategy)
2. MeF SOAP/WSDL → could wrap in a dedicated submission microservice
3. AWS-specific services → could use provider-agnostic alternatives
4. Redux for non-tax state → modern alternatives (Zustand, Jotai) are simpler
5. Astro for screener → could be part of the main app with route-level auth
