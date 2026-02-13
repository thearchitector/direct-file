# PDF Generation System

## Overview

DirectFile generates PDF renderings of completed tax returns for user review, record-keeping, and the print-and-mail option. The PDF system maps fact graph values to fields in PDF templates that replicate official IRS paper forms.

## Two Types of PDF Templates

### 1. Form Templates
PDFs that replicate official IRS paper forms (e.g., Form 1040, Schedule 1, W-2 summary). These are:
- Downloaded directly from irs.gov
- Pre-processed with Adobe Acrobat Pro to remove instructional pages
- Configured with YAML mappings from PDF fields to fact graph paths
- Must pass Acrobat Pro's Accessibility Checks

### 2. Table Templates
PDFs designed by the DirectFile team for listing additional information (e.g., multiple W-2s, multiple 1099s). These are:
- Created by DirectFile designers in Adobe Acrobat
- Support pagination (automatic page breaks when rows exceed `rowsPerPage`)
- Include per-page headers (e.g., filer name and SSN on each page)
- Custom-designed for accessibility

## PDF Configuration Format

Each PDF template has a `configuration.yml` file that maps PDF form fields to fact graph paths. The configuration lives in:
`backend/src/main/resources/pdf/{taxYear}/{FormName}/{languageCode}/configuration.yml`

### Include Conditions

Every configuration specifies when the template should be included in the tax return PDF:

```yaml
# Include when a boolean fact is true
includeWhen: /shouldIncludeSchedule1

# OR include once per collection item
includeForEach: /formW2s
```

### Form Field Mappings

For form-type templates, the `form:` key maps PDF field names to fact expressions:

```yaml
form:
  form1[0]:
    Page1[0]:
      f1_01[0]: /filers/*/firstName    # First name field
      f1_02[0]: /filers/*/lastName     # Last name field
      f1_03[0]: /filers/*/ssn          # SSN field
      f1_04[0]: /adjustedGrossIncome   # AGI
```

### Table Configuration

For table-type templates, the `table:` key configures row/column mappings:

```yaml
table:
  rowCollectionPath: /formW2s
  itemsToSkip: 0
  rowsPerPage: 3
  columns:
    - factExpression: ../employerName
      fieldName: employer
    - factExpression: ../wages
      fieldName: wages
    - factExpression: ../federalTaxWithheld
      fieldName: withheld
  oncePerPage:
    - factExpression: /filers/*/fullName
      fieldName: filerName
    - factExpression: /filers/*/ssn
      fieldName: filerSsn
```

## The PdfToYaml Utility

A utility tool at `direct-file/utils/pdf-to-yaml/` generates the initial YAML configuration from a PDF template:
- `--pdf-fields` output: Shows which internal field name corresponds to which line/input of the PDF
- `--form-template` output: Generates the starting point for the `configuration.yml`'s `form:` key
- Useful for diffing PDF versions when IRS updates forms year-to-year

## Pseudo-Facts

Some PDF fields need values that aren't directly stored as fact graph facts. "Pseudo-facts" are computed by the PDF generation code at render time. For example:
- Concatenating first name + last name into a full name field
- Formatting a date in a specific way for a PDF field
- Computing a summary value specific to PDF display

When a template needs pseudo-facts, a custom Java subclass of `PdfForm` is created with a `computePseudoFacts()` method.

## Multi-Language Support

PDFs are generated in both English and Spanish:
- Each template has separate `en/` and `es/` configuration directories
- The PDF templates themselves may differ (Spanish versions of IRS forms)
- The configuration YAML may differ (field names can change between language versions)

## Application Registration

Each PDF template must be registered in `application.yaml`:

```yaml
pdfs:
  configured-pdfs:
    - name: IRS1040
      year: 2024
      language-code: en
      location: pdf/2024/IRS1040/en/f1040_2024.pdf
      location-type: classpath
      configuration-location: pdf/2024/IRS1040/en/configuration.yml
      configuration-location-type: classpath
      cache-in-memory: true
```

## PDF Templates in the Codebase (Tax Year 2024)

The `backend/src/main/resources/pdf/2024/` directory contains 76 items covering:
- Form 1040 (main return)
- All supported schedules (1, 2, 3, 8812, B, R, EIC)
- All supported additional forms (2441, 8880, 8889, 8962, 8862, 9000)
- W-2 summaries
- 1099-R summaries
- Both English and Spanish versions
- Both form and table types

## Tax Year Transition Challenges

When the IRS updates forms for a new tax year:
1. New PDF files must be downloaded from irs.gov
2. Field names may change between years (requiring configuration updates)
3. 508-compliant versions often aren't available until after filing season starts
4. The team uses draft releases for early implementation, then updates when final/accessible versions are posted
5. The PdfToYaml utility's diff mode helps identify which fields changed

## Key Architectural Insight

The PDF generation system is notable for its configuration-driven approach:
- The fact-to-field mapping is in YAML, not code
- Adding a new form primarily involves creating a YAML configuration, not writing Java
- The same fact graph that drives the UI and XML also drives PDF generation
- The `includeWhen`/`includeForEach` conditions use the same fact graph paths as the flow conditions

This means the fact graph is truly the universal interface between all parts of the system: UI, computation, XML generation, AND PDF generation.

## Implications for a Replacement System

1. **Use a configuration-driven PDF approach**: Map fact paths to PDF field coordinates in data files, not code
2. **Plan for annual PDF updates**: The IRS changes forms every year; the update process should be streamlined
3. **Support multi-language PDFs from the start**: Separate configurations per language
4. **Consider accessibility**: PDF output must meet 508/WCAG standards
5. **Use a PDF library that supports form filling**: Python options include `reportlab`, `pdfrw`, or `PyPDF2` for filling existing PDF form templates
