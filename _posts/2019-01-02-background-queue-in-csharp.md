---
layout: post
title:  "Background Worker Queue in C#"
date:   2019-01-02 17:00:00 +0200
date:   2019-01-02 17:00:00 +0200
feature_image: "https://unsplash.it/1200/400?image=100"
category: Development
tags: [csharp, c#, background queue]
---

Sometimes it is required to run multimple tasks in the background, while one
task can only start after another has finished. In this case a simple background
queue based on the C# TPL can be used.

<!-- more -->

Most of the time a simple **Task.Run()** can be used to do work on the
background thread pool. But when one has to process multiple tasks one after the
other this does not work any more since using Task.Run() does not guarantee any
order for the processed tasks.

In this case the following code can be used to have a background queue and
execute tasks one after the other.

{% highlight csharp %}
var queue = new BackgroundQueue();
queue.QueueTask(() => { /* Do some work */ });
queue.QueueTask(() => { /* Do some more work */ });
var result = await queue.QueueTask(() => { return result; });
{% endhighlight %}

Here is the code for the background queue class:

{% highlight csharp %}
public class BackgroundQueue
{
  private Task previousTask = Task.FromResult(true);
  private object key = new object();

  public Task QueueTask(Action action)
  {
    lock (key)
    {
      previousTask = previousTask.ContinueWith(
        t => action(),
        CancellationToken.None,
        TaskContinuationOptions.None,
        TaskScheduler.Default);
      return previousTask;
    }
  }

  public Task<T> QueueTask<T>(Func<T> work)
  {
    lock (key)
    {
      var task = previousTask.ContinueWith(
        t => work(), 
        CancellationToken.None,
        TaskContinuationOptions.None,
        TaskScheduler.Default);
      previousTask = task;
      return task;
    }
  }
}
{% endhighlight %}
