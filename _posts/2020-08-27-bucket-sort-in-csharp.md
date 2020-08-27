---
layout: post
title:  "Parallel Bucket Sort in C#"
date:   2020-08-27 7:20:00 +0200
last_modified_at:   2020-08-27 7:20:00 +0200
feature_image: "https://picsum.photos/id/109/1300/400"
category: Development
tags: [csharp, c#, sorting, optimization]
---

Recently I had the problem that sorting data based on a floating-point key was
the bottleneck. I was able to speed things up using Bucketsort in C#.

<!-- more -->

Bucket sort is mainly useful when input is uniformly distributed over a range
for example [0.0 - 1.0]. The input data is first sorted into a certain number of
buckets in linear time and afterwards each bucket is sorted using a comparison
based sorting algorithm. The sorting of each individual bucket can be done in
parallel which can also improve performance compared to a sequential sorting
algorithm. At the end the sorted buckets are merged to form the final sorted
solution.

The Wikipedia article on [BucketSort](https://en.wikipedia.org/wiki/Bucket_sort)
gives a more detailed overview of the algorithm and a complexity analysis.

Here is the implementation that I used:

{% highlight csharp %}
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Threading.Tasks;

namespace ConsoleApp
{
  public class Program
  {
    public static void BucketSort(double[] data, int bucketCount)
    {
      var buckets = new List<double>[bucketCount];
      for (int i = 0; i < bucketCount; i++)
        buckets[i] = new List<double>(data.Length / bucketCount);

      var min = double.MaxValue;
      var max = -double.MaxValue;

      for (int i = 0; i < data.Length; i++)
      {
        min = Math.Min(min, data[i]);
        max = Math.Max(max, data[i]);
      }

      for (int i = 0; i < data.Length; i++)
      {
        var idx = Math.Min(bucketCount - 1, (int)(bucketCount * (data[i] - min) / (max - min)));
        buckets[idx].Add(data[i]);
      }

      Parallel.For(0, bucketCount, i => buckets[i].Sort());

      var index = 0;
      for (var i = 0; i < bucketCount; i++)
      for (var j = 0; j < buckets[i].Count; j++)
        data[index++] = buckets[i][j];
    }

    public static void Main(string[] args)
    {
      for (int algo = 0; algo < 2; algo++)
      {
        var random = new Random(0);
        var sw = Stopwatch.StartNew();

        for (int i = 0; i < 1000; i++)
        {
          var n = random.Next(20000, 200000);
          var data = new double[n];
          for (int j = 0; j < data.Length; j++)
            data[j] = 2 * random.NextDouble() - 1;

          switch (algo)
          {
            case 0:
              Array.Sort(data);
              break;

            case 1:
              BucketSort(data, n / 200);
              break;
          }
        }

        Console.WriteLine($"{(algo == 0 ? "Array.Sort" : "BucketSort")}; {sw.ElapsedMilliseconds}");
      }
    }
  }
}
{% endhighlight %}

On my system i get the following output:

```txt
Array.Sort; 11482
BucketSort; 4928
```

Here Bucketsort is considerably faster.