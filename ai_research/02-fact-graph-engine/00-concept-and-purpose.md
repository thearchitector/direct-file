# The Fact Graph: Concept and Purpose

## What Is the Fact Graph?

The Fact Graph is the central data structure and computation engine of DirectFile. It is a **declarative, XML-based knowledge graph** that models tax rules as a network of interconnected facts. Think of it like a massive spreadsheet: some cells contain user-entered data, and other cells contain formulas that compute values from those inputs and from each other.

The academic foundation for this approach is described in the paper at https://arxiv.org/pdf/2009.06103, which argues that graph structures are well-suited for modeling tax calculations.

## Why a Graph?

Tax law is fundamentally a dependency graph. Consider a simple example:

```
Adjusted Gross Income (AGI) = Total Income - Adjustments
Total Income = Wages + Interest + Unemployment + Social Security (taxable portion) + ...
Wages = Sum of all W-2 Box 1 values
Taxable Social Security = f(Total SS Benefits, Combined Income, Filing Status)
Combined Income = AGI/2 + Nontaxable Interest + SS Benefits/2
```

Notice that:
1. Each value depends on other values
2. Some dependencies are circular (AGI depends on SS, which depends on AGI) — requiring iterative resolution
3. Some values are user-entered (W-2 wages), others are computed (AGI)
4. Whether a value is even relevant depends on other values (Social Security calculation only applies if the user has SS income)
5. The same value may be needed by many different downstream calculations (AGI is used for dozens of eligibility tests)

A graph naturally represents these relationships. Each fact is a node, and dependencies are edges. The graph engine traverses the dependency tree to compute any derived value, automatically handling incomplete data, conditional logic, and circular references.

## The Two Types of Facts

### Writable Facts
Facts that are directly entered by the user through the interview UI. Examples:
- `/filers/*/firstName` — The user's first name
- `/formW2s/*/wages` — Box 1 of a W-2 form
- `/filingStatus` — The user's chosen filing status

Writable facts have a defined type (String, Dollar, Boolean, Date, Enum, etc.) and may have validation constraints (min/max values, length limits, format requirements).

### Derived Facts
Facts that are computed from other facts using declarative rules. Examples:
- `/totalWages` — Sum of all W-2 wages
- `/adjustedGrossIncome` — Computed from total income minus adjustments
- `/isEligibleForEitc` — Boolean computed from income, filing status, qualifying children, etc.

Derived facts are defined using XML computation nodes (described in the next document) and can depend on any combination of writable facts and other derived facts.

## Completeness: The Core Concept

The most important concept in the Fact Graph is **completeness**. Every fact is either **complete** (has a known, valid value) or **incomplete** (value is unknown or invalid).

- A **writable fact** is incomplete until the user provides a valid value
- A **derived fact** is incomplete if any of its dependencies are incomplete (unless the derivation explicitly handles incompleteness)

Completeness propagates through the graph: if a writable fact that feeds into AGI is incomplete, then AGI is incomplete, and everything that depends on AGI is also incomplete.

This is crucial for the tax filing use case because:
1. **A partially completed return is always valid** — The graph doesn't crash or produce errors when data is missing; it simply marks downstream values as incomplete
2. **The UI can determine what's done and what's not** — The checklist shows which sections are complete by checking whether the culminating facts for each section are complete
3. **Submission blocking** — The return cannot be submitted until all required facts are complete

## The Fact Dictionary

The Fact Dictionary is the **schema** of the Fact Graph. It defines:
- Every fact that exists (path, name, description)
- Whether each fact is writable or derived
- The type of each writable fact and its constraints
- The computation formula for each derived fact
- Which module each fact belongs to
- Which facts are exported for cross-module use

The Fact Dictionary is defined in XML files located at:
`direct-file/backend/src/main/resources/tax/`

As of the current codebase, the dictionary contains approximately **800+ facts** organized into **37 modules** (XML files).

## Modules: Organizing Facts

Facts are organized into modules, where each XML file represents one module. Examples:

