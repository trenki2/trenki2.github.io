---
layout: post
title:  "Trading with Zorro"
date:   2017-06-03 14:20:00 +0200
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

Currently I am still using Zorro. The bugs could be resolved in cooperation with
the Zorro Support Team and now I am successfully trading stocks.

## The Strategy

## References

For the interested reader here are two books with working trading strategies:

* [Short Term Trading Strategies That Work](https://www.amazon.com/Short-Term-Trading-Strategies-Softcover/dp/1616586389/ref=sr_1_1?ie=UTF8&qid=1496579181&sr=8-1&keywords=short+term+trading+strategies+that+work)
* [Bollinger Bands® Trading Strategies That Work](https://www.amazon.com/Bollinger-Trading-Strategies-Research-Strategy-ebook/dp/B00FQM36CQ/ref=sr_1_1?ie=UTF8&qid=1496579195&sr=8-1&keywords=bollinger+band+strategies+that+work)

