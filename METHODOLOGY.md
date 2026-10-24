# Methodology

How every published number is produced. The raw inputs are in this repository,
and every figure is published with the convention and the risk-free rate it used,
so any of it can be recomputed independently.

## 1. Where the numbers come from

The desk runs six Alpaca paper accounts, one per book — two of the six are
deliberate $100,000 copies of $1,000,000 siblings — on a fixed daily cycle:
after the close
it computes signals and nets them into an order plan (`stage`); at the next open
it submits that plan (`execute`); after that close it sweeps late fills, marks
positions and snapshots account equity (`mark`); then it archives everything to
parquet with an internal hash chain (`export`). The publisher reads that archive
— never the live database — and writes this repository.

**One book in this record is not that.** `maker_01` trades REAL capital across
two venues and is produced by its own collector rather than by the paper desk.
Where a convention below differs for it, the difference is stated; where this
document says "the desk", it means the six paper accounts.

**Book equity is read from the broker** — for the paper books. Each session's
NAV is Alpaca's `/v2/account` equity, taken at the desk's after-close mark; for
those six books it is not modelled, reconstructed, or derived from our own fill
records. `maker_01`'s NAV is reconstructed from two venues by its collector, and
its own methodology note states how.

## 2. Returns

Daily return is `NAV_t / (NAV_{t-1} + F_t) − 1`, where `F_t` is any **external
capital movement** on that session. Returns are **time-weighted**: money or
assets entering or leaving an account by an act that is not a trade are excluded
from the return and kept in the balance, so performance measures the capital
actually managed. With no flow this reduces exactly to compounding
`NAV_t / NAV_{t-1} − 1`, which is the case for every session of every book
except where a capital event is declared.

**Declared capital events.** A capital event is a movement into or out of an
account that is not a trade: a deposit, a withdrawal, or a broker adjustment,
where the broker adds or removes assets with no transaction, no journal entry,
no corporate action and no offsetting cash. A position removed that way is an
outflow, not a trading loss, and is treated as one. The rule is symmetric: if
the broker later puts the position back, the reversal is declared exactly like
the removal, so a loss is never excluded while the matching recovery is kept.
What remains in the return is only what the account did or did not participate
in.

Four things make this an adjustment rather than an edit, and each is checkable:

- **The raw series is never rewritten.** On the paper books `nav.csv` publishes
  `equity` exactly as the broker reported it, discontinuity included, beside
  `flow`, `adj_factor` and `equity_adj`. Both curves come out of the same file.
  `maker_01` carries the same three columns, filled by its collector's
  **unitisation**: a deposit buys units at the day's price, so it moves the
  balance and never the unit value. There `equity_adj` is the unit value,
  `adj_factor` is `equity_adj / equity`, and `daily_return` is the change in
  `equity_adj`.
- **An event changes its own session and every one after it, and nothing
  before.** For every row published before an event, `adj_factor` is exactly
  `1.0` and `equity_adj` is exactly `equity`. Adjusting the past is not
  something this mechanism can express.
- **Every event is published with its evidence**, inside the write-once,
  hash-chained, timestamped snapshot for the session it hit
  (`capital_event`). It carries the same weight as the numbers it corrects.
- **The amount is reproducible from published files.** It is stated as a
  derivation, not asserted. A position removed without proceeds, for example, is
  valued at its quantity, the net of the fills published under `detail/`, times
  the desk's own mark for the previous session.

A book with no declared event is unaffected in every respect: its multiplier is
`1.0`, its adjusted series is its raw series, and its published metrics are
byte-for-byte what they were.

**The paper books' record starts with the 2026-10-19 session.** It is measured
from each account's value at the desk's after-close mark of 2026-10-16, the
session before, so the first session's profit and loss is inside the record. The
books already held positions at that close: `initial_capital` in `meta.json` is
that value, not the capital the account was funded with, and `record_start` and
`record_base_session` name the two dates. The accounts traded before 2026-10-19;
that period is not published. A book first funded after that date starts at its
funded capital instead: its curve is anchored to the broker's own equity on the
last day before it traded, with the account fully in cash
(`inception_anchored_to_funded_capital: true`).

**This presentation is GIPS-informed and NOT GIPS-compliant.** GIPS compliance
requires third-party verification, which has not been performed. No such claim is
made anywhere.

## 3. Metrics

Every metric is computed by one function — `compute_core_metrics` in the firm's
`rvb.metrics` module — and by nothing else. No metric is recomputed in the
publisher, and none is computed in the browser: the site renders JSON that was
calculated here. That is the only way "our calculation source is public" can be
a fact rather than a claim.

**Sharpe, Sortino and Calmar are excess of the risk-free rate.** Interest on cash
is not alpha. The rate is the 3-month Treasury constant-maturity yield (FRED
`DGS3MO`), averaged over the measured window — the window the ratio covers, not
today's print. The rate actually used is echoed in every payload
(`risk_free_annual`, `risk_free_source`) so any number here can be reproduced
exactly. If FRED is unreachable the payload says `risk_free_source: unavailable`
and the ratio is explicitly gross, rather than quietly pretending the rate was
zero.

