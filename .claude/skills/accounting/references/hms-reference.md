# Reference implementation: HMS (CurixHub)

Repo: `d:\__ Software Agency\hms` — NestJS + Mongoose API (`apps/api`), React web (`apps/web`), shared TS (`packages/shared`). Use it as a worked example; copy the shapes, not the hospital specifics.

Items marked *(planned)* are part of the 2026-10-20 accountant-review overhaul; update this file as they land.

## Core

| Concept | Where |
|---|---|
| Vocabulary (types, normal balance, sources, kinds, parties) | `packages/shared/src/accounting.ts` |
| Default chart + system keys | `apps/api/src/modules/accounting/chart.ts` |
| Account schema / service (resolve by key, user accounts) | `accounting/account.schema.ts`, `accounts.service.ts` |
| Append-only journal (pre-hooks refuse update/delete) | `accounting/journal-entry.schema.ts` |
| Diff posting, kind/amount summary, balance check | `accounting/journal-math.ts` (+ spec) |
| `sync(rule)`, `postManual`, `assertOpen`, `assertFundsAvailable`, `assertMethodEnabled` | `accounting/journal.service.ts` |
| Posting rules (pure functions) | `accounting/posting-rules.ts` |
| Refund lines when cancelling paid documents | `accounting/refunds.ts`, ADR 0021 |
| Ledger, trial balance, P&L, balance sheet, cash book | `accounting/accounting-reports.service.ts` |
| Cash closing (per user/day — being replaced by shifts) | `accounting/closing.service.ts` |
| Owner capital/drawings | `accounting/owner.service.ts`, ADR 0020 |
| Payroll (runs, payslips, payments, advances) | `apps/api/src/modules/payroll/`, shared `payroll.ts`, ADR 0012 |
| Commissions (accrual, payouts) | `apps/api/src/modules/commissions/`, ADR 0010 |
| Store purchases / supplier payments | `apps/api/src/modules/store/`, ADR 0011 |
| Rebuild the books from documents | `accounting/accounting-rebuild.service.ts`, `cli/migrate.ts` |
| Decisions | `docs/adr/0009-double-entry-accounting.md` and later ADRs |

## Web

| Concept | Where |
|---|---|
| Journal / Chart / Reports / Cash & closing / Owner tabs | `apps/web/src/features/accounting/` |
| Account ledger page | `features/accounting/pages/AccountLedgerPage.jsx` |
| Date range filter | `components/form/PeriodPicker.jsx` |
| Method picker (enabled methods per hospital) | `components/billing/PaymentMethodField.jsx` |
| 80 mm slips | `components/print/SlipDocument.jsx`, `slip-printer.jsx`, `receipts.js` |
| A4 documents | `components/print/PrintDocument.jsx` |
| Payroll screens, payslip | `features/payroll/` |

## Testing pattern

API e2e (`apps/api/test/*.e2e-spec.ts`, mongodb-memory-server) runs the feature through HTTP, then asserts the trial balance balances and the exact account/party balances, including after void. `fundBooks()` in `test/app.ts` puts opening money in so payouts pass the funds check.

## Planned (overhaul)

- Chart tree (`parentId`, `isGroup`, `moneyKind`, bank details, `heldBy`) and per-method default money accounts *(planned)*
- Transfers (`TRF-`), journal Amount + Type columns, Ledgers tab, paginated ledgers *(planned)*
- Owners collection, per-owner Capital/Drawings, asset capital *(planned)*
- Shifts (`SH-`), custody accounts, handover accept, Z-report *(planned)*
- Payroll components, loans (`LN-`), salary WHT, A4 payslip in earnings/deductions format *(planned)*
- Suppliers (`SUP-`), bills (`PB-`) with GST input, WHT on payments, aging, statements *(planned)*
