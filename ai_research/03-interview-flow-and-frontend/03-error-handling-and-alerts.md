# Error Handling and Alert System

## Overview

DirectFile has a multi-layered error handling system that surfaces problems to users in contextually appropriate ways. Errors range from simple field validation messages to MeF rejection alerts that guide users through correcting their returns after a failed submission.

## Alert Types

DirectFile defines several distinct alert types, each serving a different purpose and appearing in different contexts:

### 1. Field Validation Errors
**Where they appear**: Inline, directly below the input field
**When they trigger**: When user input fails type validation (wrong format, out of range, missing required value)

Field validation uses **standardized error message patterns** defined in the ADR `adr_error-messages.md`. The team chose to create a fixed set of patterns rather than field-specific error strings because:
- **Consistency**: Every dollar field shows the same format error message
- **Translation efficiency**: Fewer unique strings = fewer translation costs
- **Implementation speed**: Developers pick from a pattern library instead of writing custom messages

#### Standard Error Patterns

| Validation Type | Pattern Example |
|----------------|-----------------|
| Required field | "Enter your [field name]" |
| Match/re-enter | "[Field name] doesn't match. Check your entry and try again." |
| Length | "[Field name] must be [N] characters" |
| Amount | "Enter an amount between $[min] and $[max]" |
| Date | "Enter a valid date" |
| Allow list | "Select a [field name]" |
| Format | "Enter [field name] in this format: [example]" |

### 2. DFAlert (Screen-Level Alerts)
**Where they appear**: On a specific screen, above or below the content
**When they trigger**: Based on a condition in the fact graph
**Purpose**: Provide contextual information, warnings, or instructions on a specific screen

```tsx
<DFAlert type="warning" i18nKey="alert.w2.boxMismatch">
  {/* Warning content */}
</DFAlert>
```

DFAlerts are simple — they show content on a single screen. They don't bubble up to the checklist or data view.

### 3. TaxReturnAlert (Section-Level Alerts)
**Where they appear**: On the screen where they're defined, AND in the checklist, AND in the data view
**When they trigger**: Based on a condition in the fact graph
**Types**:
- `type="error"`: **Blocking** — prevents submission until resolved
- `type="warning"`: **Non-blocking** — allows submission but warns the user

TaxReturnAlerts are the primary mechanism for surfacing problems that the user needs to address before filing. They aggregate across the entire return and appear in the checklist as a count of outstanding issues.

### 4. MefAlert (Rejection-Triggered Alerts)
**Where they appear**: On the relevant screen, in the checklist, and in the data view — BUT only after a MeF rejection
**When they trigger**: When MeF rejects the return with a specific error code
**Purpose**: Guide the user to fix the specific issue that caused MeF to reject their return

```tsx
<MefAlert mefErrorCode="IND-180-01" type="error">
  {/* Instructions to enter IP PIN */}
</MefAlert>
```

MefAlerts are dormant during initial filing. They only activate after a rejection. The error code matching allows precise routing: each MeF error code maps to a specific screen and specific correction instructions.

### 5. Assertion (Data View Assertions)
**Where they appear**: Only in the data view summary page
**When they trigger**: Based on conditions in the fact graph
**Purpose**: Provide status messages about completed sections (e.g., "You qualify for EITC" or "This dependent doesn't qualify for CTC")

```tsx
<Assertion type="success" i18nKey="assertion.eitc.qualifies" condition={{operator: 'isTrue', fact: '/isEligibleForEitc'}} />
```

Assertion types:
- `success`: Green — positive outcome (qualified for credit, no issues found)
- `warning`: Yellow — something to review
- `inactive`: Gray — not applicable
- `info`: Blue — informational

### 6. FactAssertion / FactResultAssertion (Collection Item Alerts)
**Where they appear**: On individual collection items in data views (e.g., each dependent's card)
**Purpose**: Show per-item status (e.g., "This child qualifies for CTC" or "This dependent doesn't meet the age test")

These are used heavily in the dependents section where each dependent may qualify for different credits with different outcomes.

## Alert Aggregation

Alerts aggregate from specific to general:

```
Screen-level alert
    ↓ bubbles to
SubSubcategory data view section
    ↓ bubbles to  
Subcategory data view page
    ↓ bubbles to
Category checklist item
    ↓ bubbles to
Overall checklist (count of issues)
```

The `alertAggregatorType` screen property controls how alerts are collected:
- `"screen"`: Alerts from this screen only
- `"sections"`: Alerts from all screens in the section

## Error Flow After MeF Rejection

When MeF rejects a return, the error handling flow is:

1. **Status service detects rejection** with one or more error codes
2. **Backend receives status change** and stores rejection codes in SubmissionEvent
3. **Frontend polls status** and receives the rejection with error codes
4. **MefAlert conditions activate** — the matching MefAlert(s) become visible
5. **Checklist shows error count** — user sees "1 error needs your attention"
6. **User navigates to the flagged screen** — sees the MefAlert with correction instructions
7. **User makes the correction** — MefAlert condition may resolve (but some require resubmission to clear)
8. **User resubmits** — new submission attempt with corrected data

### Common MeF Rejection Codes

| Code | Meaning | Correction |
|------|---------|-----------|
| `IND-180-01` | Missing IP PIN | Enter IP PIN on the signing screen |
| `IND-031-04` | SSN already filed | Contact IRS (possible identity theft) |
| `IND-181-01` | IP PIN doesn't match | Re-enter correct IP PIN |
| Various `SEIC-*` | EITC validation errors | Review qualifying child information |

## Error Message Internationalization

All error messages go through the i18next translation system:
- Error message patterns are defined as translation keys
- Field-specific details (field name, limits) are interpolated into the patterns
- Both English and Spanish versions exist for every error message
- This ensures consistent, professional error messages in both languages

## Validation Architecture

### Client-Side Validation
- Type-level validation (format, length, range) happens immediately as the user types
- Fact graph completeness is checked when navigating between screens
- Collection-level validation (e.g., at least one W-2) is checked at section boundaries

### Server-Side Validation
- The backend re-validates all data before submission
- TIN validation can be enabled/disabled (`DF_TIN_VALIDATION_ENABLED`)
- Email validation can be enabled/disabled (`DF_EMAIL_VALIDATION_ENABLED`)
- XML schema validation catches structural errors before MeF submission

### MeF Validation
- MeF performs its own validation against federal tax rules
- MeF rejections represent errors that the client/server validation didn't catch
- The MefAlert system handles these post-submission errors

## Implications for a Replacement System

1. **Standardize error message patterns**: Don't write custom error messages per field. Use a pattern library that's consistent, translatable, and maintainable.

2. **Design for post-submission errors**: MeF (or any e-filing system) will reject returns for reasons the client can't predict. Build the alert system to handle these from day one.

3. **Aggregate alerts from specific to general**: Users need to see both the big picture ("2 errors need attention") and the specific context ("Enter your IP PIN on the signing page").

4. **Separate blocking from non-blocking alerts**: Some issues must be fixed before filing (errors); others are informational (warnings). Make this distinction clear in the UI.

5. **Map rejection codes to correction screens**: When the e-filing system rejects a return, the user should be guided directly to the screen where they can fix the problem, not left to figure it out themselves.
