# DirectFile Research & Analysis

This directory contains comprehensive research documentation analyzing the IRS DirectFile open-source application. The purpose of this documentation is to inform the design and development of a new, open-source web-based tax assistance application.

## How to Navigate This Documentation

Documents are organized by topic area, numbered for suggested reading order. Start with the executive summary, then read topics relevant to your current task.

### Top-Level Documents

| Document                               | Description                                                                                                                                                                  |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `00-executive-summary.md`              | **Start here.** High-level overview of what DirectFile is, what it does, and how it works. Technology stack summary, supported tax scope, and the key architectural insight. |
| `11-recommendations-for-new-system.md` | **Read second.** Synthesized recommendations for building a replacement system: architecture decisions, implementation priorities, anti-patterns to avoid.                   |

### Detailed Topic Areas

#### `01-architecture/` — System Architecture

| Document                   | Description                                                                                                                                         |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `00-system-overview.md`    | Service architecture diagram, all services described, data flow from user input to filed return, message queue architecture, database schema design |
| `01-technology-stack.md`   | Deep dive into every technology choice and WHY it was made: Java+Scala backend, React frontend, PostgreSQL, SQS, S3, KMS, monitoring                |
| `02-data-import-system.md` | The spoke-hub architecture for pre-populating returns from IRS internal systems, mock service for development                                       |

#### `02-fact-graph-engine/` — The Core Tax Logic Engine

| Document                                    | Description                                                                                                                                                       |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `00-concept-and-purpose.md`                 | What the Fact Graph is, why a graph, writable vs derived facts, completeness, modules, collections, how it's used across the entire system                        |
| `01-fact-dictionary-xml-format.md`          | The XML format for defining facts: types, computation nodes (arithmetic, boolean, switch/case, collections), exports, optional facts pattern, real-world examples |
| `02-scala-transpilation-and-isomorphism.md` | How Scala.js enables the same code on client and server, the build/transpile process, frontend lifecycle, implications for a Python replacement                   |

#### `03-interview-flow-and-frontend/` — Interview UI and Flow

| Document                                  | Description                                                                                                                                                |
| ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `00-flow-architecture.md`                 | Category→Subcategory→Screen hierarchy, flow configuration format, conditions, screen content declarations, navigation algorithm, knockouts, batches        |
| `01-frontend-components-and-rendering.md` | React component architecture, BaseScreen rendering pipeline, fact input components, state management, URL structure, i18n, accessibility, all-screens tool |
| `02-screener-and-eligibility.md`          | Pre-login eligibility screener (Astro SSG), in-app knockout system, knockout facts and placement                                                           |
| `03-error-handling-and-alerts.md`         | Alert types (DFAlert, TaxReturnAlert, MefAlert, Assertion), error message patterns, alert aggregation, MeF rejection handling flow                         |

#### `04-tax-domain-knowledge/` — Tax Domain Concepts

| Document                                  | Description                                                                                                                                      |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `00-tax-forms-and-concepts.md`            | Form 1040 structure, filing statuses, all supported income types, credits, deductions, knockouts — explained for readers with zero tax knowledge |
| `01-fact-dictionary-modules-reference.md` | Catalog of all 37 XML modules: what each covers, file sizes, dependency map, key statistics, annual update process                               |

#### `05-data-handling-and-security/` — Data and Security

| Document                            | Description                                                                                                                                             |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `00-data-storage-and-encryption.md` | Data categories (PII, FTI), why JSON blob storage, envelope encryption architecture, S3 artifact encryption, authentication delegation, PII protections |

#### `06-mef-submission-pipeline/` — E-Filing Pipeline

| Document                     | Description                                                                                                                                               |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `00-submission-lifecycle.md` | Complete submission flow (10 steps), key identifiers, resubmission schema, MeF error handling, status service scaling concerns, EFIN/submission ID format |
| `01-pdf-generation.md`       | PDF template system: form vs table types, YAML configuration format, PdfToYaml utility, pseudo-facts, multi-language support, annual update challenges    |

#### `07-state-tax-integration/` — State Tax Data Transfer

| Document                 | Description                                                                                                                                        |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `00-state-api-design.md` | State API architecture, authorization flow, JWT+certificate authentication, endpoint descriptions, error codes, security model, enriched JSON data |

#### `08-testing-strategy/` — Testing Approach

| Document                 | Description                                                                                                                                        |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `00-testing-approach.md` | Four testing methods (completeness, correctness, flow navigation, snapshots), scenarios, static analysis, backend testing, test commands reference |

#### `09-infrastructure-and-deployment/` — Infrastructure

| Document                        | Description                                                                                                                                                 |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `00-infrastructure-overview.md` | Docker Compose services, recommended dev workflow, AWS services, environment variables, database migrations, build pipeline, code quality tools, monitoring |

#### `10-design-decisions/` — Architectural Decisions

| Document                                      | Description                                                                                                                                                                                                                                                      |
| --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `00-key-architectural-decisions.md`           | 12 major decisions with full rationale: Fact Graph, Java+Scala, React SPA, flow configuration, modules, encryption, blob storage, YAML translations, Astro screener, WCAG 2.2, resubmission schema, data import. Each with "verdict for replacement" assessment. |
| `01-organizational-process-and-governance.md` | RFC vs ADR process, DRE model, why process evolved, tax expert involvement, content review pipeline, IRS reference materials                                                                                                                                     |

## Key Insight

DirectFile's most important architectural innovation is the **Fact Graph** — a declarative, XML-based knowledge graph that models tax rules as a dependency graph of facts. This graph is the single source of truth for all tax computation, screen flow logic, and form generation. Understanding this system is critical to understanding every other part of the application.

## Document Count

- **17 documents** across **11 topic directories**
- Covers architecture, tax logic, frontend, tax domain, security, e-filing, state integration, testing, infrastructure, design decisions, and recommendations

## Audience

- **Primary**: AI agents tasked with designing and building a replacement tax filing application
- **Secondary**: Human developers and architects who need to understand the DirectFile design

## Source Material

All analysis is based on the open-source DirectFile repository at `/home/elias/Codespace/direct-file/`. Key source locations referenced throughout:

| Path                                                        | Content                                |
| ----------------------------------------------------------- | -------------------------------------- |
| `direct-file/backend/src/main/resources/tax/`               | Fact Dictionary XML modules (37 files) |
| `direct-file/df-client/df-client-app/src/flow/`             | Flow configuration (TSX)               |
| `direct-file/df-client/df-client-app/src/locales/`          | Translation files (YAML)               |
| `direct-file/fact-graph-scala/`                             | Scala Fact Graph engine                |
| `direct-file/backend/src/main/java/gov/irs/directfile/api/` | Backend Java API                       |
| `direct-file/state-api/`                                    | State API service                      |
| `docs/adr/`                                                 | Architecture Decision Records          |
| `docs/rfc/`                                                 | Requests for Comments                  |
| `docs/engineering/`                                         | Engineering documentation              |

Some code was redacted from the public release due to PII/FTI/SBU restrictions.
