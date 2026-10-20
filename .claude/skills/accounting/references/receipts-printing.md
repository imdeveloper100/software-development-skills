# Receipts and printing

## Which format

| Document | Format |
|---|---|
| Payment received (token, fee, sale, deposit), refund, payout, salary/advance/loan payment, supplier payment, owner capital/withdrawal, transfer, shift Z-report | **80 mm thermal** |
| Invoice/bill to a customer or from a supplier, payslip, statements, ledgers, reports, lab/medical reports | **A4** with letterhead |

Every money event prints something; the person handing over or receiving money gets paper.

## 80 mm thermal: continuous roll

Thermal printers feed a continuous roll and cut after the job. The page must be exactly as tall as the content, otherwise the browser splits a long receipt at A4 height or prints a page of blank paper.

- **Width:** 80 mm paper ≈ 72 mm printable. Use a monospaced layout of ~42 characters per line (12 px/9 pt font), or a 72 mm wide HTML block.
- **Height (browser):** `@page { size: 80mm auto }` is **not valid** (the `size` property takes lengths or keywords) and is ignored. Before calling print, measure the rendered receipt (`el.getBoundingClientRect().height` → mm at 96 dpi: px × 25.4 / 96), then inject `@page { size: 80mm <height + 4>mm; margin: 0 }`. For several receipts in one job, use the tallest height for all pages, or print each separately; put `break-after: page` between them so the printer cuts.
- **Height (Electron silent print):** `webContents.print({ silent: true, margins: { marginType: 'none' }, pageSize: { width: 80000, height: <microns> } })`. Pass the measured height from the renderer (mm × 1000). Optionally a `deviceName` chosen in settings for the receipt printer vs the A4 printer.
- Don't use background colours; thermal printers print black only. Use bold/size for hierarchy.

## Receipt contents

```
        <Business name>
     <Branch · address · phone>
----------------------------------
RECEIPT                RCP-00123
20 Oct 2026 11:08 am
Patient: Ali Raza (MR-0042)
----------------------------------
OPD consultation          1,500
Lab – CBC                   800
Discount                   (300)
----------------------------------
TOTAL                     2,000
Paid (Cash)               2,000
Balance                       0
----------------------------------
Cashier: Ayesha · Shift SH-00042
      Thank you · Get well soon
```

Rules:
- Document number, date **and time**, branch, cashier name (+ shift), method, amount paid, balance remaining.
- Party balance block when the party has a running account (supplier: Previous balance / This document / Total outstanding).
- Amount in words for payouts and payment vouchers.
- Reprints are marked **DUPLICATE** (and audited). Void documents print **VOID**.
- Never print internal cost/commission/profit on a customer receipt.

## A4 documents

Letterhead (logo, name, address, registration numbers), document title, number, dates, party block, line table, totals, amount in words, balance block, signature/footer ("computer-generated, no signature required" when appropriate). Use `@page { size: A4; margin: 14mm }`.
