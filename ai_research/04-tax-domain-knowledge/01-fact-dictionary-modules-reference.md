# Fact Dictionary Modules Reference

## Overview

This document catalogs every module (XML file) in the Fact Dictionary, describing what tax domain area it covers, its approximate size, and what it exports for use by other modules. This serves as a map of the entire tax logic surface area.

All files are located at: `direct-file/backend/src/main/resources/tax/`

## Module Catalog

### Core Personal Information

| Module | File Size | Description |
|--------|-----------|-------------|
| `filers.xml` | 178 KB | Primary and secondary filer personal information: names, SSNs, dates of birth, addresses, contact info, occupation, identity protection PINs, prior year AGI, citizenship status, state residency, disability status, blindness, age-based determinations. The largest module, reflecting that filer info feeds into nearly every other calculation. |
| `spouseSection.xml` | 29 KB | Spouse-specific questions and determinations for married filers: whether filing jointly, spouse's presence in household, spouse as dependent. Builds on filer data to make spouse-related eligibility decisions. |
| `filingStatus.xml` | 37 KB | Filing status determination logic. Computes which filing status the taxpayer qualifies for based on marital status, dependents, household costs, and spouse status. One of the most critical derived determinations — affects tax brackets, deduction amounts, and credit eligibility. |
| `familyAndHousehold.xml` | 176 KB | Dependents and household members: qualifying child tests, qualifying relative tests, relationship determinations, residency tests, support tests, age tests, student status, disability. Second largest module due to the complexity of dependent qualification rules across different credits. |

### Income Modules

| Module | File Size | Description |
|--------|-----------|-------------|
| `formW2s.xml` | 213 KB | W-2 wage income processing. The largest module by file size. Handles all W-2 boxes (1-14), employer info, multiple W-2 aggregation, withholding calculations, Box 12 code processing (retirement contributions, HSA, etc.), Box 13 checkboxes (statutory employee, retirement plan). |
| `interest.xml` | 51 KB | Interest income (1099-INT). Handles multiple interest reports, aggregation, special treatment of US savings bond interest, tax-exempt interest, early withdrawal penalties. Determines if Schedule B is required (>$1,500 threshold). |
| `unemployment.xml` | 26 KB | Unemployment compensation (1099-G). Handles state unemployment benefits, tax withholding on unemployment, aggregation of multiple 1099-Gs. |
| `socialSecurity.xml` | 31 KB | Social Security benefits (SSA-1099). Implements the complex taxability formula: computes "combined income" (half of SS benefits + other income + tax-exempt interest), determines taxable percentage (0%, 50%, or 85%), handles lump-sum elections. |
| `form1099Rs.xml` | 112 KB | Retirement distributions (1099-R). Handles pensions, annuities, IRA distributions, Roth conversions. Distribution code processing (Box 7) determines tax treatment. Handles rollovers, early distribution penalties. Complex module due to the many types of retirement accounts and distribution scenarios. |
| `form1099Misc.xml` | 13 KB | Miscellaneous income (1099-MISC). Handles Box 3 (other income) for prizes, awards, and other non-employment income. |
| `income.xml` | 62 KB | Income aggregation and summary. Combines all income types into total income, adjusted gross income. Implements the income summary computations that appear on Form 1040 Lines 1-11. |
| `imported.xml` | 25 KB | Data Import integration. Handles facts that are pre-populated from IRS internal systems (when available). Manages the distinction between imported data and user-entered data, including user verification flags. |

### Health-Related Modules

| Module | File Size | Description |
|--------|-----------|-------------|
| `hsa.xml` | 139 KB | Health Savings Account (Form 8889). One of the more complex modules. Handles HSA contributions (employer and personal), distributions, qualification rules (must have High Deductible Health Plan), contribution limits (self-only vs. family), excess contribution penalties, catch-up contributions for age 55+. |
| `ptc.xml` | 304 KB | Premium Tax Credit (Form 8962). The largest module by far. Handles Marketplace health insurance reconciliation: advance PTC received, actual PTC entitled, repayment of excess. Extremely complex due to household size calculations, income percentage calculations, applicable benchmarks, and the interaction between PTC and other tax calculations. |

### Credits and Deductions Modules

| Module | File Size | Description |
|--------|-----------|-------------|
| `eitc.xml` | 98 KB | Earned Income Tax Credit. One of the most impactful credits for low-income filers. Implements the EITC computation with phase-in and phase-out ranges based on filing status and number of qualifying children (0, 1, 2, 3+). Investment income test, AGI test, earned income calculations. |
| `ctcOdc.xml` | 51 KB | Child Tax Credit and Other Dependent Credit. CTC: up to $2,000 per qualifying child under 17 with SSN. ODC: up to $500 per other qualifying dependent. Phases out at higher incomes. Computes Additional Child Tax Credit (refundable portion). |
| `cdcc.xml` | 131 KB | Child and Dependent Care Credit (Form 2441). Credit for care expenses enabling work. Qualifying person determination, expense limits ($3,000/$6,000), credit percentage (20-35% based on AGI), employer-provided benefits exclusion. |
| `elderlyAndDisabled.xml` | 62 KB | Credit for the Elderly or Disabled (Schedule R). Available to age 65+ or permanently disabled with disability income. Complex eligibility and computation based on filing status, income levels, and nontaxable Social Security. |
| `saversCredits.xml` | 57 KB | Saver's Credit (Form 8880). Credit for retirement savings contributions. 10%, 20%, or 50% credit rate based on AGI. Maximum contribution considered: $2,000 per person. |
| `dependentsBenefitSplit.xml` | 23 KB | Dependent care benefits. Handles the exclusion of employer-provided dependent care benefits (from W-2 Box 10) and the interaction with the CDCC. |
| `dependentsRelationship.xml` | 46 KB | Relationship determination rules used across multiple credits. Determines qualifying child, qualifying relative, and qualifying person status for each dependent based on relationship, age, residency, and support tests. |
| `standardDeduction.xml` | 12 KB | Standard deduction computation. Base amounts by filing status, additional amounts for age 65+ and blindness, special rules for dependents who can be claimed by others. |
| `educatorAdjustment.xml` | 9 KB | Educator expense adjustment. Up to $300 per eligible K-12 educator for classroom supplies ($600 for MFJ with two educators). |
| `studentLoanAdjustment.xml` | 16 KB | Student loan interest deduction. Up to $2,500, with income-based phase-out. Not available for MFS filers. |

