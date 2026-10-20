# Posting patterns

Each pattern: **document → journal lines**. "Money" means the resolved money account (specific cash custody / bank / wallet ledger — see `chart-of-accounts.md`). `(party)` means the line carries partyType/partyId/partyName. All amounts are integer minor units.

A posting rule is a pure function returning `{ sourceType, sourceId, docDate, description, reference, method, lines[] }`. A void/cancelled document returns `lines: []` so re-sync reverses it. Posting compares the rule's lines with what is already posted for the same source and writes only the difference (per account + party).

## Entry kind (what the journal shows)

Compute from the net lines, not from the source type:
- any net income/expense line → **income** (net income > 0) or **expense**; show the profit effect (+/−);
- otherwise → **transfer**; show the amount moved and "from → to", never a blank or the word alone.

Always show the **amount moved** (Σ debits) in its own column, and the kind as a badge, so transfers are as readable as sales.

## Sales and income

| Event | Debit | Credit |
|---|---|---|
| Cash/card sale or paid service (OPD token, lab test, fee) | Money | Income – <department/category> |
| Sale on credit / service billed now, paid later | Receivables (party) | Income |
| Payment received against a bill | Money | Receivables (party) |
| Advance / deposit before the bill exists | Money | Customer/patient deposits (liability, party) |
| Deposit applied at billing | Deposits (party) | Receivables (party) |
| Discount given at sale | — (post net income) — or Discounts allowed (expense/contra-income) if it must be reported | |
| Sale with inventory (perpetual) | + Dr COGS / Cr Inventory at cost | |
| GST on sales (registered seller) | Money/Receivable (gross) | Income (net) + GST output tax (tax) |

## Refunds and returns

| Event | Debit | Credit |
|---|---|---|
| Refund of a paid sale/service (cancelled) | Income (or Sales returns) | Money |
| Refund when cancel reverses the original document | Void the source → re-sync reverses income + money automatically; if the refund leaves by a **different** method/account, also post Dr original money / Cr refund money | |
| Refund of a deposit | Deposits (party) | Money |
| Goods returned to stock | Inventory | COGS (at original cost) |

Refunds need manager permission, a reason, and the funds check on the paying account.

## Purchases and payables

| Event | Debit | Credit |
|---|---|---|
| Cash purchase of stock | Inventory | Money |
| Credit purchase / supplier bill | Inventory / Expense / Fixed asset (net) | Supplier payables (party, gross) |
| GST on bill, **registered** buyer | + GST input tax (recoverable) | |
| GST on bill, **unregistered** buyer | add GST into the cost line (not recoverable) | |
| Supplier payment | Supplier payables (party, gross) | Money (net paid) + WHT payable – suppliers (withheld) |
| Advance to supplier | Supplier advances (asset, party) | Money |
| Purchase return / debit note | Supplier payables (party) | Inventory/Expense (+ GST input reversal) |
| Supplier opening balance | Opening balances | Supplier payables (party) |

## Expenses

| Event | Debit | Credit |
|---|---|---|
| Paid expense (utilities, rent, repairs) | Expense – <category> | Money |
| Expense billed now, paid later | Expense | Accrued expenses / Supplier payables (party) |
| Prepaid (rent for a year) | Prepaid expenses (asset) | Money; monthly: Dr Expense / Cr Prepaid |

Reserve categories that belong to other modules (salaries, commissions) so they can't be booked as plain expenses.

## Transfers between own accounts

| Event | Debit | Credit |
|---|---|---|
| Cash deposited in bank | Bank – <account> | Cash – <holder> |
| Cash withdrawn from bank | Cash – <holder> | Bank – <account> |
| Bank to bank / bank to wallet | Destination | Source |
| Cashier hands cash to manager (accepted handover) | Cash – Manager | Cash – Cashier |
| Counter float issued | Cash – Counter | Main cash / safe |
| Bank charges on a transfer | + Dr Bank charges (expense) / Cr source | |

Document: `Transfer` with from, to, amount, date, reference (cheque/slip no.), note, status. Funds-check the source. Kind = transfer.

## Owners

| Event | Debit | Credit |
|---|---|---|
| Capital in — money | Money | Capital – <owner> |
| Capital in — asset (car, machine, printer, property) | Fixed assets › <item> (agreed value) | Capital – <owner> |
| Capital in — owner pays a business bill personally | Expense / Payables | Capital – <owner> |
| Withdrawal — money | Drawings – <owner> | Money (funds-checked) |
| Withdrawal — asset taken home | Drawings – <owner> | Fixed asset (book value; can't exceed it) |
| Owner's personal expense paid by business | Drawings – <owner> | Money (never an expense) |
| Opening capital on go-live | Money/assets | Capital – <owner> or Opening balances |

Drawings are equity with a debit balance. Owner's equity = Capital − Drawings + share of profit.

## Payroll (details in `payroll.md`)

| Event | Debit | Credit |
|---|---|---|
| Commission accrues (service performed/collected) | Commission expense | Commissions payable (party) |
| Payslip finalised | Salary expense (earnings) + employer contribution expense | Staff advances (recovery) + Staff loans (installment) + Income tax withheld – salaries + Contribution payables (employee + employer share) + Salaries payable (party, net) |
| Salary/commission paid | Salaries payable / Commissions payable (party) | Money |
| Advance (paid ahead of salary) | Staff advances (party) | Money |
| Loan disbursed | Staff loans (party) | Money |
| Loan repaid directly | Money | Staff loans (party) |
| Tax / EOBI paid to authority | Tax withheld / EOBI payable | Bank |

## Inventory

| Event | Debit | Credit |
|---|---|---|
| Opening stock | Inventory | Opening balances |
| Count shows less stock (loss, expiry, theft) | Stock losses | Inventory |
| Count shows more stock | Inventory | Stock losses (or Stock gains income) |
| Internal consumption (store → ward) | Consumables expense / department | Inventory |

Value at moving-average (or FIFO) cost captured at the movement, never recomputed later.

## Cash control

| Event | Debit | Credit |
|---|---|---|
| Count short | Cash short / over (expense) | Cash – <holder> |
| Count over | Cash – <holder> | Cash short / over |

A difference always needs a note.

## Fixed assets

| Event | Debit | Credit |
|---|---|---|
| Asset bought | Fixed asset | Money / Supplier payables |
| Monthly depreciation | Depreciation expense | Accumulated depreciation – <class> |
| Asset sold | Money + Accumulated depreciation | Fixed asset (cost) ± Gain/Loss on disposal |

## School ERP

| Event | Debit | Credit |
|---|---|---|
| Monthly fee voucher issued | Student fee receivables (party) | Fee income – <head> |
| Concession / scholarship | Concessions (contra-income) | Student fee receivables |
| Fee received | Money | Student fee receivables |
| Fee received in advance | Money | Fees received in advance (liability) → released monthly |
| Late fee | Receivables | Late fee income |
| Security deposit taken / refunded | Money ↔ Security deposits refundable (liability) | |

## Taxes

| Event | Debit | Credit |
|---|---|---|
| GST return: input vs output | GST output tax | GST input tax + Bank (net payable) |
| Withholding paid to tax authority | WHT payable / Income tax withheld | Bank |

## Corrections

- Wrong amount on a posted document: edit the document (if policy allows) → re-sync posts the **difference** on today's date (or the correction date) with a reference to the original.
- Wrong account/category: re-sync moves the amount (Dr new / Cr old) as a difference entry.
- Manual voucher (journal voucher) for anything without a source document: free lines, must balance, cannot use system-only accounts that modules own (receivables, payables with parties) unless the user has the accountant role, cannot be dated in a closed period.
