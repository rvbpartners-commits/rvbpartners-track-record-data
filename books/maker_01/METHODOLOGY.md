# RVB-MAKER-01 — methodology

Delta-hedged liquidity provision across two venues. Real capital.

This book trades **real capital**, which belongs to the operator. No
third-party money is managed. Read this page before comparing it with the paper
books published beside it: it is measured differently, because it is a
different kind of thing.

## Where the numbers come from

**One source of truth.** Every profit-and-loss figure on this page is taken
from the trading system's own machine export, produced on the trading box once
a day. That export reads the broker's deal registry and the derivatives venue's
API directly, and reports, per broker session:

* the realised profit and loss of the derivatives leg (fills net of fees, plus
  funding),
* the realised profit and loss of the broker leg (deals net of swap and
  commission),
* the capital movements on each account.

The publisher **recomputes none of it**. It compounds those per-session figures
from this book's frozen opening capital, unitises the capital movements out of
the return, and reconciles the result against the equity actually read on both
accounts. Earlier chains of this book did their own matching and their own
profit-and-loss reconstruction; two independent measurements of one account are
two public numbers free to disagree, and nothing in the repository would have
said which was wrong.

## The accounts are shared; the record is not

Both venue accounts carry this strategy **and** whatever the operator does by
hand on the same venues. A record that adds the two together stops measuring the
strategy — and the difference is not academic: on the inception session itself,
manual purchases on unrelated instruments came to **-17.79 USD** against a
strategy day of **+3.10**.

So the export separates them at the source, by the identity of what was traded,
and anything outside the strategy's own instruments is treated as a **capital
movement**: it is unitised exactly like a transfer, moving equity and units
while leaving the unit price — the number this record reports — untouched. It is
not deleted. Deleting it would break the reconciliation below by its own size,
because the equity actually read on the accounts contains it.

One limit, stated rather than glossed: the derivatives venue carries no order
tag, so a trade placed **by hand on one of the strategy's own instruments**
would still be counted as the strategy's. Closing that needs a tag set when the
order is sent, and there is none today.

## The opening capital, and why it is frozen

The record is anchored to **1,796.40 USD**, the equity read on
both venue accounts on the inception session (2026-09-10). It is written once,
to `inception.json`, and never recomputed: it divides every return this book
publishes, so a value free to move would silently restate the whole record.

## The realised profit of the derivatives leg is frozen per session

The derivatives venue pays funding hourly, and its API compacts those hourly
events into one calendar-day row once they are about ten days old. This book's
session closes at 21:00 UTC, so a calendar-day aggregate cannot be split at
that boundary: the 21:00-24:00 slice, which belongs to the next session, falls
back onto the previous one. The past then changes value on its own, with no
fill moved and the total carry conserved. Measured on 2026-09-20: 0.366514 USD
left the 2026-09-11 session for 2026-09-10 between two exports of identical
code.

So the realised profit and loss of the derivatives leg is frozen the first time
a session is published (`venue_a_pnl.json`, write-once per session), beside the
exchange rate (`fx_rates.json`). A published record is always recomputed from
those frozen figures, never from a later export.

The source export is stateless and re-renders its whole history every night.
When it comes to disagree with a frozen figure, the record is neither stopped
nor restated: the disagreement is published in `meta.json` under
`source_divergence`, session by session with its delta, and the published
record keeps the figure it was built from. A published attribution is what was
measured then; a re-bucketing that leaves the total unchanged is a fact about
the venue's API, not about the session.

## Capital movements never become performance

The account has received deposits, and will receive more. A deposit is a
**flow**: it buys units at that day's unit price, so it moves equity and never
moves the price.

    units_after = units_before x (equity_before + flow) / equity_before
    unit_price  = equity / units

This is not a fine point. The transfers of 2026-09-09 tripled the capital of
this account. Counted as performance they would read as +250 % in one day.

## The reconciliation residual, in two parts

Every run compares the curve with the equity actually read on both accounts
and publishes the difference in basis points, whether or not it is zero. The
two are not expected to be equal: the accounts are read hours after the last
published session closed, and the reading converts the whole broker-currency
balance at the day's rate while the curve carries that balance at frozen
rates. So the residual is published in two parts. **Translation** is what the
broker-currency balance is worth at the day's rate against the frozen rates.
**Unexplained** is everything the rate does not account for -- the session in
progress at the time of reading, the floating value of open positions, and any
real error. A run warns when the total exceeds 60 bp and refuses to publish
when the unexplained part exceeds 100 bp.

The export reads one combined equity and never the balance of each leg, so the
broker-currency balance is **derived**: the balance read on each account on the
inception session, frozen once, carried forward with the flows and realised
profit the export reports in that currency for every session. The derivation
tracks the real balance to about 11 EUR -- the account carries the floating
value of open positions, the curve books closed deals -- which moves the split
by about one basis point against a threshold of one hundred. It breaks if
broker deals land in the out-of-perimeter column, a single USD figure carrying
both legs: on such a day no split is published, and the refusal applies to the
whole residual, as it did before the split existed.

## The session, and the calendar

A session is the **broker's own trading day**: its midnight falls at 21:00 UTC.
Every day is a row, weekends included — the derivatives venue trades 24/7, so
Saturday funding is a fact about Saturday. A day with no trade is a zero, never
a missing row.

Only **closed** sessions are published. The source export is produced hours
before the session it last touches has closed, so its final row is always a
session still in progress; publishing it would fix a half-measured day into a
write-once record forever.

## What is NOT published, and why

* **The strategy.** The instruments, the venues, the sizes, the thresholds. The
  composition of this book *is* the strategy.
* **Per-session execution counts.** The source export reports its execution
  counts over its whole window, not per session. What is published instead is
  the number of sessions on which the book's profit and loss **moved**, under a
  name that says exactly that. It is not the same measurement: funding accrues
  while a position is held without any order being placed.
* **Open positions.** Profit and loss is realised. A position carried past the
  close is not marked and not estimated; it enters the record on the day it
  closes.

## Annualisation is withheld

Nothing annualised — Sharpe, Sortino, Calmar, CAGR, volatility, drawdown,
value-at-risk — is published below **60 sessions**. This
book publishes **14**. Annualising a handful of sessions produces a
number with the shape of a statistic and none of its content.

Where it is ever released, annualisation uses **365 periods a
year**, measured on this broker's own calendar (400 daily bars over 564
calendar days), not the 252 the equity books use.

Sharpe, Sortino and Calmar are quoted **excess of the risk-free rate**, which
is published beside them with its source. Interest on cash is not alpha.

## Verifying it

Every session has an immutable snapshot under `snapshots/`, hashed, carrying
the hash of the previous session, recorded in the repository-level
`CHAIN.jsonl` and timestamped by OpenTimestamps. The last session published is
**2026-09-23**. See `VERIFY.md` at the root of this repository.

## Earlier chains

This book has restarted its chain before. Each earlier chain is kept **verbatim**
under `superseded/`, with its own genesis, its own `CHAIN.jsonl`, its own
snapshots and its own derived files, and its own note saying why it was
retired. Nothing is deleted: a chained, timestamped record that can be edited
is not a record. They are simply no longer presented.
