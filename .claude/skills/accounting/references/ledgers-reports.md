# Ledgers and reports

All reports read journal lines. Two time shapes:

| Shape | Question | Reports |
|---|---|---|
| **Period** (From → To) | What happened between two dates? | P&L, ledger, cash book, collections, statements, taxes |
| **As of** (one date) | What is the position at the end of a date? | Balance sheet, trial balance, aging, receivables/payables, custody balances |

## Date ranges

- Every period report offers presets (Today, Yesterday, This week, Last 7 days, This month, Last month, This quarter, This year) **and always-visible From and To date inputs**. A preset just fills From/To; editing either shows "Custom".
- Resolve presets and "today" in the **business's timezone**, not the browser's or the server's.
- Send `from`/`to` as `YYYY-MM-DD`, inclusive both ends. Validate `from ≤ to` server-side. Put them in the URL so the view is shareable and survives refresh.
- Timestamps (shifts, audit) use the tenant timezone day boundaries; dated documents filter on their accounting date.

## Account ledger

For one account (or a **group**, as a roll-up of all descendants) and From→To:
```
Opening balance (all lines dated before From)
Date · Entry no · Source · Description/party · Debit · Credit · Running balance
Closing balance
```
- Balance shown on the account's normal side.
- Paginate; compute the opening balance of each page on the server (opening + Σ lines before the page). Never truncate silently.
- Filter by party to get a **party statement** (supplier, employee, owner, patient, student).
- Click an entry → the journal entry → the source document.
- Printable (A4) with the business letterhead and range.

## Trial balance (as of)

Every ledger with its debit or credit balance; Σ debits = Σ credits. Group subtotals optional. If it doesn't balance, the posting layer is broken — fail tests.

## Profit & loss (period)

Income by group/ledger − Cost of sales = Gross profit − Expenses = Net profit. Optional comparison column (previous period). Transfers and equity movements never appear here.

## Balance sheet (as of)

Assets = Liabilities + Equity, where Equity = Capital per owner − Drawings per owner + Opening balances + Profit to date (current period earnings computed from P&L). Show groups with subtotals; money accounts listed individually.

## Cash book / bank book (period)

Ledger of one or all money accounts with receipts/payments columns, by user/shift split for cash.

## Journal

List of entries with: date & time, entry no, source + document no, description/party, by (user, shift), **amount moved**, **kind badge** (Income / Expense / Transfer) and profit effect for income/expense. Filters: From→To, source type, kind, account (money account), party, user, search. Expand → lines.

## Operational reports commonly needed

- Daily/period collection by user/shift/method/department.
- Outstanding receivables and payables (as of) with aging.
- Owner summary per owner: capital, drawings, profit share.
- Cash position: each custody/bank/wallet balance now.
- Staff: net payable, advances, loans.
- Taxes: GST input/output, WHT deducted/paid.

## Performance

- Index journal lines by (tenant/branch, account, date) and (branch, party, date).
- Aggregate in the database; never load all entries into memory.
- Cache heavy aggregates per branch with a version key bumped on every posting.
