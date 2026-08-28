# RVB Partners — live track record (data)

The machine-readable record behind [the track-record site](https://github.com/rvbpartners-commits/rvbpartners-track-record-site).
Everything published here is generated on the trading box by an open, auditable
pipeline and pushed unedited. Nothing in this repository is written by hand.

> **These are Alpaca PAPER accounts.** No capital is at risk and every fill is
> simulated by the broker's paper engine. Past performance is not indicative of
> future results. Nothing here is investment advice, an offer, or a solicitation.
> The full disclosures are in [DISCLOSURE.md](DISCLOSURE.md) and are stamped into
> every record in this repository.

## What is here

| Path | What it is |
|---|---|
| `index.json` | Entry point: the books, their inception dates, where everything else lives |
| `CHAIN.jsonl` | Append-only hash chain over every snapshot ever published |
| `books/<book>/meta.json` | Book identity, inception, account reference, disclosure settings |
| `books/<book>/nav.csv` | `date,equity,cash,daily_return` — the whole equity curve |
| `books/<book>/metrics.json` | Computed metrics and, until there is enough history, what is withheld |
| `books/<book>/analytics.json` | Tear-sheet series: rolling Sharpe, drawdown path, monthly returns, distribution |
| `books/<book>/benchmark.csv` | SPY total return and the risk-free (cash) line, on our dates |
| `books/<book>/attributed.csv` | Per-CATEGORY contribution to the book return — **an attributed model, not broker figures** |
| `books/<book>/snapshots/<date>.json` | One immutable, hash-chained record per session |
| `books/<book>/snapshots/<date>.json.ots` | OpenTimestamps proof for that record |
| `books/<book>/detail/<date>.json` | Order / fill / position detail, grouped by strategy category, published once the cycle has executed |

Four books run, one Alpaca paper account each. They are separate portfolios
selected on different objectives, not four views of one portfolio.

| Book | Selected for |
|---|---|
| `best_cagr` | Compound growth |
| `best_sharpe` | Risk-adjusted return |
| `best_mdd` | Limiting drawdown |
| `best_composite` | A balance of all three |

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
