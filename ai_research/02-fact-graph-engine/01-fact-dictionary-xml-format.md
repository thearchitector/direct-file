# Fact Dictionary XML Format

## Overview

The Fact Dictionary is defined across multiple XML files in `direct-file/backend/src/main/resources/tax/`. Each file represents a module and follows a schema defined in `FactDictionaryModule.rng` (a RELAX NG schema).

This document explains the XML format used to define facts, their types, their computation logic, and their relationships.

## Basic Structure

```xml
<FactDictionary>
  <!-- Module-level metadata -->

  <Fact path="/somePath">
    <Name>Human-readable name</Name>
    <Description>Explanation of what this fact represents</Description>

    <!-- Either Writable OR Derived -->
    <Writable>
      <TypeDefinition />
    </Writable>

    <!-- OR -->

    <Derived>
      <ComputationExpression />
    </Derived>

    <!-- Optional export declaration -->
    <Export mef="true" downstreamFacts="true" />

    <!-- Optional placeholder (default value before user input) -->
    <Placeholder>
      <TypeDefinition>value</TypeDefinition>
    </Placeholder>
  </Fact>

  <!-- More facts... -->
</FactDictionary>
```

## Fact Paths

Every fact has a unique path that serves as its identifier. Paths use a hierarchical, filesystem-like syntax:

- **Simple paths**: `/filingStatus`, `/taxYear`, `/adjustedGrossIncome`
- **Collection paths**: `/formW2s/*/wages` — the `*` indicates this fact exists for each item in a collection
- **Filer paths**: `/filers/*/firstName` — exists for each filer (primary and optional secondary)
- **Relative paths**: `../wages` — references a sibling fact within the same collection item

## Writable Fact Types

Writable facts define what data the user can enter. Each has a type with optional constraints:

### Boolean
```xml
<Writable>
  <Boolean />
</Writable>
```

### String
```xml
<Writable>
  <String>
    <MinLength>1</MinLength>
    <MaxLength>50</MaxLength>
    <AllowedChars>letters,spaces</AllowedChars>
  </String>
</Writable>
```

### Dollar (monetary amounts)
```xml
<Writable>
  <Dollar />
</Writable>
```
Dollar facts represent monetary values. By default, a fact with value zero is NOT exported to PDF/XML generation. The `<ExportZero/>` tag overrides this behavior.

### Integer
```xml
<Writable>
  <Integer>
    <Min>0</Min>
    <Max>99</Max>
  </Integer>
</Writable>
```

### Date
```xml
<Writable>
  <Date>
    <Min>2024-01-01</Min>
    <Max>2024-12-31</Max>
  </Date>
</Writable>
```

### Enum (single selection)
```xml
<Writable>
  <Enum>
    <Value>single</Value>
    <Value>marriedFilingJointly</Value>
    <Value>marriedFilingSeparately</Value>
    <Value>headOfHousehold</Value>
    <Value>qualifyingSurvivingSpouse</Value>
  </Enum>
</Writable>
```

### MultiEnum (multiple selections — checkboxes)
```xml
<Writable>
  <MultiEnum>
    <Value>optionA</Value>
    <Value>optionB</Value>
    <Value>optionC</Value>
  </MultiEnum>
</Writable>
```

### Specialized Types
- **Tin** (Taxpayer Identification Number / SSN)
- **Ein** (Employer Identification Number)
- **Pin** (IP PIN for identity verification)
- **E164** (phone numbers in international format)

## Derived Fact Computation Nodes

Derived facts use a declarative expression language defined as XML nodes. These are the "formulas" of the spreadsheet analogy.

### Dependency — Reference Another Fact
```xml
<Dependency path="/someOtherFact" />
<Dependency module="filers" path="/livedApartAllYear" />  <!-- Cross-module reference -->
<Dependency path="../wages" />  <!-- Relative path within same collection -->
```

### Arithmetic Operations
```xml
<Add>
  <Dependency path="/factA" />
  <Dependency path="/factB" />
</Add>

<Subtract>
  <Dependency path="/factA" />
  <Dependency path="/factB" />
</Subtract>

<Multiply>
  <Dependency path="/factA" />
  <Dependency path="/factB" />
</Multiply>

<Divide>
  <Dependency path="/factA" />
  <Dependency path="/factB" />
</Divide>

<RoundDown>
  <Dependency path="/someDecimal" />
</RoundDown>

<Max>
  <Dependency path="/factA" />
  <Dependency path="/factB" />
</Max>

<Min>
  <Dependency path="/factA" />
  <Dependency path="/factB" />
</Min>
```

### Boolean Logic
```xml
<All>  <!-- AND -->
  <Dependency path="/conditionA" />
  <Dependency path="/conditionB" />
</All>

<Any>  <!-- OR -->
  <Dependency path="/conditionA" />
  <Dependency path="/conditionB" />
</Any>

<Not>
  <Dependency path="/conditionA" />
</Not>

<True />   <!-- Constant true -->
<False />  <!-- Constant false -->
```

### Comparison Operations
```xml
<GreaterThan>
  <Dependency path="/factA" />
  <Dollar>50000</Dollar>
</GreaterThan>

<LessThanOrEqual>
  <Dependency path="/factA" />
  <Dependency path="/factB" />
</LessThanOrEqual>

<Equals>
  <Dependency path="/filingStatus" />
  <Enum>marriedFilingJointly</Enum>
</Equals>
```

### Conditional Logic (Switch/Case)
The Switch/Case construct is the primary way to express conditional logic:

