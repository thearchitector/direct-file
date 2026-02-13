# Tax Forms and Concepts

## Purpose of This Document

This document explains the core tax concepts that DirectFile implements. A developer or AI agent building a replacement system needs to understand these concepts to make correct design decisions. This is NOT a comprehensive tax guide — it covers only what DirectFile supports.

## The Form 1040: The Central Tax Return

The IRS Form 1040 ("U.S. Individual Income Tax Return") is the primary federal tax form that virtually all individual taxpayers must file. DirectFile's entire purpose is to help users complete and file this form.

### What Form 1040 Contains

The 1040 has approximately 38 lines that compute the taxpayer's tax liability or refund. The high-level flow is:

```
1. Total Income (Lines 1-9)
   - Wages (W-2s)
   - Interest income
   - Unemployment compensation
   - Social Security benefits (taxable portion)
   - Other income (Alaska PFD, 1099-R distributions, etc.)

2. Adjusted Gross Income / AGI (Line 11)
   = Total Income - Adjustments to Income
   Adjustments include: educator expenses, student loan interest, HSA deduction

3. Taxable Income (Line 15)
   = AGI - Standard Deduction (or Itemized Deductions)
   DirectFile only supports Standard Deduction

4. Tax (Line 16)
   Computed from tax tables/brackets based on taxable income and filing status

5. Credits (Lines 19-21)
   Reduce tax dollar-for-dollar: Child Tax Credit, EITC, etc.

6. Other Taxes (Line 23)
   Self-employment tax, etc. (limited in DirectFile)

7. Total Tax (Line 24)

8. Payments (Lines 25-33)
   Federal tax withheld (from W-2s, 1099s), estimated payments, credits

9. Refund or Amount Owed (Lines 34-37)
   = Payments - Total Tax
   Positive = refund; Negative = amount owed
```

### Supporting Schedules DirectFile Generates

In addition to the main 1040, DirectFile generates these supporting forms when applicable:

| Form/Schedule | Purpose | When Generated |
|--------------|---------|----------------|
| Schedule 1 | Additional Income and Adjustments | Unemployment, Alaska PFD, educator expense, student loan interest |
| Schedule 2 | Additional Taxes | Self-employment tax (limited) |
| Schedule 3 | Additional Credits and Payments | Saver's Credit, estimated tax payments |
| Schedule 8812 | Credits for Qualifying Children | Child Tax Credit / Other Dependent Credit |
| Schedule B | Interest and Ordinary Dividends | When interest income exceeds $1,500 |
| Schedule R | Credit for the Elderly or Disabled | When claiming this credit |
| Form 8889 | Health Savings Accounts | When reporting HSA transactions |
| Form 8962 | Premium Tax Credit | When claiming PTC |
| Form 2441 | Child and Dependent Care Expenses | When claiming CDCC |
| Form 8880 | Credit for Qualified Retirement Savings | When claiming Saver's Credit |
| Schedule EIC | Earned Income Credit | When claiming EITC with qualifying children |

## Filing Status

Filing status is one of the most critical determinations in tax filing. It affects tax brackets, standard deduction amount, credit eligibility, and many other calculations. DirectFile supports all five filing statuses:

### 1. Single
- Unmarried, or legally separated/divorced on December 31 of the tax year
- Not qualifying for any other status

### 2. Married Filing Jointly (MFJ)
- Married on December 31 of the tax year
- Both spouses agree to file together
- Both spouses' income is reported on one return
- Generally results in the lowest combined tax

### 3. Married Filing Separately (MFS)
- Married but choosing to file separate returns
- Each spouse reports only their own income
- May result in higher combined tax but limits liability
- Some credits are reduced or unavailable

### 4. Head of Household (HoH)
- Unmarried (or "considered unmarried") on December 31
- Paid more than half the cost of maintaining a home
- Had a qualifying person living with them for more than half the year
- Better tax brackets than Single, larger standard deduction

