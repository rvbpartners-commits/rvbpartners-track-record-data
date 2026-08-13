# RVB Partners — live track record

Daily performance data for four RVB Partners portfolios, published automatically
from the trading desk.

> **Alpaca paper accounts.** No capital is at risk and fills are simulated. Past
> performance is not indicative of future results. Nothing here is investment
> advice. See [DISCLOSURE.md](DISCLOSURE.md).

Rendered at **[trackrecord.rvbpartners.fr](https://trackrecord.rvbpartners.fr)**.

## Contents

| Path | |
|---|---|
| `index.json` | Entry point: portfolios and where their files are |
| `CHAIN.jsonl` | Hash chain over every snapshot published |
| `books/<book>/nav.csv` | Daily equity and returns |
| `books/<book>/intraday.csv` | Equity at 5-minute resolution |
| `books/<book>/metrics.json` | Computed metrics |
| `books/<book>/analytics.json` | Rolling and distributional series |
| `books/<book>/benchmark*.csv` | S&P 500 total return and cash |
| `books/<book>/snapshots/` | One immutable record per session, with an OpenTimestamps proof |
| `books/<book>/detail/` | Orders, fills and positions by strategy category |

## Verifying

Each snapshot hashes its own content, carries the previous session's hash, and is
timestamped. See [VERIFY.md](VERIFY.md) for the four checks and the code to run
them. Calculation method: [METHODOLOGY.md](METHODOLOGY.md).

## Not published

Strategy names or logic, and any order that has not yet been sent to the broker.
Holdings are grouped by category. No credential has ever been in this repository.

## Contact

Technical questions: open an issue, or [contact@rvbpartners.fr](mailto:contact@rvbpartners.fr).
