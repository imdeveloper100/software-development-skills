# Cashier shifts, cash custody and handover

Reception desks, pharmacy counters, POS tills and school fee counters run in shifts (2–3 per 24 h, 8 or 12 hours). The control goal: **at any moment the books say who is holding which cash**, and every shortage is attributable to one person and one shift.

## Model

**Custody accounts.** Each person (or till) that holds cash has their own cash ledger under "Cash in hand": `Cash – Ayesha`, `Cash – Manager Bilal`, `Main cash / safe`. Created on demand. Cash taken by a cashier posts to *their* custody account, not a shared "Cash" account.

**Shift document** (`SH-00001`):
- user, branch, optional template (Morning 08:00 / 8 h…), `openedAt`, `closedAt`, status `Open | Closed`
- `openingFloat` (cash received at start, usually from an accepted handover)
- per money account: `expected`, `counted`, `difference`, note
- `handoverTo` (user), `handoverAmount`, handover status `Pending | Accepted | Disputed`
- totals by method (for the Z-report): received, refunded/paid out, net

Every journal entry created while a shift is open carries `shiftId`. Expected figures are computed from journal lines with that `shiftId` (plus the opening float), **server-side**.

**Templates** (Morning/Evening/Night 8 h, Day/Night 12 h) are labels and "overdue" warnings, not hard limits — real shifts overrun.

## Rules

1. **Shift required.** A user whose role must work in shifts cannot create any document that moves money without an open shift ("Start your shift first"). Enforce in the money-account resolver/service, not just the UI. Managers/owners may be exempt; their cash goes to their own custody account.
2. **One open shift per user** (per branch). Opening a second one is refused.
3. **Timestamps, not dates.** Shifts cross midnight; never group shift money by calendar day.
4. **Blind count.** At close the cashier enters counted cash **before** seeing the expected figure (optionally show expected after submission). Denomination breakdown (5000×n, 1000×n, …) helps but is optional.
5. **Differences post immediately.** Short: Dr Cash short/over / Cr Cash – cashier. Over: the reverse. A note is mandatory. Repeated shortages are a report, not a silent write-off.
6. **Non-cash is confirmed, not counted.** Card/bank/online receipts of the shift are listed so the cashier can tick them against the bank/wallet app or card terminal settlement. Mismatches are flagged with a note (they're investigated, not posted as short/over).
7. **Handover is a transfer the receiver accepts.** On close the cashier picks who receives the cash (next cashier, manager, owner). This creates a *pending handover*; the cash stays in the sender's custody until the receiver accepts. On accept: Dr Cash – receiver / Cr Cash – sender (source `cash-handover`). If the receiver counts a different amount, they accept what they counted with a note; the difference posts to short/over against the sender's custody (the sender is accountable).
8. **Opening float.** The next shift's opening float is the accepted handover (or the user's existing custody balance). Floats issued from the safe are transfers too.
9. **Closed shifts are immutable.** Corrections after close go through a new document by a manager (transfer / adjustment), referencing the shift.
10. **No orphan money.** A user cannot log out/end the day with an open shift unless a manager force-closes it (audited, counted by the manager).

## Z-report (80 mm, printed at close)

```
SHIFT CLOSE  SH-00042
Ayesha Khan · Morning
Opened 20 Oct 08:02  Closed 16:11
----------------------------------
Opening float            5,000
Cash received           84,300
Cash refunded/paid out  (2,000)
Expected cash           87,300
Counted cash            87,000
Short                     (300)
Note: change given wrong
----------------------------------
Card                    12,500
Online                   9,800
----------------------------------
Tokens 64 · Lab 21 · Receipts 85
Handed to: Bilal (Manager)  87,000
Signature ______________________
```

## Screens

- Header badge: "On shift 3h 12m" / "No shift" with Start / End buttons in the user menu.
- End-shift dialog: blind count → non-cash confirmation → difference note → hand over to → print Z-report.
- Pending handovers for the receiver (notification + list) with Accept / Accept with difference.
- Manager view: open shifts now, closed shifts with differences, pending handovers, cash held by each person (custody balances).

## Reports

- Cash position: balance of every custody account + safe + banks, as of now.
- Shift history with differences, by user.
- Collections by shift/user/method for any From→To range.