```xml
<Switch>
  <Case>
    <When>
      <Equals>
        <Dependency path="/filingStatus" />
        <Enum>single</Enum>
      </Equals>
    </When>
    <Then>
      <Dollar>13850</Dollar>
    </Then>
  </Case>
  <Case>
    <When>
      <Equals>
        <Dependency path="/filingStatus" />
        <Enum>marriedFilingJointly</Enum>
      </Equals>
    </When>
    <Then>
      <Dollar>27700</Dollar>
    </Then>
  </Case>
  <Case>
    <When>
      <True />  <!-- Default case -->
    </When>
    <Then>
      <Dollar>0</Dollar>
    </Then>
  </Case>
</Switch>
```

### Completeness Checks
```xml
<IsComplete>
  <Dependency path="/someOptionalFact" />
</IsComplete>
```
Returns true if the referenced fact has a complete (non-incomplete) value. This is essential for the optional facts pattern.

### Collection Operations
```xml
<!-- Sum all values in a collection -->
<SumAll>
  <Dependency path="/formW2s/*/wages" />
</SumAll>

<!-- Count items in a collection -->
<CountAll>
  <Dependency path="/familyAndHousehold/*/isDependent" />
</CountAll>

<!-- Filter a collection -->
<Filter>
  <Collection path="/familyAndHousehold" />
  <Predicate>
    <Dependency path="*/isQualifyingChild" />
  </Predicate>
</Filter>
```

### String Operations
```xml
<AsString>
  <Dependency path="/someEnum" />
</AsString>

<ToUpper>
  <AsString>
    <Dependency path="/someValue" />
  </AsString>
</ToUpper>
```

### Constant Values
```xml
<Dollar>0</Dollar>
<Integer>2024</Integer>
<String>some text</String>
<Boolean>true</Boolean>
<Date>2024-12-31</Date>
```

## Export Declarations

Facts can be exported for use in other contexts:

```xml
<Export mef="true" downstreamFacts="true" />
```

- **`downstreamFacts="true"`**: Makes the fact available to other modules. Without this, a fact from one module cannot be referenced by facts in other modules.
- **`mef="true"`**: Marks the fact as available for MeF XML generation. Facts used in the XML output must have this flag.

## The ExportZero Tag

By default, facts with a value of zero are NOT included in PDF or XML output (they appear as blank). This is correct behavior for most tax form fields — if a line is zero, it should be left blank.

However, some fields must explicitly show zero. The `<ExportZero/>` tag overrides the default:

```xml
<Fact path="/someAmount">
  <Derived>
    <Dollar>0</Dollar>
  </Derived>
  <ExportZero/>  <!-- This zero WILL appear in outputs -->
</Fact>
```

A more nuanced pattern uses conditional incompleteness to achieve "sometimes export zero":
- If a fact value is deliberately left incomplete (not just zero), PDF generation ignores it
- XML mappings can use Optional syntax for similar behavior

## Placeholder Values

Placeholders provide an initial value for writable facts, but do NOT make the fact "complete":

```xml
<Fact path="/formW2s/*/wages">
  <Writable>
    <Dollar />
  </Writable>
  <Placeholder>
    <Dollar>0</Dollar>
  </Placeholder>
</Fact>
```

This means:
- The fact has a value of 0 from the start (it's never truly blank)
- But the fact is NOT considered complete until the user actively sets it
- This is important because DirectFile generally doesn't provide ways to "clear" a written fact — once set, it stays set

## The Optional Facts Pattern

DirectFile handles optional fields (like W-2 Box 12 codes or middle initials) using a specific pattern:

1. Create a writable fact prefixed with `writable` (e.g., `/formW2s/*/writableWages`)
2. Mark it as `required='false'` in the flow
3. Create a derived fact with the original name that provides a default when the writable is incomplete:

```xml
<Fact path="/formW2s/*/writableWages">
  <Writable><Dollar /></Writable>
</Fact>

<Fact path="/formW2s/*/wages">
  <Derived>
    <Switch>
      <Case>
        <When><IsComplete><Dependency path="../writableWages" /></IsComplete></When>
        <Then><Dependency path="../writableWages" /></Then>
      </Case>
      <Case>
        <When><True /></When>
        <Then><Dollar>0</Dollar></Then>
      </Case>
    </Switch>
  </Derived>
</Fact>
```

This pattern was chosen over modifying the Scala fact graph engine to natively support optionality, because it:
- Doesn't complicate the core engine with null/default concepts
- Keeps writable values separate from downstream computation values
- Is easier to understand and test

## Complete Real-World Example

Here's a simplified version of how the standard deduction might be defined:

```xml
<!-- In standardDeduction.xml -->

<Fact path="/baseStandardDeduction">
  <Name>Base Standard Deduction</Name>
  <Description>The base standard deduction amount before additional amounts for age/blindness</Description>
  <Export downstreamFacts="true" mef="true" />

  <Derived>
    <Switch>
      <Case>
        <When>
          <Equals>
            <Dependency module="filingStatus" path="/filingStatus" />
            <Enum>single</Enum>
          </Equals>
        </When>
        <Then><Dollar>14600</Dollar></Then>
      </Case>
      <Case>
        <When>
          <Equals>
            <Dependency module="filingStatus" path="/filingStatus" />
            <Enum>marriedFilingJointly</Enum>
          </Equals>
        </When>
        <Then><Dollar>29200</Dollar></Then>
      </Case>
      <!-- More cases for other filing statuses -->
    </Switch>
  </Derived>
</Fact>
```

## Schema Validation

The XML format is validated against a RELAX NG schema (`FactDictionaryModule.rng`) that enforces:
- Valid fact paths
- Correct nesting of computation nodes
- Type consistency
- Module export rules

Additional build-time checks verify:
- Cross-module dependencies only reference exported facts
- Facts used by MeF are marked with `mef="true"`
- No circular dependencies that can't be resolved
