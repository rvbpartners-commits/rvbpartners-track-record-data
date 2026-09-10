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

## The opening capital, and why it is frozen

The record is anchored to **1,796.40 USD**, the equity read on
both venue accounts on the inception session (2026-09-10). It is written once,
to `inception.json`, and never recomputed: it divides every return this book
publishes, so a value free to move would silently restate the whole record.

## Capital movements never become performance

The account has received deposits, and will receive more. A deposit is a
**flow**: it buys units at that day's unit price, so it moves equity and never
moves the price.

    units_after = units_before x (equity_before + flow) / equity_before
    unit_price  = equity / units

This is not a fine point. The transfers of 2026-09-09 tripled the capital of
this account. Counted as performance they would read as +250 % in one day.

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
book publishes **0**. Annualising a handful of sessions produces a
number with the shape of a statistic and none of its content.

Where it is ever released, annualisation uses **259 periods a
year**, measured on this broker's own calendar (400 daily bars over 564
calendar days), not the 252 the equity books use.

Sharpe, Sortino and Calmar are quoted **excess of the risk-free rate**, which
is published beside them with its source. Interest on cash is not alpha.

## Verifying it

Every session has an immutable snapshot under `snapshots/`, hashed, carrying
the hash of the previous session, recorded in the repository-level
`CHAIN.jsonl` and timestamped by OpenTimestamps. The last session published is
**— (no closed session yet)**. See `VERIFY.md` at the root of this repository.

## Earlier chains

This book has restarted its chain before. Each earlier chain is kept **verbatim**
under `superseded/`, with its own genesis, its own `CHAIN.jsonl`, its own
snapshots and its own derived files, and its own note saying why it was
retired. Nothing is deleted: a chained, timestamped record that can be edited
is not a record. They are simply no longer presented.
