# Methodology

## Source

Four Alpaca paper accounts trade on a fixed daily cycle: signals and a netted
order plan after the close, execution at the next open, marks and an equity
snapshot after that close, then an archive. A separate publisher reads that
archive — never the live database — and writes this repository.

**Equity is read from the broker.** Each session's NAV is Alpaca's `/v2/account`
equity at the after-close mark. The curve is also published at 5-minute
resolution from the broker's portfolio history: real readings, not interpolation.

## Returns

Daily return is `NAV_t / NAV_{t-1} − 1`, time-weighted. With a single opening
deposit and no later cash flows this is simple compounding.

**The curve starts at funded capital.** The first equity snapshot is taken after
the first trading day's close and already contains that day's P&L, so each
portfolio is anchored to the broker's equity on the last day before it traded.
That also makes the funded capital the first high-water mark, so a loss in the
opening session registers as a drawdown instead of disappearing.

GIPS-informed, **not GIPS-compliant** — that requires third-party verification,
which has not been performed.

## Metrics

Every metric is computed by `compute_core_metrics` in the firm's `rvb.metrics`
module and by nothing else — not in the publisher, not in the browser.

**Sharpe, Sortino and Calmar are excess of the risk-free rate** (FRED `DGS3MO`,
averaged over the window the ratio covers). The rate used is published beside
every number. If FRED is unreachable the payload says so and the ratio is
explicitly gross.

**Annualised statistics are withheld below 60 sessions** — Sharpe, CAGR, Calmar,
volatility, drawdown, VaR, skew, kurtosis, win rate. On a handful of sessions
they are not imprecise, they are meaningless. Cumulative return, the equity curve
and the daily bars are published from day one.

## Attribution and strategy identity

Account-level equity and returns are exact: broker figures. Per-category numbers
are a **model** — the broker nets our orders, so each net fill is attributed back
to the strategies that asked for it, pro-rata by size.

Holdings, P&L and attribution are grouped by strategy **category** (mean
reversion, momentum, trend following, seasonal). Individual strategies are not
named, and P&L is a category total rather than a per-symbol line. Attribution is
weighted by book weight before aggregating, because each strategy's return is
measured against its own capital and returns across a category do not add.

## Benchmark

S&P 500 (SPY) total return on the same instants, plus a cash line accrued at the
risk-free rate. The cash line is a formula; every other series is an observation.

These portfolios carry short positions and are not index-like. The benchmark is
context, not a like-for-like comparison.

## Known limits

**Limit-order fills are biased low, so these results understate.** The desk uses
Alpaca's free IEX feed, roughly 2–3% of consolidated US equity volume, so the
range the paper fill engine sees is narrower than the real tape. Measured over
the first sessions: market orders filled 100% (72/72), limit orders 6% (7/120),
and of 18 expired limit orders none had touched its IEX bar. The direction is
certain; the magnitude has not been measured and is not stated here.

**Two broker endpoints differ slightly.** Alpaca's account equity (the published
NAV) and its portfolio-history series do not share a timing basis — about 17
basis points on 2026-08-12. Both are broker figures; every snapshot publishes
ours, theirs and the difference.

**Gaps are gaps.** Nothing is interpolated across a missing session and no value
is carried forward. An intraday session whose broker feed contradicts the
published NAV is dropped whole and reported rather than smoothed.

## Timing

NAV, returns, metrics and benchmarks are published with no delay. Order, fill and
position detail is published once its cycle has **actually executed** — the
pending order plan is never published. Current holdings are therefore public;
what is about to be traded is not.
