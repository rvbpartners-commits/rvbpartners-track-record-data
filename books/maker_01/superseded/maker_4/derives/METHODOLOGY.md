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
only the days the broker quotes.

**A weekend is not a flat weekend — only the broker leg is flat.** The other
venue trades 24/7, and funding accrues there whether or not the broker is open.
Saturday 22 and Sunday 23 August each carry their own funding (-0.0395 and
-0.0326 USD); they are separate rows with their real values, not a pair of
zeros and not a lump attached to the Monday. A calendar restricted to the days
the broker quotes did exactly that, and the sum still looked complete.

Nothing is interpolated on a day the broker does not quote: its conversion rate
is the last rate actually observed, carried forward, and the leg it converts is
zero anyway. A day on which nothing traded at all is a **zero**, never a missing
row.

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

## 6. Open positions are disclosed, never marked

Every closed session is published, **whether or not the book was flat at that
close**. The value is what this record measures and nothing else: closed deals,
plus the profit of the opposite ticket pairs locked on that broker day.

An unmatched remainder is not in that number — not marked, not estimated. Its
value is not determined yet, and it will appear on the day it locks. It carries
the only market risk the book holds, and none of the published profit. Each
snapshot states it: `nav.unmatched.tickets` and `nav.unmatched.net_volume`.

This was once a refusal to publish, and that was wrong twice over. It defended a
claim this curve does not make — a mark-to-market equivalence that only held
under the earlier one-leg convention. And it would have gone off constantly: the
strategy opens its positions in the evening by construction (26 August, entered
22:31 UTC, held until 00:26), so a flatness requirement would have switched the
record off every other night.

What the record still refuses: a session whose value cannot be established. A
locked pair must attach to **exactly one** round trip. Zero would drop the
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
