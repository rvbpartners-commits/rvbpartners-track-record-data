# RVB Partners — live track record (data)

The machine-readable record behind [the track-record site](https://github.com/rvbpartners-commits/rvbpartners-track-record-site).
Everything published here is generated on the trading box by an open, auditable
pipeline and pushed unedited. Nothing in this repository is written by hand.

> **Most of this is paper trading; one book is not.** Six books run on Alpaca
> PAPER accounts — no capital is at risk and every fill is simulated by the
> broker's paper engine. One book, `maker_01`, trades **real capital**. Each
> book states its kind beside its return, and every disclosure below records
> which kind it applies to. Past performance is not indicative of future
> results. Nothing here is investment advice, an offer, or a solicitation. The
> full disclosures are in [DISCLOSURE.md](DISCLOSURE.md); each one names the
> books it covers.

## What is here

| Path | What it is |
|---|---|
| `index.json` | Entry point: the books, their inception dates, where everything else lives |
| `CHAIN.jsonl` | Append-only hash chain over every snapshot ever published |
| `books/<book>/meta.json` | Book identity, inception, account reference, disclosure settings |
| `books/<book>/nav.csv` | `date,equity,cash,flow,adj_factor,equity_adj,daily_return` — the whole equity curve, raw and adjusted for declared capital events |
| `books/<book>/metrics.json` | Computed metrics and, until there is enough history, what is withheld |
| `books/<book>/analytics.json` | Tear-sheet series: rolling Sharpe, drawdown path, monthly returns, distribution |
| `books/<book>/benchmark.csv` | `date,spy_close,spy_cum,cash_cum` — SPY total return and the risk-free (cash) line, on our dates. `spy_close` is the raw observed level; `spy_cum` is that level expressed from **this book's own first benchmarked session**, so books with different inception dates quote the same index differently. Rebase `spy_close` yourself for any other anchor |
| `books/<book>/intraday.csv` | `timestamp,session_date,equity` — the 5-minute equity curve the chart is drawn from |
| `books/<book>/benchmark_intraday.csv` | `timestamp,spy_close,spy_cum,cash_cum` — the benchmark on the same instants, same convention as above |
| `books/<book>/attributed.csv` | Per-CATEGORY contribution to the book return — **an attributed model, not broker figures** |
| `books/<book>/snapshots/<date>.json` | One immutable, hash-chained record per session |
| `books/<book>/snapshots/<date>.json.ots` | OpenTimestamps proof for that record |
| `books/<book>/detail/<date>.json` | Order / fill / position detail, grouped by strategy category, published once the cycle has executed |

Seven books are published. Six run on Alpaca **paper** accounts, one account
each; one trades **real capital** — the operator's own money, no third-party
funds. Every book states its kind and its capital beside its return.

| Book | Kind | Selected for |
|---|---|---|
| `best_cagr` | paper | Compound growth |
| `best_sharpe` | paper | Risk-adjusted return |
| `best_mdd` | paper | Limiting drawdown |
| `best_composite` | paper | A balance of all three |
| `best_cagr_100k` | paper | **The same strategies and weights as `best_cagr`, at one tenth the capital** |
| `best_composite_100k` | paper | **The same strategies and weights as `best_composite`, at one tenth the capital** |
| `maker_01` | **real capital** | Delta-hedged liquidity provision across two venues |

The two `_100k` books are **not independent results**. They are deliberate
copies of their siblings run at $100,000 instead of $1,000,000, so that any
divergence between a pair measures the effect of account size on execution —
rounding, minimum sizes, impact — and nothing else. Read each pair as one
experiment, not as two records.

## How to check it yourself

See [VERIFY.md](VERIFY.md). In short: every snapshot hashes its own content,
each one carries the hash of the previous session, `CHAIN.jsonl` records both,
and each has an OpenTimestamps proof that it existed when it says it did. A
clone plus the verifier reproduces every published number from `nav.csv`.

## What is deliberately not here

- **Strategy identity or logic.** Holdings, P&L and attribution are grouped by
  strategy CATEGORY — mean reversion, momentum, trend following, seasonal. No
  strategy is named anywhere in this repository, and P&L is reported per category
  rather than per symbol. The metric calculation code is open (see
  METHODOLOGY.md); the strategies are not.
- **Anything forward-looking.** The desk builds its order plan after the close
  for the next open. That plan is never published — detail is released only once
  its cycle has actually executed, which is a stricter test than any delay.
- **Credentials.** No API key, token or secret has ever been in this repository,
  and the publisher refuses to write a payload that looks like one.

## Provenance

Published by the RVB Partners desk on its own hardware. GitHub Actions is not
involved in producing this data and holds no broker credential.
