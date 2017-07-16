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
trading, I noticed that trade execution is wrong. With Zorro I stumbled over too
many bugs, so I don't trust the software any longer. I regret that I bought the
sponsored Edition.

<!-- more -->

## Getting Data

I decided to write a very simple backtester which can only handle daily EOD
Data. To be able to backtest my strategy first I needed historical data. There
are some free sources of historical data online.

In the past once could download EOD data from **Yahoo Finance**. They changed
their service recently and now it is a bit harder to get the data bit it is
still possible. There is a github repository which provides C# code to download
stock quotes [here](https://github.com/dennislwy/YahooFinanceAPI).

Another free source of EOD data is **Google Finance**. An example url to query data is
[http://www.google.com/finance/historical?q=NASDAQ%3aADBE&startdate=Jan+01%2C+2009&enddate=Aug+2%2C+2012&output=csv](http://www.google.com/finance/historical?q=NASDAQ%3aADBE&startdate=Jan+01%2C+2009&enddate=Aug+2%2C+2012&output=csv).

There is also [**Quandl**](https://www.quandl.com/) and [**Alpha
Vantage**](https://www.alphavantage.co/) which provide free daily data.
Both require an API key.