# Executive Summary: IRS DirectFile

## What Is DirectFile?

DirectFile is a web application built by the IRS that allows US taxpayers to electronically file their federal tax return (Form 1040) for free, directly with the Internal Revenue Service. It was developed by an in-house team of IRS technologists with support from USDS, GSA, and vendor teams (TrussWorks, Coforma, ATI).

## Core Mission

DirectFile translates the US Internal Revenue Code (26 USC) into **plain-language questions** whose answers should be known to taxpayers without needing external instructions or publications. The taxpayer's answers are then automatically translated into standard IRS tax forms and transmitted electronically to the IRS's Modernized e-File (MeF) system.

## Key Product Characteristics

1. **Interview-based**: Rather than presenting blank tax forms, DirectFile walks the user through a series of questions organized into logical categories (about you, income, deductions, credits, etc.). The application determines which questions to ask based on previous answers.

2. **Mobile-first**: Designed to work equally well on phones, tablets, and desktops.

3. **Bilingual**: Available in English and Spanish, using the i18next internationalization framework with YAML-based translation files.

4. **Accessible**: Targets WCAG 2.2 AA compliance, exceeding the baseline Section 508 requirements. Designed for users with diverse abilities and access needs.

5. **Free and direct**: No third-party intermediary. The return goes straight from the taxpayer to the IRS.

## What Tax Situations Does It Support?

DirectFile supports a subset of tax situations that covers a large number of US taxpayers. Based on the fact dictionary modules found in the codebase, it supports:

### Income Types
- **W-2 wage income** (employment wages, tips, other compensation)
- **1099-INT interest income** (bank interest, etc.)
- **1099-G unemployment compensation**
- **SSA-1099 Social Security benefits**
- **1099-R retirement distributions** (pensions, annuities, IRAs)
- **1099-MISC miscellaneous income**
- **Alaska Permanent Fund dividends**
- **Health Savings Account (HSA) distributions** (Form 8889)
- **Dependent care benefits**

### Filing Statuses
- Single
- Married Filing Jointly (MFJ)
- Married Filing Separately (MFS)
- Head of Household (HoH)
- Qualifying Surviving Spouse (QSS)

### Credits
- **Earned Income Tax Credit (EITC)** — the largest and most complex credit
- **Child Tax Credit (CTC) / Other Dependent Credit (ODC)**
- **Child and Dependent Care Credit (CDCC)** (Form 2441)
- **Credit for the Elderly or Disabled** (Schedule R)
- **Saver's Credit** (Form 8880)
- **Premium Tax Credit (PTC)** (Form 8962)
- **Educator Expense Adjustment**
- **Student Loan Interest Deduction**

### Deductions
- **Standard Deduction** (including additional amounts for age 65+ and blind)
- DirectFile does NOT support itemized deductions (Schedule A)

### Tax Computation
- Regular income tax calculation using tax tables/brackets
- Self-employment tax (limited, via Schedule 2)
- Estimated tax payments

### What It Does NOT Support (Knockouts)
If a taxpayer's situation falls outside the supported scope, they are "knocked out" — shown a message explaining that DirectFile cannot handle their return, and directed to other filing options. Common knockouts include:
- Itemized deductions
- Self-employment income (Schedule C)
- Capital gains/losses (Schedule D)
- Rental income (Schedule E)
- Business income (K-1s)
- Multiple state residency
- Non-resident aliens
- Various complex tax situations

## How the System Works — High-Level Flow

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐     ┌─────────────┐
│  Screener   │────▶│  Interview   │────▶│   Review &   │────▶│  Submit to  │
│ (Eligibility│     │  (Questions) │     │    Sign      │     │    MeF      │
│  Check)     │     │              │     │              │     │             │
└─────────────┘     └──────────────┘     └──────────────┘     └─────────────┘
                           │                                        │
                    ┌──────▼──────┐                          ┌──────▼──────┐
                    │ Fact Graph  │                          │  XML/PDF    │
                    │ (Tax Logic  │                          │ Generation  │
                    │  Engine)    │                          │             │
                    └─────────────┘                          └─────────────┘
```

1. **Screener**: A set of static pages (built with Astro SSG) that ask preliminary eligibility questions before the user logs in. If any answers indicate the user doesn't qualify, they are redirected away.

2. **Authentication**: Users authenticate through an external Credential Service Provider (CSP) that performs identity verification. DirectFile does not handle authentication directly — it integrates with an external identity provider.

3. **Interview**: The React single-page application walks the user through questions organized into categories:
   - You and Your Family (personal info, spouse, dependents, filing status)
   - Income (W-2s, 1099s, Social Security, etc.)
   - Credits and Deductions
   - Your Taxes (estimated payments, tax amount, payment/refund method)
   - Sign and Submit

4. **Fact Graph Processing**: Every answer the user provides is written into the Fact Graph. The graph automatically computes all derived facts (tax calculations, eligibility determinations, etc.) in real-time.

5. **Review and Sign**: The user reviews a summary of their return, enters their IP PIN (if applicable), and signs electronically.

6. **Submission**: The backend converts the fact graph data into IRS MeF XML format, validates it against the MeF schema, and submits it electronically. The system then polls for acceptance/rejection.

7. **State Tax Handoff**: After federal filing, the user can optionally authorize transfer of their federal return data to a state tax filing tool via the State API.

## Technology Stack Summary

| Layer | Technology | Why |
|-------|-----------|-----|
| Frontend | React 18 + TypeScript | SPA with USWDS design system, mobile-first |
| Tax Logic Engine | Scala (transpiled to JS via Scala.js) | Runs on both JVM backend and JS frontend |
| Backend API | Java 21 + Spring Boot | REST API, handles auth, data persistence, MeF integration |
| Database | PostgreSQL 15 | Stores user data, tax returns, submissions |
| Message Queue | AWS SQS (LocalStack for dev) | Async communication between services |
| Object Storage | AWS S3 (LocalStack for dev) | Filed tax return artifacts (PDFs, XML) |
| Encryption | AWS KMS | Envelope encryption for taxpayer data at rest |
| Cache | Redis | Session data, status caching |
| Monitoring | OpenTelemetry + Prometheus + Grafana | Observability |
| Containerization | Docker Compose | Local development and deployment |

## Key Architectural Insight

The single most important design decision in DirectFile is that **the Fact Graph is the single source of truth for all tax-related state**. It is:
- The data model (stores all user-entered and computed data)
- The computation engine (derives tax calculations from user inputs)
- The flow controller (determines which screens to show)
- The validation engine (determines what data is complete/valid)
- The form generator input (feeds into PDF and XML generation)

This means any replacement system must have an equivalent central knowledge representation for tax logic that can serve all these roles simultaneously.
