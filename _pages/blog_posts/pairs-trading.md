---
layout: single
title: "Pairs Trading"
permalink: /blog_posts/pairs-trading/
mathjax: true
---

## Pair Finding for Statistical Arbitrage Strategies

This post assumes some understanding of pairs trading and statistical arbitrage strategies. If you're not familiar, feel free to check out my introductory post on [pairs trading](/blog_posts/statarb-intro/). Otherwise, the brief recap below may be sufficient.

### Brief Recap of Cointegration

A typical approach to learning about pairs trading is to find pairs for which a cointegrated time series can be obtained from the linear combination of two $$I(1)$$ time series. In other words, we seek an $$I(0)$$ process $$\epsilon(t)$$ such that:

$$\epsilon(t, p_1(t), p_2(t)) = \alpha p_1(t) + \beta p_2(t)$$

where $$p_1(t)$$ and $$p_2(t)$$ are two prices series associated with the underlying assets being traded. Without loss of generality (other than a constant scaling of $$\epsilon$$), we can write this equation as:

$$\epsilon(t, p_1(t), p_2(t)) = p_1(t) + B p_2(t)$$

This is is a nice, intuitive result. If $$\epsilon$$ is $$I(0)$$, then all we need to do is use the hedge ratio ($$B$$) to purchase particular combinations of the two assets such that when $$\epsilon$$ is far from the mean around which it is stationary, we enter into a position which profits from a mean reversion.

A number of practical problems arise when one goes from learning this theoretical knowledge to attempting an implementation of it for their own model development.

1. The same signals for entries can indicate a breaking of the cointegration relationship. Given that statarb strategies are based on these relatively small profits aggregated over a large number of trades (and pairs), the tail risk associated with these decoupling events *must* be avoided.

2. The optimal hedge ratio is not constant. This introduces an optimization problem where one must balance the practicality (and costs) of constant portfolio rebalancing against the sensitivity of the cointegration relationship to the hedge ratio. In other words, if you have a function characterizing the stationariy of your cointegration $$f(\epsilon)$$, then you really should care a lot about $$\frac{\partial f}{\partial B}$$.

3. Finding a sufficiently large set of pairs to trade is, in and of itself, not straightforward.

Points number 1, and 2 will be addressed in another set of posts, but for now we focus on point 3.

### How to actually find pairs in practice?

When learning about statistical arbitrage, one is often presented with an example of a pair which is cointegrated, and one which is not.
In reality one will find that truly cointegrated pairs are rare.
What then is a straightforward strategy for finding pairs which are cointegrated? This is a bit of a tedious process, and it's the thing that initially caused me to develop my Starbie package.

![Starbie Package](/images/starbie.png){: width="50%"}

One of the most useful features of starbie is a set of tools for finding pairs (or baskets) of financial instruments which can be cointegrated. A basic approach to do this for pairs is the following:

1. Take a collection of financial instruments and their price data.
2. Construct the $$ nAsset \choose 2$$ unique pairs.
3. Run a cointegration test (such as Engle-Granger) on each pair.
4. Grab only those which meet your specifiec cointegration thresholds.
5. Optionally run some additional filtering for risk mitigation.

As always, informing your initial selection of instruments with a good understanding of the market will make things much easier. Resist the urge to look for a 100% hands off approach. A little bit of market intution and sound reasoning can go a long way.

Let's look at how to do this very quickly and easily with Python and starbie.
-------
### 1. Take a collection of assets and their price data.

Here I use the price data as a numpy array with shape `(nAssets, nTimesteps)`. Grab their corresponding dates in an object with shape `nTimesteps`. Then grab the corresponding tickers in an object with shape `nAssets`. Here I've done this for some carefully selected tech stocks, and their daily price data from 2020-2025.

```python
daily: np.ndarray (nAssets, nTimesteps)
date: np.ndarray (nTimesteps)
tickers: List[str] (nAssets)
```

### 2. Construct the $$ nAsset \choose 2$$ unique pairs.

