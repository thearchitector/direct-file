# Testing Strategy

## The Testing Problem in Tax Software

Testing a tax filing application is uniquely challenging because:
1. Tax law is complex with many interacting rules — a change in one area can affect calculations in completely unrelated areas
2. Correctness is critical — a wrong number on a tax return has legal and financial consequences
3. The combinatorial space is enormous — filing statuses × income types × credit eligibility × deduction scenarios × family configurations = millions of possible combinations
4. Some bugs only manifest in specific combinations (e.g., Married Filing Separately + Social Security income + lived apart all year)

DirectFile's testing strategy addresses this through four complementary testing methods, all running automatically on every pull request.

## The Four Testing Methods

### 1. Completeness Testing — "Did we collect all the data we need?"

**Problem**: After a user completes a section of the interview, have all the necessary facts been set? If the dependent section doesn't fully resolve whether a child qualifies for the Child Tax Credit, the credit calculation will be incomplete (and possibly wrong or blocking).

**Approach**: `flowSetsFacts` tests simulate a user clicking through all possible paths in the flow and verify that certain "culminating facts" are always complete by the end of each section.

**How it works**:
1. Start with a fact graph representing a specific scenario
2. Simulate traversing every possible path through a section of the flow
3. For every path, set every possible combination of answers
4. After each complete traversal, check that the required exported facts are complete

**Location**: `df-client/df-client-app/src/test/completenessTests/`

**Strengths**:
- Low margin for manual error (the simulator finds paths automatically)
- Catches the most common category of bugs (incomplete facts bubbling through to downstream calculations)

**Weaknesses**:
- Computationally expensive (NP-hard in the worst case)
- Can't yet handle the largest sections (spouse, dependents) due to combinatorial explosion

**Historical bugs caught by completeness testing**:
- When filing as Married Filing Separately, the system never collected `secondaryFiler/canBeClaimed`, which left `/filersCouldntBeDependents` incomplete, which broke the deductions section
- When filing as married and filers didn't live apart for at least six months, the Social Security section broke because `/livedApartAllYear` was incomplete

### 2. Correctness Testing — "Are our calculations right?"

**Problem**: Given a complete set of inputs, does the fact graph compute the correct derived values?

**Approach**: Unit tests that create a fact graph with specific writable fact values, then assert that derived facts have the expected values.

**How it works**:
1. Create a fact graph instance
2. Set specific writable facts to known values
3. Let the graph derive all computed facts
4. Assert that specific derived facts equal expected values

**Location**: `df-client/df-client-app/src/test/factDictionaryTests/`

**Test format**: The team considered (and partially implemented) spreadsheet-based tests where:
- One sheet contains "given facts" (writable paths and values)
- Another sheet contains "expected facts" (derived paths and expected values)
- A test harness loads the given facts and verifies the expected facts

This format was chosen because:
- Spreadsheets can be reviewed by non-programmer tax experts
- Test scenarios can be based on real IRS test cases (e.g., VITA Volunteer Assistors test, Pub 6744)
- Scenarios can inherit from each other (test variations by overriding specific facts)

**Strengths**:
- Can test specific edge cases and boundary conditions
- Fast to run
- Can be verified by tax subject matter experts

**Weaknesses**:
- Requires a human to identify which scenarios to test (subjective and difficult)
- Requires the test author to know the correct answer (do taxes correctly)

**Example tests**:
- After a certain income amount, the student loan adjustment phases out
- A taxpayer over 65 receives an additional standard deduction amount
- A child without an SSN can qualify a taxpayer for EITC but doesn't count in the credit computation

### 3. Functional Flow Navigation Testing — "Does the user see the right screens?"

**Problem**: After each screen, the system must use the current fact graph state to choose the correct next screen. If navigation is wrong, the user might skip critical questions or see irrelevant ones.

**Approach**: For each screen, test that given a starting fact graph state and the user's answer, the system navigates to the correct next screen.

**How it works**:
1. Set up a fact graph in a specific state
2. Set the current screen
3. Set a fact value (simulating user input)
4. Call the navigation logic
5. Assert the next screen is correct

**Location**: `df-client/df-client-app/src/test/functionalFlowTests/`

**Strengths**:
- Directly tests the user's experience
- Catches when screen reordering or new screens break navigation

**Weaknesses**:
- Prone to manual error (developer must reason about all possible next screens)
- Becomes maintenance-heavy as screens are added/reordered

**Example tests**:
- A US citizen all year → asked about state residency. Not a citizen all year → asked about year-end citizenship status.
- Single, in-scope state → proceed to TIN entry. Multiple states → knockout screen.

### 4. Flow Snapshot Testing — "Did the screen order change unexpectedly?"