**Annualised statistics are published from 60 sessions.** Sharpe, CAGR, Calmar,
volatility, maximum drawdown, VaR, skew, kurtosis and win rate are published once
a book reaches that threshold; until then the payload carries an
`insufficient_history` block with `have` and `need`. Cumulative return, the equity
curve, and the best and worst day are published from the first session.

## 4. Book-level versus per-category

These are not equally hard numbers and are never presented as though they were.

- **Book level is exact.** Broker equity, broker fills.
- **Per-category is an attributed model.** The broker holds one net position per
  symbol, so per-category numbers come from our own order record: every order is
  written with its owners before it is submitted, and a fill goes to the owners of
  the order it filled, never to a strategy that merely shares the symbol. The
  attribution is internally consistent and **approximately** sums to the book — it
  does not close exactly, and the residual is published as its own column in
  `attributed.csv` so you can size the gap rather than take this sentence on
  trust. A different rule would give different numbers from the same fills.
  Anything sourced this way is labelled *attributed*.

### Strategy identity is not published

Holdings, P&L and attribution are grouped by the **category** of the strategy
holding them — mean reversion, momentum, trend following, seasonal — and never by
the individual strategy. The category answers what kind of risk is being taken.
The identity of each strategy, and its logic, are not published.

Two consequences worth stating rather than leaving to be noticed:

- **P&L is a category total, not a per-symbol line.** A per-symbol profit column
  under a named style is the trade record itself: which names the strategy makes
  money on, and when it gets out. The category total answers "which style is
  working" and stops there.
- **Attribution is weighted before it is aggregated.** Each strategy's return is
  measured against its own capital, so summing returns across a category would be
  arithmetic nonsense — a 2% move on a 0.7% sleeve and a 2% move on a 32% sleeve
  are not the same event. Each is weighted by its book weight first, which turns
  it into a *contribution*: additive, and summing to roughly the book's own daily
  return. The individual weights are used for that calculation and are not
  published; the per-category totals are.

## 5. The benchmark

SPY total return (split- and dividend-adjusted), on the same dates, plus a cash
line accrued at the risk-free rate on trading days. `benchmark.csv` and
`benchmark_intraday.csv` publish both the raw level (`spy_close`) and the
rebased series (`spy_cum`).

**`spy_cum` is measured from each book's own first benchmarked session**, so it
answers "what would the index have done over this book's life?" Two books that
opened on different dates therefore quote the same index differently on the same
day, by exactly the index's move between their two opening dates. `spy_close` is
the observed level, carries no base, and is the column to use to compare books
with each other or to rebase against any other anchor.

**The cash line accrues at the rate that had printed by each date**, not at a
mean over the whole window. A window mean would let early sessions earn
interest at a rate set partly by prints that had not happened yet, and would
silently rewrite the published line every time FRED updates.

**These books are not SPY-like.** They carry shorts and multi-asset legs. The
benchmark is context for the question "versus just holding the index?", not a
like-for-like comparison, and it should not be read as one.

## 6. Known biases and limits

**Two broker endpoints disagree slightly.** Alpaca's `/v2/account` equity (our
published NAV) and its `/v2/account/portfolio/history` daily series do not share
a timing basis, so their figures for the same session can differ by several
basis points. Both are broker figures, and neither is adjusted to match the
other. Every snapshot publishes ours, theirs, and the difference in basis points
(`broker_cross_check`), so the spread can be sized session by session.

**Gaps are gaps.** If the box was down, the series has a hole. Nothing is
interpolated across it and no value is carried forward to hide it.

**A symbol the loader cannot price is frozen, not liquidated.** It keeps its last
known state rather than being marked to a guess.

**Broker adjustments are declared, not absorbed.** The desk reconciles the
broker's holdings against its own records. A position the broker adds or removes
with no transaction behind it is declared as a capital event, published with its
evidence, and excluded from the return.

**A broker may restate its own history; this record does not restate itself.**
If the broker later revises its figure for a session already published, the
snapshot for that session is not amended. It records what the broker reported
when it was written, and it is hash-chained and timestamped. Corrections
go forward, as declared events.

## 7. Publication timing

**Everything is published as soon as it is real.** There is no delay on NAV,
returns, metrics or benchmarks, nor on order, fill and position detail.

What replaces a delay is a test, and it is a stricter one. A cycle's detail is
released only once that cycle has **actually executed** — that is, once the desk
has recorded that the orders went to the broker. The desk computes its order plan
after the close for the next open, and a plan can also sit unexecuted for days if
a stage failed; either way it fails the execution test and stays private. A
date-based delay would never have added a guarantee here, only a wait: it is the
execution check, not the calendar, that prevents publishing orders before they
exist at the broker.

As a result, **current holdings are public.** Anyone can see what the portfolios hold, more or less as they hold
it. What they cannot see is what is about to be traded, or which strategy holds
what.
