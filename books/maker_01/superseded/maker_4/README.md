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

---

## Addendum, 2026-09-10 — what actually replaced it

The section above announces a replacement chain under schema
`rvb.track-record.snapshot/maker/5`, starting at the post-reset inception
2026-09-02. **That chain never published a session.** `snapshots/` stayed
empty from 2026-09-05 to 2026-09-10, the repository-level `CHAIN.jsonl` carries
no `maker_01` line, and the book's derived files went on presenting the
2026-08-27 numbers of the chain archived here. The producer was failing before
it wrote anything: its scheduled task exited non-zero on every run.

What replaces this archive is therefore **`maker/6`**, and it differs from the
announced `/5` in three ways that a reader should know about.

**The perimeter is wider.** Chain `/3`-`/4` published ONE instrument on two
venues, on roughly 430 USD of capital. The current chain publishes a
**multi-instrument** portfolio on the same two venues. The composition itself
is not published — then or now: it is the strategy.

**The capital is not the same capital.** The transfers of 2026-09-09 tripled
the account, and the broker had reset it on 2026-09-02 in the hedging-to-netting
conversion described above.

**The unit price restarts; it does not continue.** This is the one place where
a reader could reasonably expect otherwise, so it is stated plainly. A unit
price is only meaningful against an unbroken custody of the same capital, and
that continuity was broken by the broker's own account reset. The new chain is
anchored to the equity read on both accounts on its inception session
(2026-09-10), frozen in `books/maker_01/inception.json`, and its unit price
starts from there. Nothing bridges the two: a bridge would be an assertion no
record supports.

What DOES continue is the book — same strategy, same two venues, same operator,
same real capital — and the fact that nothing has been deleted to make the new
chain look better than the old one. Both archives remain here, verifiable from
their own genesis.

**The source changed too.** `/3`-`/5` reconstructed the record by querying the
two venues from the publisher and matching round trips there. `/6` takes every
profit-and-loss figure from the trading system's own machine export — one
source of truth — and recomputes none of it. That is why several fields present
in this archive (execution counts per session, unmatched ticket residuals,
out-of-scope calibration impact) are **absent** from `/6` rather than zero: they
came from a matching that no longer exists, and a flat value is not a
measurement of nothing.

### The derived files beside this note

`derives/` holds the regenerated (non-chained) files that presented this
archived chain — its NAV, daily profit and loss, metrics, tear sheet,
methodology and index metadata — moved here rather than overwritten when the
new chain took over the book's directory. They were never hash-chained; the
snapshots are the evidence, and they verify unchanged.
