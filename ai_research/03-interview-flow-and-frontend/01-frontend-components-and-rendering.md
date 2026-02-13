# Frontend Components and Rendering

## Overview

The DirectFile frontend is a React 18 Single Page Application (SPA) that renders screens based on the flow configuration and fact graph state. The rendering pipeline transforms declarative flow configuration into interactive tax interview screens.

## Key Component Architecture

### App.tsx — Application Root
The root component sets up:
- React Router for URL-based navigation
- Redux store for non-tax application state
- FactGraphContext provider (the singleton fact graph instance)
- i18next provider for translations
- Authentication context
- Error boundaries

### BaseScreen.tsx — The Screen Renderer
`BaseScreen` is the central rendering component for interview screens. It:

1. **Reads the current URL** via React Router `useParams()`
2. **Looks up the screen configuration** from the flow using the URL
3. **Resolves the collection ID** if the screen is within a collection (from URL query params)
4. **Determines the next screen** callback using `getNextScreen()`
5. **Renders the `<Screen>` component** with the screen's content configuration

```
URL: /flow/income/jobs/w2-wages?/formW2s=#uuid-123
                ↓
BaseScreen reads URL params
                ↓
Looks up ScreenConfig for "w2-wages" from flow
                ↓
Resolves collection ID: formW2s → #uuid-123
                ↓
Renders <Screen> with content declarations and collection context
```

### Screen.tsx — Content Renderer
The `Screen` component takes a screen configuration and renders its content declarations as actual React components. It:
- Renders the heading
- Renders info displays
- Renders fact input fields (each field reads from and writes to the fact graph)
- Renders alerts and assertions
- Provides "Save and continue" / "Back" navigation buttons
- Executes SetFactAction side effects when the screen loads

### Fact Input Components
Each fact type has a corresponding React component for data entry:

| Fact Type | Component | UI Element |
|-----------|-----------|------------|
| Dollar | `DollarField` | Currency input with formatting |
| String | `GenericStringField` | Text input |
| Boolean | `BooleanField` | Yes/No radio buttons |
| Enum | `EnumField` | Radio button group or dropdown |
| MultiEnum | `MultiEnumField` | Checkbox group |
| Date | `DateField` | Date inputs (month/day/year) |
| Tin | `TinField` | SSN input with masking |
| Ein | `EinField` | EIN input with formatting |
| Pin | `IpPinField` | IP PIN input |

Each input component follows the same pattern:
1. On render, reads the current value from the fact graph for its path
2. On user input, validates the value against the fact type constraints
3. On valid input, writes the value to the fact graph
4. Displays validation errors using standardized error message patterns
5. Supports the `required='false'` prop for optional fields

### Checklist.tsx — Progress Tracker
The checklist component:
1. Reads the flow configuration to enumerate all categories and subcategories
2. For each subcategory, evaluates its `completeIf` condition against the fact graph
3. Renders a checklist with status indicators (not started, in progress, complete)
4. Provides navigation links to jump to any section
5. Aggregates and displays `TaxReturnAlert` and `MefAlert` items from all screens

### DataView.tsx — Section Summary
The data view component:
1. Reads the subcategory's subsubcategories and their screen configurations
2. For each screen, reads the values of facts that were collected
3. Displays the values in a structured summary format
4. Provides "Edit" buttons that navigate back to the relevant screens in review mode
5. Shows assertions and alerts relevant to the section

## State Management

