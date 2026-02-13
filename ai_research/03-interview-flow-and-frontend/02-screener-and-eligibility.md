# Screener and Eligibility System

## Overview

The screener is the very first thing a user encounters before logging in to DirectFile. It is a set of simple yes/no eligibility questions designed to quickly determine whether a user's tax situation is supported. If any answer indicates the user doesn't qualify, they are redirected to alternative filing options before they invest time in creating an account.

## Why a Separate Application?

The screener is built as a **separate static site** (using Astro SSG), NOT as part of the React SPA. The reasons are:

### Security: Unprotected Pages
- The screener pages are NOT behind authentication — anyone can visit them without logging in
- The main React SPA IS behind authentication
- Keeping unprotected routes in a separate deployment prevents accidentally exposing protected routes or creating auth bypass bugs

### Performance: Static Content
- The screener is mostly static content with simple yes/no questions
- No tax computation, no fact graph, no dynamic data
- A static site loads instantly, which is critical for the first-impression experience
- Static pages can be served from a CDN with no server-side processing

### Deployment Independence
- Content changes to the screener (updated eligibility criteria, new tax year messaging) can be deployed independently from the main application
- No risk of screener content changes breaking the tax filing flow

### Content Management
- The screener content changes more frequently than the main application (seasonal updates, messaging changes)
- A static site generator makes content changes easier for non-engineers
- Integration with i18next provides the same translation workflow as the main app

## Technology: Astro SSG

The team evaluated three options for the screener:

### Option 1 (Rejected): Plain HTML/JS
- Too much manual work for content management
- No component reuse
- No built-in i18n support

### Option 2 (Rejected): Another React SPA
- Overkill for mostly static content
- Would require running the full React build pipeline for simple content changes
- Heavier page loads than necessary for simple yes/no pages

### Option 3 (Chosen): Astro Static Site Generator
- **Ships minimal JavaScript**: Astro is designed for content-focused sites with little interactivity
- **Component support**: Can use React components if needed for interactive elements
- **i18next integration**: Same translation framework as the main app
- **Multi-page architecture**: Natural fit for a series of eligibility questions
- **Fast builds**: Static HTML is generated at build time, not at request time

## Screener Flow

The screener walks the user through eligibility questions organized roughly as:

1. **Tax year**: "Are you filing for the current tax year?"
2. **Residency**: "Did you live in a supported state?"
3. **Filing status**: "Are you filing as Single, MFJ, HoH, etc.?"
4. **Income types**: "Did you have [unsupported income type]?" (capital gains, self-employment, etc.)
5. **Deductions**: "Do you want to itemize deductions?"
6. **Other situations**: "Do any of the following apply to you?" (foreign income, rental income, etc.)

If any answer indicates an unsupported situation, the screener:
- Shows a message explaining that DirectFile can't handle their return
- Provides links to alternative filing options (Free File, VITA, commercial tax software)
- Does NOT require the user to create an account or provide any personal information

If all answers indicate a supported situation:
- The user is directed to the authentication flow (CSP login)
- After authentication, they enter the main React SPA interview

## Knockout System (In-App)

In addition to the screener (pre-login), the main application has an in-app knockout system for situations that are only discoverable during the interview:

### How Knockouts Work in the Flow

Knockout screens are marked with `isKnockout={true}` in the flow configuration:

```tsx
<Screen route="multiple-states-knockout" isKnockout={true} condition={{operator: 'isTrue', fact: '/livedInMultipleStates'}}>
  {/* Content explaining why Direct File can't handle their return */}
</Screen>
```

When the condition becomes true (based on user input), the knockout screen is shown.

### Knockout Facts in flow.xml

The `flow.xml` module contains boolean derived facts that aggregate knockout conditions:

- `/flowIsKnockedOut` — Master knockout flag (OR of all individual knockout conditions)
- Individual knockout facts for specific situations:
  - `/hasUnsupportedIncomeType`
  - `/livedInMultipleStates`
  - `/wantsToItemize`
  - `/hasUnsupportedCredits`
  - etc.

### Knockout Placement in the Flow

Knockout screens are strategically placed at the point in the interview where the disqualifying information is collected. For example:
- The state residency knockout appears in the "About You" section, right after the user enters their state
- The income type knockout appears in the "Income Sources" section
- The itemized deduction knockout appears in the "Deductions" section

This placement minimizes the amount of work the user does before being knocked out.

### Creating a New Knockout

Adding a new knockout requires:
1. **Identify the criteria**: What tax situation is unsupported?
2. **Create or use existing facts**: Boolean fact(s) that detect the unsupported situation
3. **Add the knockout screen to the flow**: With appropriate condition and placement
4. **Write content**: Explain why the user can't use DirectFile and where to go instead
5. **Add translations**: English and Spanish versions
6. **Write tests**: Functional flow tests verifying the knockout activates correctly

## Implications for a Replacement System

1. **Pre-login eligibility screening is essential**: Don't make users create an account before discovering they can't use the system.

2. **Use a lightweight approach for the screener**: A static site or simple server-rendered pages — not the full application framework.

3. **Place in-app knockouts early in the relevant section**: Minimize wasted effort before delivering bad news.

4. **Provide actionable alternatives on knockout**: Don't just say "you can't use this" — say "here are your other options."

5. **Make knockout conditions testable**: Express them as derived boolean facts in the rules engine so they can be unit tested and snapshotted.

6. **Plan for scope expansion**: As the system supports more tax situations, some knockouts should become entry points to new features. Design the knockout system to be easily convertible to supported features.