**Problem**: Changes to the flow or fact dictionary might accidentally change the screen sequence for existing scenarios.

**Approach**: For each predefined scenario, run through the entire flow (simulating a user clicking through all screens) and store the sequence of screens seen. On subsequent runs, compare the new sequence against the stored snapshot.

**How it works**:
1. Load a scenario (a JSON file representing a complete fact graph)
2. Simulate clicking through every screen from start to finish
3. Record the ordered sequence of screen routes visited
4. Compare against the previously stored snapshot
5. If different, the test fails — developer must verify the change is intentional and update the snapshot

**Strengths**:
- Catches unintended side effects from any code change
- No manual test authoring needed (the snapshot is auto-generated)
- Covers the full flow, not just individual transitions

**Weaknesses**:
- When a legitimate change affects many scenarios, many snapshots need updating (can produce "100 errors")
- Doesn't verify correctness — only that the sequence hasn't changed

### 5. Static Analysis

**Approach**: TypeScript type safety and custom build-time checks ensure structural correctness:

**What it checks**:
- Fact paths referenced in the flow actually exist in the fact dictionary
- Writable facts are written with the correct type
- Derived facts only depend on facts in their own module or exported facts from other modules
- Facts used by MeF are exported with `mef="true"`
- Translation keys referenced in the flow exist in the YAML files

**Strengths**:
- Zero runtime overhead
- Catches entire categories of bugs at compile time
- Full codebase coverage (not sampling-based)

## Scenarios: The Test Data Foundation

**What are scenarios?**
Scenarios are JSON files representing complete fact graphs — each one represents a specific taxpayer's situation. They are stored as JSON and used as input to multiple test types.

**How they're used**:
- Flow snapshot tests: Run each scenario through the full flow
- Completeness tests: Start from a scenario and explore all paths
- Correctness tests: Use scenarios as starting points for specific assertions

**Characteristics**:
- A scenario must be "complete" — representing a return that could be submitted (all required facts set)
- There are several hundred predefined scenarios covering different combinations of filing statuses, income types, credits, etc.
- Scenarios are version-controlled and updated when the fact dictionary changes

**migrateScenarios script**: When a code change adds new facts or modifies existing ones, the `npm run migrate-scenarios` script batch-updates all scenario JSON files. The developer modifies the migration script to apply the correct updates, then runs it. This prevents manually editing hundreds of files.

## Backend Testing

### Java/Spring Tests
- **Unit tests**: `./mvnw test` runs the full test suite for any Spring Boot project
- **Framework**: JUnit (standard Java testing)
- **Coverage**: Jacoco Maven plugin generates coverage reports at `<app>/target/site/jacoco/index.html`
- **Static analysis**: SpotBugs (compiled code analysis) and PMD (source code analysis) run as pre-commit hooks

### Manual Testing with Subject Matter Experts
Beyond automated tests, the IRS testing team performs:
- **Scenario testing**: Integration tests that apply scenarios end-to-end and verify the tax burden matches expected values (black box pass/fail)
- **Manual testing**: Tax experts use the application to find issues that automated tests can't catch — particularly errors in the team's understanding of tax law

## Test Commands Summary

| Command | What It Tests | Location |
|---------|--------------|----------|
| `npm run test` | All frontend vitest tests | df-client-app |
| `npm run test:ci` | CI tests except completeness and functional flow | df-client-app |
| `npm run test:ci:2` | Completeness and functional flow tests | df-client-app |
| `npm run test factDictionaryTests` | Fact dictionary correctness tests | df-client-app |
| `./mvnw test` | Backend Java tests | backend/ |
| `./mvnw spotbugs:check` | Java static analysis (SpotBugs) | backend/ |
| `./mvnw pmd:check` | Java static analysis (PMD) | backend/ |
| `sbt test` | Scala fact graph unit tests | fact-graph-scala/ |

## Key Lessons for a Replacement System

1. **Invest in completeness testing early**: The most common and damaging bugs in DirectFile were incompleteness bugs where a missing fact cascaded through to break unrelated calculations.

2. **Module boundaries enable tractable testing**: Without modules, you'd need to test "everything, everywhere, all of the time." With modules, you can test each module independently and then test the ~140 exported facts at module boundaries.

3. **Scenarios are invaluable**: Having hundreds of predefined complete tax situations as test data enables rapid regression testing and makes it easy to verify changes.

4. **Tax experts must be in the loop**: Automated tests verify that the system behaves as coded. Only human tax experts can verify that the coding matches the tax law.

5. **Snapshot tests catch surprises**: Even when you think a change is isolated, snapshot tests reveal unexpected ripple effects across the entire flow.
