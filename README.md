# RVB Partners — live track record (data)

The machine-readable record behind [the track-record site](https://github.com/rvbpartners-commits/rvbpartners-track-record-site).
Everything published here is generated on the trading box by the firm's publishing
pipeline and pushed unedited. Nothing in this repository is written by hand.

> **Six books trade on paper; one traded real capital.** Six books run on Alpaca
> paper accounts — no capital is at risk in them, and every fill is simulated by the
> broker's paper engine — and publish here from the 19 October 2026 session.
> `maker_01` traded real capital, the firm's own, until it closed on 16 October 2026;
> its record ends at that session. Each book states its kind beside its return, and
> every disclosure records which kind it applies to. Past performance is not
> indicative of future results. Nothing here is investment advice, an offer, or a
> solicitation. The full disclosures are in [DISCLOSURE.md](DISCLOSURE.md); each one
> names the books it covers.

## What is here

| Path | What it is |
|---|---|
| `index.json` | Entry point: the books, their inception and record-start dates, where everything else lives |
| `research.json` | Aggregate statistics of the firm's strategy research — catalogue counts, search and deflation figures — with no strategy named; copied on each run from the firm's committed summary, outside the hash chains |
| `CHAIN.jsonl` | Append-only hash chain over every snapshot in the live chains. `maker_01`'s earlier chains are kept verbatim under `books/maker_01/superseded/`, each with its own `CHAIN.jsonl` |
| `books/<book>/meta.json` | Book identity, inception, record start, account reference, disclosure settings |
| `books/<book>/nav.csv` | `date,equity,cash,flow,adj_factor,equity_adj,daily_return` — the whole equity curve, raw and adjusted for declared capital events |
| `books/<book>/metrics.json` | Computed metrics and, until there is enough history, what is withheld |
| `books/<book>/analytics.json` | Tear-sheet series: rolling Sharpe, drawdown path, monthly returns, distribution |
| `books/<book>/benchmark.csv` | `date,spy_close,spy_cum,cash_cum` — SPY total return and the risk-free (cash) line, on our dates. `spy_close` is SPY's level as fetched, split- and dividend-adjusted; `spy_cum` is that level expressed from this book's own first benchmarked session, so books with different inception dates quote the same index differently. Rebase `spy_close` yourself for any other anchor |
| `books/<book>/intraday.csv` | `timestamp,session_date,equity` — the 5-minute equity curve the chart is drawn from |
| `books/<book>/benchmark_intraday.csv` | `timestamp,spy_close,spy_cum,cash_cum` — the benchmark on the same instants, same convention as above |
| `books/<book>/attributed.csv` | `date,category,contribution,book_return,residual` — per-category contribution to the book return: an attributed model, not broker figures. `residual` is what the model leaves unexplained |
| `books/<book>/snapshots/<date>.json` | One immutable, hash-chained record per session. Each paper book's chain opens with a base record dated 2026-10-16: the close its record is measured from, with `sessions` 0 and no return |
| `DETAIL_CHAIN.jsonl` | Append-only chain over the correctable artefacts — execution detail and the 5-minute series. Separate from `CHAIN.jsonl` because those files legitimately gain new versions; every version appends its own entry |
| `books/<book>/snapshots/<date>.json.ots` | OpenTimestamps proof for that record |
| `books/<book>/detail/<date>.json` | Execution detail, published once the cycle has executed: the book's orders and fills, per symbol, and its open positions — symbol, quantity, average price, mark and cost basis — grouped by strategy category. A file is keyed by the session that prepared the cycle, whose orders went out at the next open: `detail/2026-10-16.json` holds the 2026-10-19 open |

The table above describes the paper books. `maker_01` publishes a different set: it
has no `attributed.csv` and no `detail/` — its composition is the strategy. Its
`nav.csv` carries the same columns as the paper books, filled by its collector's
unitisation: `equity_adj` is the unit value, which moves only with profit and loss,
`adj_factor` is `equity_adj / equity`, and `flow` is the capital movement on that
date. `cash` is left empty, because its cash sat on two venues in two currencies. Its
`intraday.csv` has one reading per round trip rather than every five minutes, and its
benchmark files carry the cash line only (`spy_cum` empty, no `spy_close`). It adds
`daily.csv`, its own `METHODOLOGY.md`, `events.json`, and the four frozen inputs its
records are recomputed from: `inception.json`, `fx_rates.json`, `venue_a_pnl.json`
and `venue_b_pnl.json`. Read that book's own note before comparing it with the
others.

Seven books are published here. Six run on Alpaca paper accounts, one account each,
and appear from the 19 October 2026 session. `maker_01` traded real capital, the
firm's own — no third-party money is managed — and closed on 16 October 2026. Every
book states its kind and its capital beside its return.

| Book | Kind | Selected for |
|---|---|---|
| `best_cagr` | paper | Compound growth |
| `best_sharpe` | paper | Risk-adjusted return |
| `best_mdd` | paper | Limiting drawdown |
| `best_composite` | paper | A balance of all three |
| `best_cagr_100k` | paper | The same strategies and weights as `best_cagr`, at one tenth the capital |
| `best_composite_100k` | paper | The same strategies and weights as `best_composite`, at one tenth the capital |
| `maker_01` | real capital, closed 16 October 2026 | Delta-hedged liquidity provision across two venues |

The two `_100k` books are not independent results. They are deliberate copies of
their siblings run at $100,000 instead of $1,000,000, so that any divergence between
a pair measures the effect of account size on execution — rounding, minimum sizes,
impact — and nothing else. Read each pair as one experiment, not as two records.

## How to check it yourself

See [VERIFY.md](VERIFY.md). In short: every snapshot hashes its own content, each one
carries the hash of the previous session, `CHAIN.jsonl` records both, and each has an
OpenTimestamps proof that it existed no later than its Bitcoin block. A clone is all
it takes to check the chain and the proofs, and every published figure comes with its
inputs, its convention and its rate, so it can be recomputed independently.

## What is deliberately not here

- **Strategy identity or logic.** No strategy is named anywhere in this repository:
  holdings and attribution are grouped by strategy category — mean reversion,
  momentum, trend following, seasonal. Each holding is published with its symbol,
  quantity, average price and the session's mark, so its open P&L follows from its
  own row, and the book's fills are published per session. Where a category holds a
  single strategy, its weight and attribution are that strategy's. Metric
  definitions and conventions are set out in METHODOLOGY.md; the strategies are not
  published.
- **Anything forward-looking.** The desk builds its order plan after the close for
  the next open. That plan is never published — detail is released only once its
  cycle has actually executed, which is a stricter test than any delay.
- **Credentials.** No API key, token or secret has ever been in this repository, and
  the publisher refuses to write a payload that looks like one.

## Provenance

Published by the RVB Partners desk on its own hardware. GitHub Actions is not
involved in producing this data and holds no broker credential.