### Tax Computation and Payment

| Module | File Size | Description |
|--------|-----------|-------------|
| `taxCalculations.xml` | 63 KB | Final tax computation. Implements tax bracket calculations, combines all credits, computes total tax, subtracts payments and credits, determines refund or amount owed. This is the culminating computation module. |
| `estimatedPayments.xml` | 9 KB | Estimated tax payments made during the year. Allows entry of quarterly payments that reduce tax owed. |
| `paymentMethod.xml` | 13 KB | Refund delivery and payment method preferences. Direct deposit (routing + account number) or paper check. Electronic payment or mailed check for amounts owed. |
| `refundPrefs.xml` | 17 KB | Refund preferences and options. How the taxpayer wants to receive their refund. |

### Signing and Submission

| Module | File Size | Description |
|--------|-----------|-------------|
| `signing.xml` | 51 KB | Electronic signature and identity verification. IP PIN handling, self-select PIN, date of birth verification, prior year AGI verification. The signing module is complex because MeF requires specific signature data. |

### Flow and System Control

| Module | File Size | Description |
|--------|-----------|-------------|
| `flow.xml` | 35 KB | Flow control facts. Boolean facts that determine which sections of the interview are shown, knockout conditions, section completeness flags. These are the "glue" facts that connect tax logic to the UI flow. |
| `constants.xml` | 33 KB | Tax year constants. Dollar thresholds, percentage rates, phase-out ranges, and other values that change with each tax year. Centralizing these in one module makes annual updates straightforward. |
| `validations.xml` | 1 KB | Cross-cutting validation rules. |
| `copy.xml` | 2 KB | Copy/convenience facts that make other modules more readable. |
| `mefTypes.xml` | 2 KB | MeF-specific type definitions for XML generation. |

### Schedule Modules

| Module | File Size | Description |
|--------|-----------|-------------|
| `schedule2.xml` | 1 KB | Schedule 2 (Additional Taxes). Minimal — limited self-employment tax support. |
| `schedule3.xml` | 8 KB | Schedule 3 (Additional Credits and Payments). Saver's Credit, estimated tax payment credits. |

## Module Dependency Map (Simplified)

```
                          filers.xml ──────────────────┐
                              │                         │
                    ┌─────────┼──────────┐              │
                    ▼         ▼          ▼              │
              spouseSection  filingStatus  familyAndHousehold
                    │         │          │              │
                    └────┬────┘          │              │
                         ▼               │              │
                    ┌────────────────────┤              │
                    │                    │              │
         ┌─────────┼─────────┐          │              │
         ▼         ▼         ▼          ▼              │
      formW2s  interest  unemployment  dependentsRelationship
         │         │         │                │         │
         └────┬────┘─────────┘                │         │
              ▼                               │         │
           income ◄───────────────────────────┘         │
              │                                         │
    ┌─────────┼──────────────────────┐                  │
    ▼         ▼         ▼            ▼                  │
  eitc    ctcOdc     cdcc    standardDeduction          │
    │         │         │            │                  │
    └────┬────┘─────────┘────────────┘                  │
         ▼                                              │
    taxCalculations ◄───────────────────────────────────┘
         │
         ▼
      signing → flow.xml
```

This is a simplified view. In reality, many modules have cross-dependencies via exported facts. The `constants.xml` module is referenced by nearly every other module for threshold values.

## Key Statistics

- **Total modules**: 37 XML files
- **Total fact dictionary size**: ~2.2 MB of XML
- **Largest module**: `ptc.xml` (304 KB) — Premium Tax Credit
- **Smallest modules**: `validations.xml` (1 KB), `schedule2.xml` (1 KB)
- **Estimated total facts**: ~800+ (writable + derived)
- **Estimated exported facts**: ~140 "culminating facts"
- **Most interconnected module**: `taxCalculations.xml` — depends on exports from nearly every other module

## Annual Update Process

When tax law changes for a new year:
1. **Update `constants.xml`** — New dollar amounts, brackets, thresholds
2. **Update affected computation modules** — New rules, modified formulas
3. **Update `flow.xml`** — New knockout conditions, new eligibility rules
4. **Update PDF templates and configurations** — New IRS form versions
5. **Update translation files** — New content, updated amounts in explanatory text
6. **Run all test suites** — Verify completeness, correctness, flow navigation, and snapshots
7. **Update scenarios** — Migrate test data to reflect new tax year values
