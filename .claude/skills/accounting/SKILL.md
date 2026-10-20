---
name: accounting
description: Double-entry accounting rules for business apps (hospital/HMS, pharmacy, retail POS, inventory/distribution, school ERP, any SaaS that moves money) — chart of accounts with groups and sub-accounts, money accounts (cash custody, multiple banks, wallets), posting rules and an append-only journal, reversals instead of edits, ledgers and party sub-ledgers, transfers between accounts, owners' capital and drawings (money or assets), cashier shifts with blind count and cash handover, payroll (earnings/deductions, commission, advances vs loans, EOBI, salary withholding tax, payslip format), suppliers/payables with bills, GST input and withholding tax, period locks, funds checks, trial balance / P&L / balance sheet, From→To date ranges, and 80 mm thermal receipts. Use whenever a feature creates, moves, reports or prints money — sales, receipts, refunds, expenses, purchases, salaries, capital, transfers, fees, tax — or when designing a ledger, report or receipt.
---

# Accounting

## Purpose

Money features must produce books an accountant would sign: every rupee traceable to a source document, a person and a moment; nothing edited or deleted after the fact; balances that always reconcile; reports that agree with each other. Owners in small businesses (hospitals, pharmacies, shops, schools) use these books mainly to **detect theft and leakage**, so traceability beats convenience.

This skill is domain-agnostic. `references/hms-reference.md` shows one full implementation (NestJS + Mongo + React).

## Load the references you need

| Task | Read |
|---|---|
| Designing/changing accounts, groups, codes, bank/cash accounts, a new product's starter chart | `references/chart-of-accounts.md` |
| Any new money event (sale, refund, purchase, expense, transfer, capital, salary, fee, tax) | `references/posting-patterns.md` |
| Cashier/reception shifts, cash count, handover, Z-report | `references/cash-shifts.md` |
| Salaries, commission, advances, loans, EOBI, salary tax, payslips | `references/payroll.md` |
| Suppliers, bills, payables, GST, withholding tax, aging, supplier statement | `references/payables-tax.md` |
| Ledgers, trial balance, P&L, balance sheet, cash book, statements, date filters | `references/ledgers-reports.md` |
| Receipts, slips, invoices, payslips, printing on thermal printers | `references/receipts-printing.md` |
| Seeing a working example in code | `references/hms-reference.md` |

## Non-negotiable invariants

1. **Double entry.** Every journal entry has ≥ 2 lines; Σ debit = Σ credit; each line is a debit *or* a credit, never both, never negative. Reject unbalanced entries at the service boundary, not in the UI.
2. **Integer minor units.** Store money as integers (whole rupees in PKR apps, paisa/cents otherwise). Never floats. Rounding happens once, in a named function, and the remainder goes to a defined line (usually the last installment / last line).
3. **Append-only journal.** Journal entries are never updated or deleted (enforce with DB hooks / permissions, not just convention). A correction is a **new entry** — a reversal or a difference ("diff posting"). The original stays visible.
4. **Source documents own the money; the journal mirrors them.** Users create documents (token, invoice, bill, payslip, transfer). A **posting rule** — a pure function `document → lines` — turns each document into journal lines. Re-syncing a document posts only the difference between what the rule says now and what is already posted for `(sourceType, sourceId)`. That makes posting idempotent and corrections automatic.
5. **Void, don't delete.** Documents get `status: Void` + `voidReason` + who/when. The posting rule returns no lines for a void document, so re-sync reverses it.
6. **Every account has a ledger.** Any account (and any group, as a roll-up) must be openable as a ledger with opening balance, dated lines, running balance and closing balance for any From→To range.
7. **Parties are sub-ledgers.** Receivable/payable/advance/loan/capital lines carry `partyType` + `partyId` + `partyName` (patient, customer, student, employee, supplier, owner) so each party has its own statement without one account per party.
8. **Transfers are not income or expense.** Moving money between your own accounts (cash→bank, counter→safe, bank→bank, cashier→manager) is Dr destination / Cr source, profit effect 0, and must be shown as a transfer, filterable, with from → to.
9. **Owner money is equity.** Capital brought in and drawings taken out are never income, expense, salary or advance. Each owner has their own Capital (IN) and Drawings (OUT) accounts. Capital can be money or any asset at agreed value.
10. **Period lock.** Books close through a date; nothing may be dated on or before it. Late corrections post on today's date. Reopening is owner-only and audited.
11. **Funds check.** Money going out of a cash/bank/wallet account must not exceed that account's balance (as of the posting date). Credit purchases are not payouts.
12. **Who, when, where.** Every entry stores created-by (id + name), timestamp, IP/device, branch/tenant, and the shift it belongs to when shifts exist.
13. **Scope.** Accounts are per tenant (shared chart); balances are per branch/company because entries are branch-scoped. Never take tenant id from the request body.
14. **Reports read the journal only.** P&L, balance sheet, trial balance, cash book, statements, dashboards are all computed from journal lines — never from document tables — so they always agree. Cache derived figures only with explicit invalidation.

## Decision checklist for any new money feature

1. **What is the source document?** Name it, give it a human number (`TRF-00001`), status, void reason, created-by.
2. **What lines does it post?** Write the posting rule from `posting-patterns.md`. List every account and whether it carries a party.
3. **Which money account?** Payment method alone is not enough when there are several banks/wallets or per-cashier cash: resolve to a specific account (explicit choice → method default → cashier custody).
4. **What date?** Document date for the first posting; correction date (today) for later diffs; refuse closed periods.
5. **Can it pay out more than exists?** Add the funds check.
6. **Does it change profit, or only move value?** Make the journal's "kind" (income / expense / transfer) obvious in the UI.
7. **Who may do it?** Server-side permission; managers-only for discounts, refunds, voids, back-dating; owner-only for equity and reopening books.
8. **Is there a party?** Attach it, so the party statement is complete.
9. **What does it print?** Payment events print an 80 mm receipt; formal documents (invoice/bill, payslip, statement) print A4.
10. **Test:** an e2e scenario that runs the feature then asserts the **trial balance still balances** and the relevant account/party balances are exactly as expected, including after void.

## Red flags (stop and fix)

- `UPDATE`/`DELETE` on journal lines, or "edit amount" on a posted document without a reversal.
- A single `amount` field with sign meaning "in/out" instead of debit/credit lines.
- Balances stored on documents and incremented — balances are derived from the journal (cache only with invalidation).
- Hard-coding "Cash → account 1010" when there can be several cash/bank accounts.
- Overpayment silently turned into an advance/credit. Ask, warn, or refuse.
- Owner withdrawals booked as expenses or salary; transfers booked as income.
- "Owed" figures that ignore advances/loans the person owes back — show **net payable**.
- Date filters resolved in browser time instead of the business's timezone; presets without a free From→To range.
- Thermal receipts sized as A4 or with a fixed page height that splits long receipts.
- GST charged by an unregistered buyer posted as recoverable input tax (it is part of cost).

## Definition of done (money features)

- Posting rule is a pure function with unit tests (balanced, void → no lines, rounding).
- Service checks: permission, period open, method enabled, funds available, party attached.
- Document is numbered, voidable with reason, audited.
- Journal shows it with a clear kind and a readable description; it appears in the right ledgers and party statement.
- Trial balance e2e check passes; reports agree.
- Receipt/print exists in the right format.
