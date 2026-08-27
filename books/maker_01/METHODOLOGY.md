# RVB-MAKER-01 — methodology

This book is not one of the desk's paper portfolios. It trades **real capital**
— the operator's own — across two venues, and it needs a handful of conventions
the equity books do not. Each one is stated here, with the measurement that
forced it.

The general methodology in the repository root still applies: one metrics
module, hash-chained write-once snapshots, no figure computed in a browser.

## 1. What a session is

The **broker's own trading day** (UTC+3). Its midnight falls at 21:00 UTC, which
is the session close used throughout. The calendar is every *calendar* day, not
only the days the broker quotes: the other venue trades 24/7, so Saturday
funding is a fact about Saturday. A day on which nothing traded is a **zero**,
never a missing row.

Annualisation, where it is ever released, uses **259 periods a
year** — measured on this broker's calendar, not the equity desk's 252.

## 2. Both legs, and why one leg inverts the sign

The hedge account runs in **hedging** mode. An opposing order there does not
close the existing ticket, it opens a second one. So the account reports no
realised profit at all while a matched pair is open — even though the pair's
value is already fixed. For a matched buy and sell of equal size:

    profit_buy  = (P_market - P_buy_entry)  x V
    profit_sell = (P_sell_entry - P_market) x V
    ---------------------------------------------
    sum         = (P_sell_entry - P_buy_entry) x V

The market price cancels between the legs. **The pair is economically realised
the moment the second ticket opens**, and its value cannot move afterwards.

The record therefore counts a locked pair on the day it locked, and the broker
leg passes **entirely** through that pairing — matched first-in-first-out, with
an exact pro-rata split when the sizes differ, because floating profit is linear
in size. A settled ticket is **not** counted again: it already lives in the pair
it belonged to.

Both mistakes were made and both are measured. Counting the derivatives leg
alone, the round trip of 26–27 August reads **-4.50 USD** where the pair makes it
**+0.18**, and the book as a whole read **-1.80 USD** where it is **+3.19** —
the sign, not the size. Adding the settled deals *on top of* their pair
double-counted **+0.85** on 13 August and **-1.58** on 25 August.

The unpaired tail is excluded. It is the hedge of a position still open on the
other venue, and it is the only part that still carries market risk.

## 3. Foreign exchange — a conversion of FLOWS

The broker leg is denominated in EUR and is about half this
book. **Each session's leg is converted once, at that session's closing rate,
and never revalued.** No exchange-rate movement therefore enters the curve.

The alternative — revaluing the whole EUR balance at the
current rate, which is what a live dashboard does because it is simpler to
display — injects about **18 bp a day** of variation that is not trading
(daily standard deviation 34.6 bp on roughly half the book), against a measured
**14.9 bp** a day of actual profit and loss. The noise would exceed the signal.

This is the **only** deliberate difference between this published curve and the
operator's local view, and it is worth a few cents a day. Every snapshot
publishes the gap as `fx.delta_bps`, together with both legs in their own
currency, so a reader can reconvert at any rate they like and reproduce either
number.

## 4. What is inside the record, and what is not

* **Inception is 2026-08-12** — the first session inside the operator's
  declared start of automated trading. Executions before it were manual fee
  calibration and are in **no** performance figure. They moved money, so they
  sit inside the opening capital.
* **Nothing before inception is published.** The equity at the *open* of the
  inception session (213.12 USD) is the denominator of
  that first day's return; it is a starting capital, not a point on the curve.
* **Calibration on other instruments is inside the return.** It is real money
  lost on this account inside the window. Treating it as a capital adjustment
  would flatter the curve by about 3.5% of its result. The instruments are not
  published; the impact is, per session and in total.
* **Deposits buy units.** The account was funded three times. A curve derived
  from `NAV_t / NAV_(t-1) - 1` would have printed **+598%** on one of those days.
  Flows change the number of units, never the unit price.

## 5. The unit of account is the round trip

This book does not trade every day. Counting it in sessions gives a denominator
that measures the calendar rather than the strategy. Annualised statistics stay
withheld until **30 round trips** — a stricter bar than the desk's 60 sessions,
because sessions accumulate by the passage of time and round trips do not.

## 6. What the record refuses to publish

A session is published only if the book was **economically flat** at its close:
no position on the derivatives venue, and no unpaired volume on the broker side.
A book that is merely locked has a deterministic value and is publishable; a
book carrying open risk is not, and the publication stops rather than marking it
at a number it does not have.

Each locked pair must attach to **exactly one** round trip. Zero would drop the
amount from the total; two would count it twice; neither is visible in a total.
Both refuse to publish.

The reconstructed equity is reconciled against what the two accounts actually
report, with the unmatched unrealised leg added back. The residual is published
whether or not it is zero — a field that only appears when it is inconvenient is
not a measurement.

## 7. What is deliberately not published

The traded instrument, the venues, the thresholds, the grid, the position size
and the entry and exit logic. The operational log publishes **outages and the
dates of parameter changes** — availability is a fact about a track record that
a reader is owed — but never a parameter's value, which is the strategy itself.

---

*Chain restarted at schema `maker/3` on the convention above. The previous
one-leg chain is kept verbatim under `books/maker_01/superseded/`: it verifies
as it always did, and it is wrong about the world, which no hash can detect.*