| Module File | Domain Area | Example Facts |
|------------|-------------|---------------|
| `filers.xml` | Personal info about taxpayers | `/filers/*/firstName`, `/filers/*/ssn`, `/filers/*/dateOfBirth` |
| `filingStatus.xml` | Filing status determination | `/filingStatus`, `/isMarriedFilingJointly` |
| `formW2s.xml` | W-2 wage income | `/formW2s/*/wages`, `/formW2s/*/federalTaxWithheld` |
| `interest.xml` | Interest income (1099-INT) | `/interestReports/*/interestIncome` |
| `eitc.xml` | Earned Income Tax Credit | `/isEligibleForEitc`, `/eitcAmount` |
| `ctcOdc.xml` | Child Tax Credit | `/childTaxCreditAmount` |
| `taxCalculations.xml` | Final tax computation | `/taxOwed`, `/totalRefund`, `/adjustedGrossIncome` |
| `standardDeduction.xml` | Standard deduction | `/standardDeductionAmount` |
| `flow.xml` | Flow control facts | `/flowIsKnockedOut`, `/flowIsComplete` |

### Module Visibility Rules

By default, facts are **private to their module**. This is a critical design decision that prevents spaghetti dependencies:

- A fact in `eitc.xml` CANNOT directly reference a private fact from `filers.xml`
- To use a fact across modules, it must be explicitly **exported** with `<Export downstreamFacts="true" />`
- Facts needed by MeF XML generation must be exported with `<Export mef="true" />`
- The build system enforces these boundaries at compile time — if a module references a non-exported fact from another module, the build fails

### Why Modules Matter

Before modules were introduced, all ~800 facts lived in a single file and any fact could depend on any other fact. This led to:
- Bugs where changes in one area (e.g., spouse section) would break unrelated areas (e.g., deductions)
- Inability to reason about which facts were safe to use downstream
- Testing nightmares — you couldn't test one section in isolation

Modules create well-defined boundaries with ~140 "culminating facts" that serve as the API between modules. These exported facts are the ones that determine tax tests, eligibility, and computations. They have high test coverage and are guaranteed to be complete by specific points in the flow.

## Collections: Handling Multiple Items

Some facts exist in collections — repeated groups of facts for things like:
- Multiple W-2 forms (`/formW2s/*`)
- Multiple dependents (`/familyAndHousehold/*`)
- Multiple 1099 forms (`/interestReports/*`)
- Filers (`/filers/*` — primary and optional secondary/spouse)

Collections use a wildcard `*` in their path, and each item in the collection is identified by a UUID. For example:
- `/formW2s/#a1b2c3d4/wages` — Wages from a specific W-2
- `/filers/#e5f6g7h8/firstName` — First name of a specific filer

Collections can be filtered, iterated, and aggregated in derived facts. Aliases are used to allow one collection's facts to reference another collection's context.

## How the Fact Graph Is Used Throughout the System

| System Component | How It Uses the Fact Graph |
|-----------------|---------------------------|
| **Frontend Interview** | Reads facts to populate screen fields; writes facts when user enters data; checks fact completeness to determine flow navigation |
| **Flow/Screen Config** | Uses boolean derived facts as conditions to determine which screens to show |
| **Checklist** | Checks completeness of section-level facts to show progress |
| **Tax Computation** | All tax math is expressed as derived facts |
| **PDF Generation** | Reads fact values to populate PDF form fields |
| **MeF XML Generation** | Reads exported facts to generate IRS-compliant XML |
| **Validation** | Uses TaxReturnAlert facts to detect and surface errors |
| **Knockouts** | Uses boolean facts to determine if a user's situation is unsupported |
| **State API** | Exports fact values in enriched JSON format for state tax tools |

## Key Design Principles

1. **Single Source of Truth**: All tax-related state lives in the fact graph. There is no separate "tax calculation service" — the graph IS the calculation.

2. **Declarative over Imperative**: Tax rules are declared as XML configuration, not coded as imperative logic. This makes them reviewable by tax experts who may not be programmers.

3. **Isomorphic Execution**: The same fact graph code runs on both client (JavaScript) and server (JVM). This guarantees consistency and enables real-time client-side computation.

4. **Graceful Incompleteness**: The graph is designed to reason about incomplete information. A partially-filled return doesn't crash — it simply has incomplete derived facts.

5. **Modularity**: Facts are organized into modules with explicit export boundaries, enabling independent testing and reducing unintended coupling.

6. **Configuration as Code**: The fact dictionary XML files are version-controlled, code-reviewed, and tested like any other code change.