```python
from itertools import combinations
tickerPairs = list(itertools.combinations(tickers, 2))
```

This code snippet is the equivalent of doing:

```python
tickerPairs = []
for i in range(nAssets):
    for j in range(i+1, nAssets):
        tickerPairs.append((tickers[i], tickers[j]))
```

### 3/4. Run a cointegration test (such as Engle-Granger) on each pair. Grab only those which meet your specifiec cointegration thresholds.

If you implement this yourself, all you need to do is

1. Pick a method for carrying out the cointegration test.
2. Implement it.
3. Decide what thresholds work for you in terms of the p-value and test statistic that comes from your stationarity test (like ADF or KPSS).



```python
from starbie.statarb import pair_finder
pValues, testStats, cointTickers, coeffs = pair_finder(daily, dates, tickers)
```

This is handled by starbie where I have a number of different ways to carry out this test for cointegrated pairs (as well as some very fast low level implementations that I use when searching for triples where the combinatorial problem becomes much more problematic.)

The output is a series of p-values, test statistics, the paired tickers, and the coefficients in the cointegration relationship.

### 5. Optionally run some additional filtering for risk mitigation.

This will vary wildly on what you're trading, the time frame, and your access to deeper market data, fundamental data, or alternative data. You need to be creative in this regard. Mitigating tail risk is everything in statarb, and there are a lot of ways to do it. 

Simply increasing the number of pairs you trade and minimizing your exposure to any given symbol is the poor man's risk mitigation. Of course, one should go further than just this. Starbie does this as a default by only returning disjoint pairs, e.g. generating pairs where a given instrument appears only once across the entire portfolio. This is necessary because often one instrument will have some strange event which causes it to be cointegrated with many of the others when it trends against the broader market.

Additionally, cointegration is nice, but of course finding a pair and trading it is not particularly sophisticated or persistent over time. Don't overlook even simple metrics like a rolling correlation over the log returns, or using the Hurst exponent to characterize market regimes. I won't go into it here, but mitigating tail risk is what separates the people who consistently have high Sharpe statarb strategies from those who trade one or two pairs and are disappointed by the results after a large drawdown.

Here are some relevant papers on the topic which I found interesting:

[Modeling risk in arbitrage strategies using finite mixtures](https://www.tandfonline.com/doi/full/10.1080/14697680802595635)

[What is Statistical Arbitrage?](https://www.scirp.org/html/12-1501441_83611.htm)

Section 2.3 on [Essays in Staistical Arbitrage](https://eprints.soton.ac.uk/366275/)

A future post may be dedicated solely to risk mitigation in statarb strategies. In the meanwhile, here's a quick bonus in terms of visualization.

### Bonus: Visualizing the cointegration relationships

A first glance I like to use is simply plotting the cointegrated time series against some key levels for entries/exits. Typically I will include the entry/exit thresholds that I like to use, which can be as simple as a $$2\sigma$$ entry and a $$1\sigma$$ exit. 

Never underestimate the power of really spending some time looking at and trying to draw insights from your data. The temptation exists to have everything automated, but I think it's incredibly important to spend some time really understanding your data manually, by hand when possible.

Just as an example, this can improve your intuition for hyperparameter selection in any sort of model development. If you visualize the cointegrated pair, and you see that oscillations happen on the order of 100 time steps, this may inform your selection for the number of lags to use in a Hurst exponent calculation.

Anyways, starbie makes these visualizations very easy. We take the pairs it found before, and visualize the cointegrated time series against some $$\sigma$$ levels.

```python
from starbie.statarb import visualize_found_pairs
visualize_found_pairs(pValues, testStats, cointTickers, coeffs, daily, dates, tickers)
```

This results in the following plot for a pair_find over some tech stocks. The black line is the cointegrated time series, blue lines are the $$\sigma$$ level, and the red lines are the $$2\sigma$$ levels.

![Visualization](/images/cointExampleStarbie.png){: width="100%"}

None of these were particularly my taste, but feel free to check them out for yourself. Thanks for reading, and feel free to look through my other posts.



