# Interview Flow Architecture

## Overview

DirectFile presents tax questions as an "interview" — a guided sequence of screens that adapts based on the user's answers. The flow is the configuration that defines which screens exist, what order they appear in, what data they collect, and under what conditions they are shown or hidden.

The flow is currently defined as JSX/TSX code (React components), but the design intention is that it should eventually become pure configuration editable by tax experts who are not programmers. Every element in the flow is serializable.

## The Hierarchy: Category → Subcategory → SubSubcategory → Screen

The interview is organized into a nested hierarchy:

```
Flow
├── Category: "you-and-your-family"
│   ├── Subcategory: "about-you"
│   │   ├── SubSubcategory: "basic-info"
│   │   │   ├── Screen: "about-you-intro"
│   │   │   ├── Screen: "about-you-basic-info"
│   │   │   └── Screen: "about-you-contact-info"
│   │   └── SubSubcategory: "identity"
│   │       ├── Screen: "about-you-ssn"
│   │       └── Screen: "about-you-ip-pin"
│   ├── Subcategory: "spouse"
│   │   └── ...
│   ├── Subcategory: "dependents"
│   │   └── ...
│   └── Subcategory: "filing-status"
│       └── ...
├── Category: "income"
│   ├── Subcategory: "income-sources"
│   ├── Subcategory: "jobs" (W-2s)
│   ├── Subcategory: "unemployment"
│   ├── Subcategory: "interest"
│   ├── Subcategory: "retirement"
│   ├── Subcategory: "social-security"
│   └── Subcategory: "total-income-summary"
├── Category: "credits-and-deductions"
│   ├── Subcategory: "deductions"
│   └── Subcategory: "credits"
├── Category: "your-taxes"
│   ├── Subcategory: "estimated-taxes"
│   ├── Subcategory: "amount"
│   ├── Subcategory: "payment-method"
│   └── Subcategory: "other-preferences"
└── Category: "complete" (sign-and-submit)
    ├── Subcategory: "review"
    ├── Subcategory: "sign"
    ├── Subcategory: "print-and-mail"
    └── Subcategory: "submit"
```

### What Each Level Represents

| Level | UI Representation | Purpose |
|-------|------------------|---------|
| **Category** | Major section headers in the checklist | Groups related topics (e.g., all income types) |
| **Subcategory** | Checklist items with 1-2 data points | A logical unit of work (e.g., "W-2 Income") |
| **SubSubcategory** | Sections within a data review page | A reviewable/editable chunk of a subcategory |
| **Screen** | Individual question pages | One page of the interview with fields and content |

### The Checklist

The checklist is a progress tracker that shows all categories and subcategories. It computes progress by checking which facts are complete:

- Each subcategory has a `completeIf` condition — a boolean fact (or set of facts) that must be true for that section to be marked complete
- The checklist reads the flow configuration and the fact graph state to build itself
- Users can jump to any section from the checklist, not just proceed linearly

### Data Views

Each subcategory with multiple screens has a **data view** — a summary page showing all the data the user entered in that section. Data views have:
- Grouped display of entered values organized by subsubcategory
- Edit buttons that take the user back to the relevant screens in "review mode"
- Assertions and alerts that surface warnings or errors

Some simple subcategories use `actAsDataView={true}` on a single screen, meaning that screen itself serves as both the data entry point and the summary view.

## Flow Configuration File Structure

The main flow is defined in:
`df-client/df-client-app/src/flow/flow.tsx`

It imports subcategory definitions from chunk files:
`df-client/df-client-app/src/flow/flow-chunks/`

```
flow-chunks/
├── you-and-your-family/
│   ├── AboutYouSubcategory.tsx
│   ├── SpouseSubcategory.tsx
│   ├── FamilyAndHHSubcategory.tsx
│   └── FilingStatusSubcategory.tsx
├── income/
│   ├── IncomeSourcesSubcategory.tsx
│   ├── JobIncomeSubcategory.tsx
│   ├── UnemploymentIncomeSubcategory.tsx
│   ├── InterestIncomeSubcategory.tsx
│   ├── RetirementIncomeSubcategory.tsx
│   ├── SocialSecurityIncomeSubcategory.tsx
│   ├── AlaskaPermanentFundSubcategory.tsx
│   ├── DependentCareBenefitsSubcategory.tsx
│   ├── HsaSubcategory.tsx
│   └── TotalIncomeSummarySubcategory.tsx
├── credits-and-deductions/
│   ├── DeductionsSubcategory.tsx
│   └── CreditsSubcategory.tsx
├── your-taxes/
│   ├── EstimatedTaxesSubcategory.tsx
│   ├── AmountSubcategory.tsx
│   ├── PaymentMethodSubcategory.tsx
│   └── OtherPreferencesSubcategory.tsx
└── sign-and-submit/
    ├── ReviewSubcategory.tsx
    ├── SignSubcategory.tsx
    ├── PrintAndMailSubcategory.tsx
    └── SubmitSubcategory.tsx
```

## Flow Declaration Types

