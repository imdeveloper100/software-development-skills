# Chart of accounts

## Structure: groups and ledgers

The chart is a **tree**. Two node kinds:

| Node | Can be posted to? | Balance |
|---|---|---|
| **Group** (header) | No | Sum of its descendants |
| **Ledger** (postable account) | Yes | From its own journal lines |

Five root types fix the normal side of every account below them:

| Type | Normal side | Grows with | Statement |
|---|---|---|---|
| Asset | Debit | Debit | Balance sheet |
| Liability | Credit | Credit | Balance sheet |
| Equity | Credit | Credit | Balance sheet |
| Income | Credit | Credit | P&L |
| Expense (incl. cost of sales) | Debit | Debit | P&L |

`normalBalance(type, debit, credit) = asset|expense ? debit − credit : credit − debit`.

A child must have the same type as its parent. Re-parenting is allowed only within the same type. Archive (not delete) a ledger, and only when its balance is zero; archived ledgers keep their history and still appear in reports for periods where they had activity.

## Codes

Numeric, sortable, stable. Common layout:

| Range | Type |
|---|---|
| 1000–1999 | Assets (10xx cash & bank, 11xx receivables, 12xx advances/loans/inventory, 13xx tax recoverable, 15xx fixed assets) |
| 2000–2999 | Liabilities (20xx payables, 21xx staff payables, 21xx/22xx taxes payable, 25xx loans taken) |
| 3000–3999 | Equity (30xx capital per owner, 31xx drawings per owner, 39xx opening balances / retained earnings) |
| 4000–4999 | Income |
| 5000–5999 | Cost of sales |
| 6000–6999 | Operating expenses |

- New ledgers under a group take the next free code inside the group's range.
- Codes and names are copied onto each journal line at posting time, so renaming an account never rewrites history.
- Give every account the code your code needs a **stable `systemKey`** (e.g. `ar_patients`, `salary_expense`) — posting rules look accounts up by key, never by code or name, so the business can renumber/rename freely. User-created accounts have no key.

## Money accounts

Money accounts are ledgers where cash/bank/wallet balances live. Mark them with `moneyKind: cash | bank | wallet`.

- **Cash.** One ledger per **place or person holding cash**: "Main cash / safe", "Cash – Reception counter", "Cash – Ayesha (cashier)", "Cash – Manager". This is what lets the books show who holds the money; handovers are transfers between them (see `cash-shifts.md`).
- **Bank.** One ledger **per bank account**: bank name, account title, account number, optional `heldBy` (the owner whose name the business account is titled in). "Meezan – Arshad – 0123", "HBL – Awais – 4455".
- **Wallet.** One per wallet/merchant account (JazzCash, Easypaisa, Stripe…).

Payment methods (Cash, Card, Bank transfer, Cheque, Online) are **how** money moved; money accounts are **where** it landed. Resolve a payment to a specific money account in this order:

1. account explicitly chosen on the form;
2. for cash taken by a cashier on shift: that cashier's custody account;
3. the default account configured for the method (Settings › Finance);
4. the method's group's first/main account.

Card receipts usually settle into a bank account (optionally via a "Card clearing" asset if settlement lags and fees must be tracked).

Personal accounts of owners are **not** in the chart. Money from an owner's personal account into the business is **Capital in**; money from the business to the owner is **Drawings**.

## Fixed assets

Group "Fixed assets" with sub-groups: Land & buildings, Vehicles, Machinery & equipment, Furniture & fixtures, Computers & printers. One ledger per significant item ("Vehicles – Honda Civic LEB-1234") or per class for small items. Assets come in by purchase (Dr asset / Cr money or payable) or as owner capital (Dr asset / Cr Capital – owner). Depreciation (when implemented) posts Dr Depreciation expense / Cr Accumulated depreciation – <class> (contra-asset).

## Owners' equity

Per owner:
- `Capital – <owner>` (group Owners' capital — "IN")
- `Drawings – <owner>` (group Owners' drawings — "OUT"; equity account with a normal **debit** balance that reduces equity)

Plus `Opening balances` (equity) for migrating existing balances, and `Retained earnings` if the product closes years. Profit for the period is shown in equity on the balance sheet as "Current year earnings" (computed: income − expenses), not posted, unless a year-end close is implemented.

## Taxes

- `GST input tax` (asset, recoverable) — only if the business is GST-registered.
- `GST output tax` (liability) — only if the business charges GST on sales.
- `Withholding tax payable – suppliers` (liability)
- `Income tax withheld – salaries` (liability)
- Contribution payables (EOBI, social security, provident fund) (liability)

## Starter charts by product

Always include: the money groups (Cash in hand, Bank accounts, Online wallets), Receivables, Staff advances & loans, Fixed assets, Payables, Taxes payable, Salaries payable, Owners' capital/drawings, Opening balances, Cash short/over, Salaries & wages.

| Product | Add |
|---|---|
| **Hospital / clinic** | Patient receivables; income per department (OPD, Surgery, Lab, Radiology/Ultrasound/ECG/X-Ray, Wards & rooms, Pharmacy, Canteen); Commissions payable to doctors/staff + Staff commissions & shares (expense); inventory per store; cost of sales per store; Patient deposits (liability) for advance deposits. |
| **Pharmacy** | Inventory – medicines (batch/expiry tracked in stock module, value in one ledger or per category); Cost of goods sold; Stock losses (expiry, breakage, shrinkage); Customer receivables (credit customers, panels/insurers); Sales returns (contra-income) or reverse sale; GST output if registered/applicable; Supplier payables; Purchase discounts/bonus stock. |
| **Retail POS** | Sales per category; Sales returns; Discounts allowed (expense or contra-income); Inventory; COGS; Card clearing; Gift cards / store credit (liability); Customer receivables (credit sales); per-till cash custody accounts. |
| **Inventory / distribution** | Inventory per warehouse; Goods in transit; COGS; Purchase returns; Freight-in (capitalised into inventory cost or expense); Customer receivables with credit limits; Customer advances (liability); Supplier advances (asset). |
| **School ERP** | Student fee receivables (party = student); Fee income per head (Tuition, Admission, Transport, Exam, Hostel); Concessions/scholarships (contra-income); Fee received in advance (liability); Security deposits refundable (liability); Late fee income; Teacher salaries; Transport expenses. |

## Migration from a flat chart

- Add `parentId`, `isGroup`, `moneyKind`, bank details, `heldBy` as optional fields.
- Create the groups, then parent existing ledgers under them. Keep existing codes and keys; rename the old single "Cash"/"Bank"/"Wallet" to "Main …" — they become the first child of their group. No journal rewrite.
- Migrations are idempotent and safe to re-run.
