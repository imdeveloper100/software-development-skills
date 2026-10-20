# Suppliers, payables and tax

## Supplier master

`SUP-0001`: name, contact person, phone, email, address, NTN, STRN (sales-tax registration), GST registered (yes/no), category (medicines, lab reagents, utilities, equipment…), payment terms (days), opening balance + as-of date, status Active/Inactive. Tenant-level master; balances per branch come from the journal.

The supplier is the **party** on every payables line. Never make each purchase its own party — the supplier's ledger must show all their bills and payments together.

## Bills (purchase invoices received)

`PB-00001` + supplier's own invoice number, bill date, due date (bill date + terms), lines:

| Field | Notes |
|---|---|
| Description / product | Stock lines go through the inventory module (quantities, batches) |
| Account | Inventory, an expense category, or a fixed asset |
| Qty × rate | Integer money |
| GST % | Per line (0, 5, 17, 18…) from the tax-rate list |

Posting:
```
Dr Inventory / Expense / Fixed asset   (net of GST)
Dr GST input tax                        (only if WE are GST-registered)
   Cr Supplier payables (party)         (gross)
```
If the business is **not** GST-registered, GST is part of the cost: add it to the line account.

Void reverses. A bill with payments allocated can't be voided until the allocations are removed (or void creates a debit balance the supplier owes back — choose one policy and enforce it).

## Payments to suppliers

```
Dr Supplier payables (party)     gross amount settled
   Cr Money                      net paid
   Cr WHT payable – suppliers    withholding deducted
```

- Withholding rate depends on the supplier's status (filer/non-filer, goods/services) — store the rate on the supplier or pick it on the payment; never hard-code.
- Allocate payments to bills: default oldest-first (FIFO), allow manual allocation. Unallocated payment = supplier advance (debit balance on the supplier).
- Funds-check the money account.
- Print an 80 mm (or A4) payment receipt/voucher with the balance block.

## Statement block (on every bill and payment print)

```
Current invoice          1,200
Previous balance        10,000
Total outstanding       11,200
```
For a payment: Previous balance, This payment (and WHT), Balance after payment. "Previous balance" = supplier party balance on Supplier payables **before** this document (as of its date, excluding it).

## Supplier ledger / statement

Party ledger for a From→To range: opening balance, each bill (credit), payment (debit), WHT, returns, running balance, closing balance. Printable A4 to send to the supplier for reconciliation.

## Aging

Outstanding per supplier bucketed by days past due: Current, 1–30, 31–60, 61–90, 90+. Based on unpaid bill balances after allocation.

## Tax configuration

Settings › Taxes:
- "Business is GST-registered" (+ STRN).
- Rates list: name, %, kind (`gst_input`, `gst_output`, `wht_supplier`, `wht_salary`), active.
- Accounts: GST input tax (asset), GST output tax (liability), WHT payable – suppliers (liability), Income tax withheld – salaries (liability).

## Paying tax

"Pay tax" document: tax account (liability), period, amount, bank, challan/CPR number → Dr tax liability / Cr Bank. GST settlement: Dr GST output / Cr GST input / Cr Bank (net). Taxes report for a period: GST input, GST output, WHT deducted (suppliers, salaries), paid, balance payable.

## Other payables

Same party pattern for: doctors' payable shares, landlord, utilities (if billed), insurance panels (receivables), customer credit accounts.
