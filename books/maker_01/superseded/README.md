# Superseded record — one-leg convention

This directory holds the **entire** previously published record of this book,
moved here verbatim on 2026-08-27. Nothing in it has been rewritten: a chained,
timestamped record that can be edited is not a record. It is simply no longer
presented.

## Why it was superseded

The broker account runs in **hedging** mode. An opposing order there does not
close the existing ticket, it opens a second one — so the account reports no
realised profit at all while a matched pair is open, even though the pair's
value is already fixed. For a matched buy and sell of equal size:

    profit_buy  = (P_market − P_buy_entry)  × V
    profit_sell = (P_sell_entry − P_market) × V
    ---------------------------------------------
    sum         = (P_sell_entry − P_buy_entry) × V

The market price cancels. The pair is economically realised the moment the
second ticket opens.

The superseded record counted the derivatives leg and, on the broker side, only
what the account had already settled. That is not half the answer — it is the
**wrong sign**: the 2026-08-26/27 round trip reads −4.50 USD on one leg where
the pair makes it +0.18, and the book as a whole read −1.80 USD where it is
+3.19.

Two further faults were found at the same time and are fixed in the new chain:

* a settled ticket was counted **twice** — once in the settled deals, once in
  the pair it belonged to (+0.85 on 08-13 and −1.58 on 08-25, double);
* the calendar came from the broker's daily bars, which have no Saturday or
  Sunday, so weekend funding was carried onto the following Monday and two
  rows were missing from a curve that is meant to be daily.

The superseded chain also starts on 2026-08-05, a week before this book's
inception: those sessions are manual fee calibration, not the strategy.

## What replaces it

A new chain, from genesis, under schema `rvb.track-record.snapshot/maker/3`.
It starts at the inception (2026-08-12) and carries every day since, weekends
included.

## Verifying this archive

`CHAIN.jsonl` here is the exact set of lines removed from the repository-level
chain, in order, unmodified. Each entry's `sha256` still matches its file in
`snapshots/`, and each `prev_hash` still matches the previous entry's `hash`,
back to the 64-zero genesis. It verifies as it always did — it is just wrong
about the world, which no hash can detect.
