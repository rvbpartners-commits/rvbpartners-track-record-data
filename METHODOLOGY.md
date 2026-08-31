# Methodology

How every published number is produced. If something here is unclear or looks
wrong, the raw inputs are in this repository and the calculation code is named
below — check it.

## 1. Where the numbers come from

The desk runs four Alpaca paper accounts on a fixed daily cycle: after the close
it computes signals and nets them into an order plan (`stage`); at the next open
it submits that plan (`execute`); after that close it sweeps late fills, marks
positions and snapshots account equity (`mark`); then it archives everything to
parquet with an internal hash chain (`export`). The publisher reads that archive
— never the live database — and writes this repository.

**Book equity is read from the broker.** Each session's NAV is Alpaca's
`/v2/account` equity, taken at the desk's after-close mark. It is not modelled,
reconstructed, or derived from our own fill records.

## 2. Returns

Daily return is `NAV_t / (NAV_{t-1} + F_t) − 1`, where `F_t` is any **external
capital movement** on that session. Returns are **time-weighted**: money or
assets entering or leaving an account by an act that is not a trade are excluded
from the return and kept in the balance, so performance measures the capital
actually managed. With no flow this reduces exactly to compounding
`NAV_t / NAV_{t-1} − 1`, which is the case for every session of every book
except where a capital event is declared.

**Declared capital events.** An earlier version of this document said these
accounts had a single opening deposit and no subsequent external flows. That
stopped being true on 28 August 2026, when the broker removed a position from
one paper account overnight with no sale, no journal entry, no corporate action
and no cash credit; equity fell by exactly the position's market value. That is
an outflow, not a trading loss, and it is treated as one.

Over the following weekend the same broker put the position back, in the
identical quantity and again with no transaction, which on the raw equity is a
+3% jump in a single step outside any session. **The reversal is declared
exactly like the removal.** Excluding a loss and keeping the recovery would be
the cherry-picking this record exists to make impossible, so both directions are
flows and both are published with their evidence. What is left in the return is
only what the account genuinely did or did not participate in: the two
declarations net to -981.64, which is the position's move on the one session it
was not in the account.

Four things make this an adjustment rather than an edit, and each is checkable:

- **The raw series is never rewritten.** `nav.csv` publishes `equity` exactly as
  the broker reported it, discontinuity included, beside `flow`, `adj_factor`
  and `equity_adj`. Both curves come out of the same file.
- **An event changes its own session and every one after it, and nothing
  before.** For every row published before an event, `adj_factor` is exactly
  `1.0` and `equity_adj` is exactly `equity`. Adjusting the past is not
  something this mechanism can express.
- **Every event is published with its evidence**, inside the write-once,
  hash-chained, timestamp-anchored snapshot for the session it hit
  (`capital_event`). It carries the same weight as the numbers it corrects.
- **The amount is reproducible from published files.** It is stated as a
  derivation, not asserted: for the 28 August event, 71.236611 shares (the net
  of six fills published under `detail/`) at the desk's own 422.62 mark for the
  previous session.

A book with no declared event is unaffected in every respect: its multiplier is
`1.0`, its adjusted series is its raw series, and its published metrics are
byte-for-byte what they were.

**The curve starts at funded capital, not at the first mark.** The desk's first
equity snapshot is taken after the first trading day's close, so it already
contains that day's profit and loss. Starting the curve there would silently
delete the first session. Instead each book's series is anchored to the broker's
own equity on the last day *before* it traded — the account fully in cash. When
the broker's history does not reach back that far, `meta.json` says so
(`inception_anchored_to_funded_capital: false`).

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

**Annualised statistics are withheld until 60 sessions.** Sharpe, CAGR, Calmar,
volatility, maximum drawdown, VaR, skew, kurtosis and win rate are suppressed
below that threshold and the payload carries an `insufficient_history` block with
`have` and `need`. On a handful of sessions these are not imprecise estimates,
they are meaningless ones. Cumulative return, the equity curve, and the best and
worst day are published from day one, because those are statements of what
happened rather than estimates of anything.

## 4. Book-level versus per-category

These are not equally hard numbers and are never presented as though they were.

- **Book level is exact.** Broker equity, broker fills.
- **Per-category is an attributed model.** The broker nets our orders, so a
  single net fill is attributed back to the strategies whose write-ahead intents
  contributed to it, pro-rata by requested size; internal opposite intents cross
  at the fill price. The attribution is internally consistent and sums to the
  book, but a different rule would give different numbers from the same fills.
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
line accrued at the risk-free rate on trading days.

**These books are not SPY-like.** They carry shorts and multi-asset legs. The
benchmark is context for the question "versus just holding the index?", not a
like-for-like comparison, and it should not be read as one.

## 6. Known biases and limits

**Two broker endpoints disagree slightly.** Alpaca's `/v2/account` equity (our
published NAV) and its `/v2/account/portfolio/history` daily series do not share
a timing basis; on 2026-08-12 they differed by about 17 basis points for
`best_cagr`. Both are broker figures. Every snapshot publishes ours, theirs, and
the difference in basis points (`broker_cross_check`). Neither is adjusted to
match the other.

**Gaps are gaps.** If the box was down, the series has a hole. Nothing is
interpolated across it and no value is carried forward to hide it.

**A symbol the loader cannot price is frozen, not liquidated.** It keeps its last
known state rather than being marked to a guess.

**The broker's books are not infallible, and we do not paper over it when they
are wrong.** On 28 August 2026 a position vanished from one paper account with
no transaction behind it, and over the following weekend it came back the same
way. The desk's own broker-versus-shadow reconciliation raised both within
hours; both are declared as capital events, published with their evidence, and
excluded from the return. It is disclosed here because a record that only
reports the broker's good days is not a record.

**A broker may restate its own history; this record does not restate itself.**
After putting the position back, Alpaca also revised its own figure for the 28
August close, so its books now behave as though nothing had been missing. The
snapshot published for that session is not amended. It recorded what the broker
reported when it was written, it is hash-chained and timestamp-anchored, and a
record that can be rewritten because the broker changed its mind three days
later is not evidence of anything. Corrections go forward, as declared events.

## 7. Publication timing

**Everything is published as soon as it is real.** There is no delay on NAV,
returns, metrics or benchmarks, and since 13 August 2026 there is no delay on
order, fill and position detail either.

What replaces a delay is a test, and it is a stricter one. A cycle's detail is
released only once that cycle has **actually executed** — that is, once the desk
has recorded that the orders went to the broker. The desk computes its order plan
after the close for the next open, and a plan can also sit unexecuted for days if
a stage failed; either way it fails the execution test and stays private. A
date-based delay would never have added a guarantee here, only a wait: it is the
execution check, not the calendar, that prevents publishing orders before they
exist at the broker.

The consequence is deliberate and is the operator's decision: **current holdings
are public.** Anyone can see what the portfolios hold, more or less as they hold
it. What they cannot see is what is about to be traded, or which strategy holds
what.
