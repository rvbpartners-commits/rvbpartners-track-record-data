# Disclosures

Every one of these is stamped into every record in this repository, not just
rendered on the site. They are specific limitations of this track record, in
descending order of how much they should change how you read it.

## Paper accounts — no capital at risk

*CRITICAL*

These results come from Alpaca PAPER accounts. No real money is invested, no capital is at risk, and every fill is simulated by the broker's paper engine against its market data. They are a live-execution rehearsal of the strategies, not a record of managing client money.

## Past performance is not indicative of future results

*CRITICAL*

Nothing on this site is investment advice, an offer, or a solicitation to buy or sell any financial instrument. Past performance — simulated or otherwise — is not indicative of future results.

## Per-strategy figures are an attributed model

*IMPORTANT*

Book-level equity and returns are EXACT: they are read from the broker's own account endpoint. Per-strategy returns are not. The broker nets our orders, so each net fill is attributed back to the strategies whose write-ahead intents contributed to it, pro-rata by requested size. That attribution is a model. It is internally consistent and it sums to the book, but a different attribution rule would produce different per-strategy numbers from the same fills.

## Annualised statistics are withheld until there is enough history

*IMPORTANT*

Sharpe, CAGR, Calmar, volatility and maximum drawdown are suppressed until a book has at least 60 marked sessions. On a handful of sessions they are not imprecise, they are meaningless. Cumulative return and the equity curve are shown from the first day, because those are statements of what happened.

## Everything is published as soon as it is real

*NOTE*

NAV, returns and metrics are published with no delay. Order, fill and position detail is published as soon as the cycle that produced it has ACTUALLY EXECUTED — there is no additional waiting period. What is never published is the pending order plan: the desk computes its orders after the close for the next open, and that plan stays private until it has been sent. The execution test, not a delay, is what prevents publishing orders before they exist at the broker. The consequence is deliberate: current holdings are public.

## External capital movements are excluded from the return

*IMPORTANT*

A capital movement that is not a trade does not belong in a performance figure. When money or assets enter or leave an account by an act that is not the manager's, the return is measured over the capital actually managed and the movement is kept in the balance. This is the standard time-weighted treatment of a deposit, a withdrawal, or a broker adjustment. Nothing is hidden and nothing is rewritten: the raw broker equity is published unchanged in nav.csv beside the flow, the multiplier and the adjusted index, so both curves can be drawn from the same file. Every declared event is published with its evidence inside the write-once, hash-chained, timestamp-anchored snapshot for the session it hit. A book with no declared event is unaffected: its multiplier is exactly 1.0 and its adjusted series is its raw series.

## Holdings are shown by strategy category, not by strategy

*NOTE*

Positions and attribution are grouped by the STYLE of the strategy that holds them — mean reversion, momentum, trend following, seasonal — and never by the individual strategy. The category answers what kind of risk is being taken; the identity of each strategy, and its logic, are not published. Per-position profit and loss is reported at the category level for the same reason: a per-symbol P&L line under a named style is the trade record itself.

## GIPS-informed, not GIPS-compliant

*NOTE*

Returns are time-weighted: an external capital flow is excluded from the return and kept in the balance, so performance measures the capital actually managed. With no flow this reduces to simple compounding of daily NAV, which is the case for every book except where a capital event is declared beside the curve. The presentation is informed by GIPS practice but makes NO claim of GIPS compliance: that requires third-party verification, which has not been performed.
