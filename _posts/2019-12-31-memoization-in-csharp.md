---
layout: post
title:  "Memoization in C#"
date:   2018-12-31 12:00:00 +0200
date:   2018-12-31 12:00:00 +0200
feature_image: "https://unsplash.it/1200/400?image=524"
category: Development
tags: [csharp, c#, memoization]
---

Memoization is a technique for improving performance by caching the return
values of expensive function calls. In this post I show how you can use this
technique in C#.

<!-- more -->

A typical example to illustrate the effectiveness of memoization is the
computation of the fibonacci sequence.

{% highlight csharp %}
Func<int, int> fib = null;
fib = n => n > 1 ? fib(n - 1) + fib(n - 2) : n;

// Without memoization
var sw = Stopwatch.StartNew();
for (int i = 0; i < 10000; i++)
    fib(10);
Console.WriteLine(sw.ElapsedTicks);

// Memoize function
fib = fib.Memoize();

// With memoization
sw = Stopwatch.StartNew();
for (int i = 0; i < 10000; i++)
    fib(10);
Console.WriteLine(sw.ElapsedTicks);
{% endhighlight %}

Output:
```
13532
2744
```

A simple way to apply the memoization as a refactoring could be the following:

Before:
{% highlight csharp %}
public object Function(int value)
{
  object result;
  // ... expensive code ...
  return result;
}
{% endhighlight %}

After:
{% highlight csharp %}
private Func<int, object> functionMemo;

public object Function(int value)
{
  if (functionMemo == null)
  {
    functionMemo = Memoizer.Memoize<int, object>((v) =>
    {
      // Here comes the original code

      object result;
      // ... expensive code ...
      return result;
    });
  }

  return functionMemo(value);
}
{% endhighlight %}

To use the Memoizer class and extension methods you need the following code:

{% highlight csharp %}
using System;
using System.Collections.Generic;

namespace Memoize
{
  public static class Memoizer
  {
    public static Func<R> Memoize<R>(Func<R> func)
    {
      object cache = null;
      return () =>
      {
        if (cache == null)
          cache = func();
        return (R)cache;
      };
    }

    public static Func<A, R> Memoize<A, R>(Func<A, R> func)
    {
      var cache = new Dictionary<A, R>();
      return a =>
      {
        if (cache.TryGetValue(a, out R value))
          return value;
        value = func(a);
        cache.Add(a, value);
        return value;
      };
    }
  }

  public static class MemoizerExtensions
  {
    public static Func<R> Memoize<R>(this Func<R> func)
    {
      return Memoizer.Memoize(func);
    }

    public static Func<A, R> Memoize<A, R>(this Func<A, R> func)
    {
      return Memoizer.Memoize(func);
    }
  }
}
{% endhighlight %}