### 5. Qualifying Surviving Spouse (QSS)
- Spouse died in the previous or current tax year
- Was eligible to file jointly the year of spouse's death
- Has a dependent child
- Maintains a household for that child
- Gets MFJ tax brackets and standard deduction for up to 2 years after spouse's death

## Income Types

### W-2 Wage Income
The most common income type. A W-2 form is issued by each employer and contains:
- **Box 1**: Wages, tips, other compensation (the main income figure)
- **Box 2**: Federal income tax withheld
- **Box 3-6**: Social Security and Medicare wages and taxes
- **Box 12**: Various codes for special items (retirement contributions, HSA, etc.)
- **Box 13**: Checkboxes (statutory employee, retirement plan, third-party sick pay)
- **Box 14**: Other information (state/local taxes, union dues, etc.)

A taxpayer may have multiple W-2s (from different jobs or employers).

### 1099-INT Interest Income
Reports interest earned from bank accounts, CDs, bonds, etc.
- **Box 1**: Interest income
- **Box 3**: Interest on US Savings Bonds and Treasury obligations
- **Box 4**: Federal income tax withheld
- **Box 8**: Tax-exempt interest

If total interest exceeds $1,500, Schedule B is required.

### 1099-G Unemployment Compensation
Reports unemployment benefits received from state unemployment agencies.
- **Box 1**: Unemployment compensation
- **Box 4**: Federal income tax withheld
- **Box 11**: State income tax withheld

### SSA-1099 Social Security Benefits
Reports Social Security benefits received.
- **Box 3**: Benefits paid
- **Box 4**: Benefits repaid
- **Box 5**: Net benefits (Box 3 - Box 4)

The taxable portion of Social Security is computed using a complex formula that depends on "combined income" (half of SS benefits + other income + tax-exempt interest). The taxable portion ranges from 0% to 85% of benefits.

### 1099-R Retirement Distributions
Reports distributions from pensions, annuities, retirement plans, IRAs, and insurance contracts.
- **Box 1**: Gross distribution
- **Box 2a**: Taxable amount
- **Box 4**: Federal income tax withheld
- **Box 7**: Distribution code (determines tax treatment)

The tax treatment varies significantly based on the distribution code and the type of retirement account.

### 1099-MISC Miscellaneous Income
Reports miscellaneous income not covered by other forms.
- **Box 3**: Other income (prizes, awards, etc.)

### Alaska Permanent Fund Dividend
Alaska residents receive an annual dividend from the state's Permanent Fund. This is reported as other income on Schedule 1, Line 8g.

### HSA (Health Savings Account) — Form 8889
Reports contributions to and distributions from Health Savings Accounts.
- Employer contributions (from W-2 Box 12, Code W)
- Personal contributions (deductible)
- Distributions (tax-free if used for qualified medical expenses)

## Standard Deduction

The standard deduction reduces taxable income. DirectFile only supports the standard deduction (NOT itemized deductions). The amount depends on:

1. **Filing status**: Each status has a base amount (e.g., $14,600 for Single in 2024, $29,200 for MFJ)
2. **Age**: Additional amount if taxpayer or spouse is 65 or older
3. **Blindness**: Additional amount if taxpayer or spouse is blind
4. **Dependency**: Reduced amount if taxpayer can be claimed as a dependent

The additional amounts for age/blindness differ based on filing status:
- Single/HoH: $1,950 per condition
- MFJ/MFS/QSS: $1,550 per condition

## Credits

Credits directly reduce the tax owed (as opposed to deductions, which reduce taxable income). Credits are generally more valuable than deductions of the same amount.

### Earned Income Tax Credit (EITC)
The EITC is the most complex credit DirectFile supports. It is a **refundable** credit (can result in a refund even if no tax is owed) designed for low-to-moderate income workers.

Eligibility depends on:
- Earned income (wages, self-employment income)
- Adjusted Gross Income
- Filing status
- Number of qualifying children (0, 1, 2, or 3+)
- Investment income must be below a threshold
- Must have a valid SSN
- Cannot file as MFS
- Age requirements (for those without qualifying children)

