# Disclosures

The terms on which the books in this repository should be read. Each item
states which books it applies to: some books trade on paper and one trades
real capital, and an item true of one kind is not necessarily true of the
other.

Each record also carries a disclosure block for its own book.

## Paper accounts

*Critical* · applies to the paper books

The paper portfolios run on Alpaca paper accounts. No capital is invested in them: the desk sends real orders, and the broker fills them from its paper engine against its market data. They record how the strategies execute live; they are not a record of managing client money.

## Real capital

*Critical* · applies to the real-capital book

One portfolio trades the firm's own capital, with fills executed on its venues. The firm manages no third-party money. The capital is published beside the return, since a return on a smaller account does not scale directly to a larger one: costs do not scale with size.

## Past performance

*Critical* · applies to every book

Nothing on this site is investment advice, an offer, or a solicitation to buy or sell any financial instrument. Past performance, simulated or otherwise, is not indicative of future results.

## Per-strategy figures

*Important* · applies to every book

Account-level equity and returns are read from the broker's account. Per-strategy figures are modelled: the broker nets the desk's orders, so each net fill is attributed back to the strategies that contributed to it, pro-rata by requested size. A different attribution rule would give different per-strategy figures from the same fills.

## Annualised statistics

*Important* · applies to every book

The Sharpe ratio, annual return, Calmar ratio, volatility and maximum drawdown are published once a portfolio has 60 marked sessions. Cumulative return and the equity curve are published from the first session.

## Publication timing

*Note* · applies to every book

Net asset value, returns and metrics are published without delay. Order, fill and position detail is published once the cycle that produced it has executed. The order plan the desk prepares after the close is published only once it has been sent, so current holdings are public.

## Capital movements

*Important* · applies to every book

Deposits, withdrawals and broker adjustments are excluded from the return and kept in the balance, the standard time-weighted treatment. Raw broker equity is published unchanged in nav.csv beside the flow, the adjustment factor and the adjusted series, and each declared event is published with its evidence in the snapshot for the session it affected.

## Strategy categories

*Note* · applies to every book

Positions and attribution are grouped by strategy category, such as mean reversion, momentum, trend following or seasonal, rather than by individual strategy. The identity and logic of each strategy are not published.

## GIPS

*Note* · applies to every book

Returns are time-weighted. The presentation is informed by GIPS practice but is not GIPS-compliant; compliance requires third-party verification, which has not been performed.
