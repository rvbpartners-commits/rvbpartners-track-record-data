# Superseded record — locked-pair (HEDGING) convention, maker/3–maker/4

This directory holds the previously published record of this book for
sessions **2026-08-12 through 2026-08-27**, moved here verbatim on
2026-09-05. Nothing in it has been rewritten: a chained, timestamped record
that can be edited is not a record. It is simply no longer presented.

This is a **second** archive, alongside the one already kept directly under
`books/maker_01/superseded/` from an earlier restart. Each restart's chain is
kept in its own subdirectory, verified independently from its own genesis —
that first archive is untouched by this one.

## Why it was superseded

The broker account was converted from **hedging** to **netting** mode, and
the broker reset the account in the process. Under hedging, an opposing order
opens a second ticket rather than closing the existing one, so a matched pair
is realised the moment the second ticket opens — the convention this archive's
records were computed under (see `books/maker_01/superseded/README.md` for the
full explanation and worked example). Under netting, an opposing order closes
the existing ticket directly, and the account realises the difference on that
same closing deal: there is no second, opposite ticket to wait for or match
against.

The two conventions are not interchangeable. Feeding netting-mode activity
through the hedging-era matching logic finds no opposite ticket to pair
against and drops an already-realised, already-banked profit into "still
open, still at risk" instead of publishing it on the day it was earned.

## What replaces it

A new chain, from genesis, under schema `rvb.track-record.snapshot/maker/5`.
It starts at the account's post-reset inception (2026-09-02) and realises each
closed position on its own closing session, independent of any other ticket.

## Verifying this archive

`CHAIN.jsonl` here is the exact set of lines removed from the repository-level
chain, in order, unmodified. Each entry's `sha256` still matches its file in
`snapshots/`, and each `prev_hash` still matches the previous entry's `hash`,
back to the 64-zero genesis.
