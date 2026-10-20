# Payroll

## Concepts

| Concept | Meaning | Account |
|---|---|---|
| **Gross earnings** | Everything earned for the month | Salary expense (and component expense accounts) |
| **Commission** | Earned per service/sale, accrues as it happens | Commission expense / Commissions payable (party) |
| **Advance** | Part of a future salary paid early; recovered from the **next** payslip(s) | Staff advances (asset, party) |
| **Loan** | Separate amount lent; repaid in a fixed **installment schedule**, by payroll deduction or direct repayment | Staff loans (asset, party) |
| **Deductions** | Unpaid leave, advance recovery, loan installment, income tax, employee contributions (EOBI, PF), fines | Each credits its own account |
| **Employer contributions** | Employer's share of EOBI/social security/PF — a cost to the business, not deducted from the employee | Dr contribution expense / Cr contribution payable |
| **Net payable** | What the business owes the employee now | Salaries payable + Commissions payable − Staff advances (party) |

## Net pay formula

```
Gross       = basic + allowances (pro-rated) + commission for the month
              + other earnings (bonus, Eidi, overtime, arrears)
Deductions  = unpaid leave + advance recovery + loan installment
              + income tax withheld + employee contributions + other
Net payable = Gross − Deductions
```

Pro-rating: by calendar days employed in the month (joiners/leavers). Unpaid leave: (basic + allowances) ÷ days in month × unpaid days. Round once per line; integers only.

## Components

Keep a **catalogue** of pay components (Settings › Payroll): name, type (`earning | deduction | employer_contribution`), amount type (fixed / % of basic), taxable flag, and the account it posts to. Employees have **recurring** components (house rent, conveyance, EOBI); payslips also accept **one-off** lines (Eid bonus, overtime this month, fine). Seed: Bonus, Eidi, Overtime, Arrears, EOBI (employee), EOBI (employer), Provident fund.

EOBI (Pakistan): employee contribution is a deduction (1% of minimum wage), employer contribution is a separate cost (5%) — both credit "EOBI payable", paid monthly to EOBI. Rates change; never hard-code them — store in settings.

Salary tax: income tax is **withheld** from salary (not GST). Store a monthly withholding amount per employee (or compute from the configured annual slabs ÷ 12 if the product needs it); post Cr "Income tax withheld – salaries"; clear it with "Pay tax" (Dr liability / Cr bank).

## Lifecycle

1. **Monthly run** per branch: generate draft payslips for everyone employed in the month.
2. **Draft** is editable: unpaid leave days, one-off lines, advance recovery (default = min(outstanding advance, gross)), loan installment (default = scheduled installment; "skip this month" moves it to the end of the schedule), tax.
   - Re-propose defaults whenever the draft refreshes, **unless the user edited that field** (track `edited` flags). An advance given after the draft was generated must still be recovered.
3. **Finalise** (after month end, or early with confirmation): lock the payslip and post it, dated the last day of the month.
4. **Correct** a finalised payslip by **void + regenerate**, never by editing.
5. **Pay**: payments clear salary payable first, then commission payable. A payment cannot exceed net payable; **excess is never silently an advance** — the user must choose "Advance" explicitly (and see "Rs X more than owed will be an advance").

## Payslip posting

```
Dr Salary expense                 (salary earnings + other earning components)
Dr EOBI expense (employer)        (employer contribution)
   Cr Staff advances   (party)    (advance recovery)
   Cr Staff loans      (party)    (loan installment)
   Cr Income tax withheld          (tax)
   Cr EOBI payable                 (employee + employer share)
   Cr Salaries payable (party)    (balance = net salary)
```

Commission is already in Commissions payable (accrued as earned); the payslip **shows** it in earnings and in net payable but does not re-post it. If deductions exceed the salary part, take the excess from commission: Dr Commissions payable (party) / Cr the deduction account.

## Loans

Document `LN-00001`: employee, principal, installments, installment amount (last absorbs rounding), start month, repayment mode `payroll | direct`, status `Active | Closed | Void`, schedule (month, due, paid, balance).

- Disburse: Dr Staff loans (party) / Cr Money (funds-checked). Print a loan receipt/acknowledgement.
- Payroll repayment: payslip deduction line for the installment due.
- Direct repayment: Dr Money / Cr Staff loans (party).
- Closed automatically when the balance reaches zero. Early settlement allowed.
- Loan ≠ advance: advances are recovered in full from the next salary by default; loans follow the schedule.

## Overview screen (per employee)

Salary · Commission (month) · Deductions · **Net payable** (negative = "owes Rs X") · Paid (month/year, counting only money actually paid out) · Advance outstanding · Loan balance. Never label a gross figure "owed" when the employee also owes the business money.

## Payslip (A4, formal)

```
            <Hospital/Company name + logo>
                  Salary Payslip
┌──────── Employee & Pay Period ─────────────────────────┐
 Employee  Dr. Abdur Rehman     Employee ID  EMP-0012
 Department Medicine            Designation  Consultant
 Pay period 01–31 Oct 2026      Pay date     01 Nov 2026
┌──────── Earnings ──────────┬──────── Deductions ───────┐
 Basic salary      174,194   │ Advance recovery   30,306
 Commission          3,200   │ Loan installment    5,000
 Eidi                5,000   │ Income tax          2,500
                             │ EOBI                  370
 Gross earnings    182,394   │ Total deductions   38,176
                                   Net pay       144,218
 Rupees one lakh forty-four thousand two hundred eighteen only
 Loan balance after this month: 25,000
 This is a computer-generated document and requires no signature.
```

No transaction list on the payslip — payments and advances belong on the **pay statement** (a running ledger for the employee).

## Pay statement

Employee party ledger across Salaries payable, Commissions payable, Staff advances and Staff loans for a From→To range: opening, every payslip/payment/advance/loan line, running net, closing.