The credit amount is computed using a complex formula with phase-in and phase-out ranges that vary by filing status and number of qualifying children.

### Child Tax Credit (CTC) / Other Dependent Credit (ODC)
- **CTC**: Up to $2,000 per qualifying child under age 17 with a valid SSN
- **ODC**: Up to $500 per other qualifying dependent
- Partially refundable via the Additional Child Tax Credit (ACTC)
- Phases out at higher income levels

### Child and Dependent Care Credit (CDCC) — Form 2441
Credit for expenses paid for care of qualifying individuals (children under 13, disabled dependents) to allow the taxpayer to work.
- Maximum expenses: $3,000 for one qualifying individual, $6,000 for two or more
- Credit percentage: 20% to 35% of expenses, depending on income
- Non-refundable

### Credit for the Elderly or Disabled — Schedule R
Available to taxpayers who are:
- Age 65 or older, OR
- Under 65 and permanently/totally disabled with disability income
- Below certain income thresholds

### Saver's Credit — Form 8880
Credit for eligible contributions to retirement savings plans (401(k), IRA, etc.).
- Credit rate: 10%, 20%, or 50% of contributions, depending on AGI
- Maximum contribution considered: $2,000 per person

### Premium Tax Credit (PTC) — Form 8962
For taxpayers who purchased health insurance through the Health Insurance Marketplace.
- Reconciles advance payments of the PTC received during the year
- Can result in additional credit or repayment of excess advance payments

## Adjustments to Income (Above-the-Line Deductions)

These reduce AGI and are available even to taxpayers who take the standard deduction:

### Educator Expense Adjustment
- Up to $300 per eligible educator ($600 for MFJ with two educators)
- For K-12 teachers who spent their own money on classroom supplies

### Student Loan Interest Deduction
- Up to $2,500 of interest paid on qualified student loans
- Phases out at higher income levels
- Not available for MFS filers

### HSA Deduction
- Deduction for personal contributions to a Health Savings Account
- Subject to annual contribution limits based on coverage type (self-only vs. family)

## Estimated Tax Payments

Taxpayers who expect to owe $1,000 or more may make quarterly estimated tax payments throughout the year. These payments are credited against the total tax liability.

## Withholding

Federal income tax withheld from paychecks (W-2 Box 2), unemployment (1099-G Box 4), Social Security (SSA-1099), retirement distributions (1099-R Box 4), and other sources is credited against the total tax liability.

## Refund and Payment

After computing total tax and subtracting payments/credits:
- **Refund**: If overpaid, the taxpayer can receive a refund via direct deposit or check
- **Amount owed**: If underpaid, the taxpayer must pay the balance

DirectFile supports refund delivery via:
- Direct deposit to a bank account (routing + account number)
- Paper check mailed to the taxpayer's address

For amounts owed, DirectFile supports:
- Electronic payment
- Check mailed to the IRS

## Knockouts: Unsupported Tax Situations

DirectFile explicitly does NOT support these situations and "knocks out" users who have them:

- **Itemized deductions** (Schedule A): Mortgage interest, state/local taxes, charitable contributions
- **Self-employment income** (Schedule C): Freelance, gig economy, business income
- **Capital gains/losses** (Schedule D): Stock sales, cryptocurrency, real estate
- **Rental income** (Schedule E): Landlord income
- **Farm income** (Schedule F)
- **Partnership/S-Corp income** (Schedule K-1)
- **Foreign income or accounts** (FBAR, Form 8938)
- **Alternative Minimum Tax** (Form 6251)
- **Multiple state residency** during the tax year
- **Non-resident alien** status
- **Dependents with investment income** above threshold (Form 8615)
- **Household employer** taxes (Schedule H)
- Various other complex situations

The knockout mechanism checks these conditions early in the interview flow so users aren't wasted time entering data that DirectFile can't process.
