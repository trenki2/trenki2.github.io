---
layout: post
title:  "Trading with Zorro"
date:   2017-06-04 15:11:00 +0200
last_modified_at: 2017-06-10 10:47:00 +0200
feature_image: "https://unsplash.it/1300/400?image=744"
categories: [Trading]
tags: [stock, trading, trading automation, zorro]
excerpt: |
  Zorro is like the swiss army knife for trading. The Lite-C scripting 
  language is easy to use and allows you to express your trading idea with 
  very little code.
---
A while go I developed some interest in trading and did some research to find
out which software I might use for automated forex or stock trading.

There is commercial software like TradeStation, MultiCharts, NinjaTrader,
AmiBroker, Neuroshell etc. There is also the free MetaTrader but it only works
with Forex brokers.

And then there is [Zorro](http://zorro-trader.com/). Zorro is free but if you
want to trade with a sizeable amount of money you are required to buy the
sponsored Zorro S edition.

## The Good

Zorro is like the swiss army knife for trading. The Lite-C scripting language is
easy to use and allows you to express your trading idea with very little code.
I prefer it over the other commercial alternatives.

If you want to risk your money you can do random trading very easily like this:

{% highlight c %}
function run()
{
  BarPeriod = 1440;

  if (random() < 0)
    reverseLong(1);
  else
    reverseShort(1);
}
{% endhighlight %}

Now you could hit trade, connect to your broker and wait.

## The Bad

In this last year where I used Zorro i stumbled over a multitude of bugs which
was pretty annoying and always gave me a bad feeling. I'm myself a software
developer and know it is not easy to make software bug free but especially for a
trading software that manages my money I would have expected a higher quality
standard. I need to be able to trust the software!

Some bugs could be resolved in cooperation with the Zorro Support Team but it
seems there are still some left. With my newest trading strategy I noticed trade
execution is not correct so I missed out on profit.

Currently I am still using Zorro but I plan to replace it with my own software
when I have the time.

## The Strategy

A simple strategy taken from one of the books in the references trades the S&P
500 index based on the VIX COBE volatility index.

{% highlight cpp %}
function run()
{
  BarPeriod = 1440;
  StartDate = 2000;
  Confidence = 100;

  PlotWidth = 700;
  PlotHeight1 = 400;

  Capital = 10000;
  var InitialMargin = Margin = Capital * 0.05;

  assetList("AssetsIndexYahoo.csv");

  assetHistory("^VIX", FROM_YAHOO);
  assetHistory("^GSPC", FROM_YAHOO);

  asset("^VIX");
  vars vixPrice = series(priceClose());
  var vixSMA = SMA(vixPrice, 10);

  asset("^GSPC");

  LifeTime = 8;
  if (NumOpenLong == 0 && vixPrice[0] > vixSMA * (1 + 0.046))
    enterLong();

  if (NumOpenLong > 0 && vixPrice[0] < vixSMA * (1 - 0.035))
    exitLong();
}
{% endhighlight %}

```txt
Monte Carlo Analysis... Median AR 91%
Profit 77920$  MI 385$  DD 9773$  Capital 10962$
Trades 285  Win 68.1%  Avg +67.4p  Bars 4
CAGR 13.75%  PF 1.67  SR 0.69  UI 3%  R2 0.98
```

![Screenshot]({{ site.url }}/assets/images/vixspy.png)

__Note:__ Unfortunately Yahoo does not provide free history data any more, so
this script does not work any longer as is. You would have to remove the
`assetHistory` calls and get the data from some other source than Yahoo.

## References

For the interested reader here are some books with working trading strategies:

* [Short Term Trading Strategies That Work](https://www.amazon.com/Short-Term-Trading-Strategies-Softcover/dp/1616586389/ref=sr_1_1?ie=UTF8&qid=1496579181&sr=8-1&keywords=short+term+trading+strategies+that+work)
* [Bollinger Bands® Trading Strategies That Work](https://www.amazon.com/Bollinger-Trading-Strategies-Research-Strategy-ebook/dp/B00FQM36CQ/ref=sr_1_1?ie=UTF8&qid=1496579195&sr=8-1&keywords=bollinger+band+strategies+that+work)
* [Automated Stock Trading with Zorro](https://www.amazon.com/Automated-Stock-Trading-Zorro-Simons-ebook/dp/B071RQXC9X/ref=sr_1_fkmr0_1?ie=UTF8&qid=1496582333&sr=8-1-fkmr0&keywords=automatic+stock+trading+with+zorro)