The flow uses a set of React component types that serve as declarative configuration (they don't render anything themselves):

### `<Flow>` — Top-level container
Contains one or more `<Category>` children.

### `<Category route="...">` — Major section
- `route`: URL segment (e.g., `you-and-your-family`)
- Contains `<Subcategory>` children

### `<Subcategory route="..." completeIf={...}>` — Checklist item
- `route`: URL segment
- `completeIf`: Fact path(s) that must be complete for this section to be done
- `collectionContext`: If the entire subcategory relates to a collection (e.g., `/formW2s`)
- `displayOnlyIf`: Conditions for showing this subcategory (e.g., only show retirement if user has retirement income)
- `skipDataView`: If true, this subcategory doesn't have a separate summary page
- Contains `<Screen>`, `<SubSubcategory>`, `<CollectionLoop>`, and `<Gate>` children

### `<Screen route="..." condition={...}>` — Individual question page
- `route`: URL segment (becomes part of the full URL path)
- `condition`: Optional boolean fact that must be true for this screen to appear
- `actAsDataView`: This screen serves double duty as the data review page
- `isKnockout`: If this screen's condition is met, the user is knocked out of DirectFile
- `routeAutomatically`: Whether navigation to the next screen happens automatically
- Contains content declarations (see below)

### `<Gate condition={...}>` — Conditional wrapper
Wraps screens or other elements, only showing them if the condition is true. Gates can be nested.

```tsx
<Gate condition={{ operator: 'isTrue', fact: '/hasSpouse' }}>
  <Screen route="spouse-info">...</Screen>
  <Screen route="spouse-income">...</Screen>
</Gate>
```

### `<CollectionLoop loopName="..." collection="...">` — Repeating section
Handles collections of items (multiple W-2s, multiple dependents, etc.):
- `loopName`: Identifier for this loop
- `collection`: Fact path of the collection (e.g., `/formW2s`)
- `autoIterate`: Whether to automatically move to the next item
- `donePath`: Writable fact that tracks whether the user has finished this loop
- Contains screens that repeat for each item in the collection

### `<SubSubcategory route="..." editable={...}>` — Reviewable section
Groups screens into an editable chunk within a data view:
- `route`: URL segment
- `editable`: Whether the edit button appears in the data view (default true)
- `hidden`: Whether this section is hidden in the data view

## Conditions: How Flow Logic Works

Screens and gates use **conditions** to determine visibility. A condition references a boolean fact in the fact graph:

```typescript
type RawCondition = {
  operator: 'isTrue' | 'isFalse' | 'isComplete' | 'isIncomplete';
  fact: string; // Path to a boolean or any fact
};
```

- `isTrue`: The boolean fact must be complete AND true
- `isFalse`: The boolean fact must be complete AND false
- `isComplete`: The fact (any type) must have a complete value
- `isIncomplete`: The fact must NOT have a complete value

**Critical design constraint**: All flow conditions must be simple boolean checks against the fact graph. Any complex logic (e.g., "show this screen if income > $50,000 AND filing status is MFJ") must be computed as a derived boolean fact in the fact dictionary, then referenced as a simple `isTrue` condition.

This separation keeps the flow configuration simple and pushes all complex logic into the testable, type-safe fact graph.

## Screen Content Declarations

Within a `<Screen>`, various content declaration components define what appears on the page:

### `<Heading>` — Page title (exactly one per screen)
The primary heading text for the screen.

### `<InfoDisplay>` — Informational text (zero or more)
Explanatory content, instructions, or context. Can contain structured HTML via YAML translation references.

### Fact/Field Components — Data entry fields
- `<Dollar path="..." />` — Dollar amount input
- `<GenericString path="..." />` — Text input
- `<DateOfBirth path="..." />` — Date picker for birth dates
- `<Boolean path="..." />` — Yes/No radio buttons
- `<Enum path="..." />` — Single-select dropdown or radio group
- `<MultiEnum path="..." />` — Multi-select checkboxes
- `<Tin path="..." />` — SSN/TIN input with masking
- `<Ein path="..." />` — EIN input
- `<Pin path="..." />` — IP PIN input

Each field:
- Reads its current value from the fact graph on render
- Writes its value to the fact graph when the user enters valid data
- Shows validation errors based on fact type constraints
- Can be marked `required='false'` for optional fields

### `<SetFactAction path="..." value="..." />` — Automatic fact setting
Sets a fact value when the user reaches this screen, regardless of their input. Used for tracking progress or setting derived workflow values.

### Alert Components
- `<DFAlert>` — Simple alert content on a screen
- `<TaxReturnAlert type="error|warning">` — Blocking/non-blocking alerts that bubble to checklist and data view
- `<MefAlert mefErrorCode="...">` — Alerts triggered by MeF rejection codes
- `<Assertion>` — Alerts that appear only in data view
- `<FactAssertion>` / `<FactResultAssertion>` — Alerts on collection items (e.g., dependent qualification results)

## Screen Navigation

Navigation between screens is handled by `getNextScreen.ts`:

1. Given the current screen and fact graph state, determine the next screen
2. Walk forward through the flow, checking each screen's condition
3. Skip screens whose conditions are not met
4. If in review mode, return to the data view after the current subsubcategory
5. Handle collection loops (iterate to next item or exit loop)

The URL structure mirrors the flow hierarchy:
```
/flow/you-and-your-family/about-you/about-you-intro
/flow/income/jobs?/formW2s=#uuid
/flow/credits-and-deductions/credits/eitc-qualifying
```

Collection items include a query parameter with the collection item's UUID.

## Knockouts

A knockout screen is shown when the user's situation is not supported by DirectFile. Knockout screens:
1. Have `isKnockout={true}` set on the screen
2. Are associated with a fact in the `/flowIsKnockedOut` derived fact (in `flow.xml`)
3. When the knockout condition is true, the user sees the screen explaining why they can't use DirectFile
4. The user is directed to alternative filing options

Examples of knockout conditions:
- User lived in multiple states
- User has self-employment income
- User wants to itemize deductions
- User is a non-resident alien

## Batches: Feature Flags for Tax Scope

DirectFile uses a "batches" system to control which tax features are enabled. Each batch (e.g., `hsa-0`, `ptc-0`, `edc-0`) represents a tax scope item that can be independently toggled on or off. This allows:
- Gradual rollout of new tax features
- Different feature sets for different environments
- Easy A/B testing of new capabilities

The batch system is defined in `batches.ts` and integrates with the flow by conditionally showing/hiding subcategories.