### Fact Graph (Primary State — Tax Data)
The fact graph is a global singleton initialized on app startup. It is NOT managed by React state (it's opaque to React). Components access it via the `FactGraphContext`:

```typescript
const { factGraph } = useFactGraph();
```

This design means React doesn't "know" when the fact graph changes. Instead:
- Each screen is rendered fresh when navigated to (SPA page transitions)
- The screen reads the latest fact graph state on render
- Multiple concurrent parts of the UI reacting to fact graph changes would require creative engineering (but this wasn't needed because only one screen is visible at a time)

### Redux (Secondary State — Application State)
Redux stores non-tax state such as:
- Authentication/session information
- UI preferences
- Navigation history
- Feature flags

### Session Storage
Some ephemeral state is stored in browser session storage rather than Redux or the fact graph. This includes temporary UI state that shouldn't persist across sessions.

## Navigation and URL Structure

### URL Format
```
/flow/{category}/{subcategory}/{screen}[?{collection}={itemId}]
```

Examples:
- `/flow/you-and-your-family/about-you/about-you-intro`
- `/flow/income/jobs/w2-wages?/formW2s=#a1b2c3d4`
- `/flow/credits-and-deductions/credits/eitc-qualifying`

### getNextScreen Algorithm
When the user clicks "Save and continue":
1. Start from the current screen in the flow
2. Walk forward to the next screen in sequence
3. If the next screen has a `condition`, evaluate it against the fact graph
4. If the condition is false (or the screen is behind a gate that's false), skip it
5. Continue until a screen with a passing condition is found
6. If in review mode, return to the data view after the current subsubcategory
7. Handle collection loop iteration (next item, exit loop, etc.)

### Review Mode
When a user clicks "Edit" from a data view, they enter review mode. In this mode:
- They are taken to the relevant screen(s) within a subsubcategory
- After completing the screens in that subsubcategory, they return to the data view
- They do NOT continue forward through the rest of the flow
- This is controlled by the `reviewMode` query parameter

## Internationalization (i18n)

### Translation File Structure
Translations are stored in YAML files:
- `src/locales/en.yaml` — English (primary)
- `src/locales/es.yaml` — Spanish

YAML was chosen over JSON because:
- More human-readable (important for non-engineer content editors)
- Less error-prone when editing complex nested content
- Supports structured HTML representation without embedding raw tags in strings

### Translation Key Organization
Translation keys follow the screen hierarchy and content type:
```yaml
/info/you-and-your-family/about-you/intro:
  heading: "Let's get started with some information about you"
  body:
    - "You'll need to know:"
    - ol:
      - li:
        - "<strong>Your Social Security number</strong>"
      - li:
        - "<strong>Your date of birth</strong>"
```

Content is organized by type within each screen: headings, info displays, fields, data views, etc.

### Fact Interpolation
Translation strings can reference fact values using double curly brackets:
```yaml
heading: "Your tax refund is {{/totalRefund}}"
```

If the string starts with interpolation, it must be wrapped in double quotes:
```yaml
heading: "{{/secondaryFiler/firstName}}'s information"
```

### Content Review and Translation Workflow
1. Content is written in English by the content team
2. English content is submitted to IRS General Counsel (GC) for legal review
3. After GC approval, content is sent to the translation team
4. An export script (`npm run export-locales`) generates Excel workbooks organized by flow category
5. Translators work in the Excel files
6. An import script (`npm run import-locales`) merges translations back into `es.yaml`
7. Translation references (`$t(path./to-value)`) allow reuse of common strings

### Pseudolocalization
For development and testing, a pseudo-locale option (`VITE_USE_PSEUDO_LOCALE=true`) replaces Spanish with a pseudolocalized version of English. This helps identify:
- Untranslated strings (they appear in normal English)
- Layout issues with longer translated text
- Hardcoded strings that bypassed the i18n system

## Accessibility (a11y)

### Target: WCAG 2.2 AA
DirectFile targets WCAG 2.2 AA compliance, exceeding the baseline Section 508 requirements. This was a deliberate decision to be "forward-thinking" and set the application up with the latest accessibility guidance.

### Implementation
- **USWDS components**: The US Web Design System (via `@trussworks/react-uswds`) provides pre-built accessible components
- **Semantic HTML**: Proper heading hierarchy, ARIA labels, form associations
- **Keyboard navigation**: All functionality accessible via keyboard
- **Screen reader support**: Tested with NVDA (Windows) and VoiceOver (macOS)
- **Focus management**: Proper focus handling during page transitions
- **Error announcements**: Form errors are announced to screen readers
- **Color contrast**: Meets WCAG 2.2 AA contrast requirements, with some APCA (WCAG 3.0 draft) checks

### Content Security Policy (CSP)
The application implements a Content Security Policy to prevent XSS and other injection attacks. The CSP configuration is maintained in the application and must be updated when new external resources are added.

## The All-Screens Development Tool

A special development page at `/df/file/all-screens/index.html` displays every screen in the application along with its conditions. This is invaluable for:
- Content review (seeing all screens without clicking through the flow)
- Verifying screen conditions
- Checking that all screens are properly configured
- Design review
