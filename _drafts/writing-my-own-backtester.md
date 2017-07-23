---
layout: post
title:  "Writing My Own Backtester"
date:   2017-07-16 13:34:00 +0200
last_modified_at: 2017-07-06 16:27:00 +0200
feature_image: "https://unsplash.it/1300/400?image=1057"
category: Trading
tags: [stock, trading, trading automation, zorro, backtesting]
---

I decided to write my own backtesting and live trading system for Interactive
Brokers to replace Zorro. With Zorro I was having serious issues while live
trading, I noticed that trade execution is wrong. I stumbled over too many bugs,
so I don't trust Zorro any longer, that is why it needs to be replaced.

<!-- more -->

## Getting Historical Data

I decided to write a very simple backtester which can only handle daily EOD
data. To be able to backtest my strategy I needed historical data. There
are some free sources of historical data online.

In the past one could download EOD data from **Yahoo Finance**. They changed
their service recently and now it is a bit harder to get the data but it is
still possible. There is the
[YahooFinanceAPI](https://github.com/dennislwy/YahooFinanceAPI) on github
writen in C# to download stock quotes.

Another free source of EOD data is **Google Finance**. An example url to query data is
[http://www.google.com/finance/historical?q=NASDAQ%3aADBE&startdate=Jan+01%2C+2009&enddate=Aug+2%2C+2012&output=csv](http://www.google.com/finance/historical?q=NASDAQ%3aADBE&startdate=Jan+01%2C+2009&enddate=Aug+2%2C+2012&output=csv).

There is also [**Quandl**](https://www.quandl.com/) and [**Alpha
Vantage**](https://www.alphavantage.co/) which provide an API to download free
daily EOD data. Both require an API key.

## Trade Engine Core

When developing a trading strategy, I want to be able to backtest it and then
trade the same strategy live without modification.

I created an abstract TradeEngine class that can execute a trading algorithm and
the core code is the same for backtesting and live trading. In backtesting mode
a `Backtester` class calls the `Tick` method repeatedly to simulate the next
trading day while in live trading the `Tick` method is run once per day.

Using this technique I can verify and debug the core engine with historical data
and then expect that live trading also works just fine. The only difference will
be the data. Trade entry and exit and all other things will be the same as in
the simulation.

```csharp
public class TradeEngine : ITradeApi
{
  public IStrategy Strategy { get; private set; }

  public Dictionary<string, IAsset> Assets { get; private set; } = new Dictionary<string, IAsset>();
  public TradeManager TradeManager { get; private set; }

  public TradeEngine(IStrategy strategy, TradeManager tradeManager)
  {
    Strategy = strategy;
    TradeManager = tradeManager;
  }

  public void Initialize()
  {
    Strategy.Initialize();
    TradeManager.SetInitialBalance(Capital);
  }

  public void Tick(DateTime now)
  {
    if (now.DayOfWeek == DayOfWeek.Saturday || now.DayOfWeek == DayOfWeek.Sunday)
      return;

    Now = now;

    var tradeData = new TradeData();
    foreach (var kvp in Assets)
    {
      var asset = kvp.Value;
      asset.Tick(now);
      if (asset.IsHistoryReady && asset.IsMarketOpen)
        tradeData.History[asset.Symbol] = asset.History;
    }

    TradeManager.Tick(now);
    Strategy.HandleData(tradeData);
  }

  public void Finish()
  {
    Strategy.Finish();
  }
}
```

## Drawing Equity Curve

I also needed some visual representation of the equity and drawdown curves. For
this I used the [ZedGraph](https://github.com/ZedGraph/ZedGraph) library. With
ZedGraph it was quite easy to draw some plots.

![Screenshot]({{ site.url }}/assets/images/trading/trade.png)

After writing approximately 2000 Lines of code I have a functioning backtester
and a few strategies. The above equity curve is from my Bollinger Band mean
reversion strategy.