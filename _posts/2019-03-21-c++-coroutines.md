---
layout: post
title:  "Simple Coroutines in C++"
date:   2019-03-21 9:00:00 +0200
#last_modified_at:   2019-01-02 17:00:00 +0200
feature_image: "https://unsplash.it/1200/400?image=202"
category: Development
tags: [c++, coroutines, boost, execution_context]
---

Recently i was interested in how I could implement a simple coroutine solution
in C++. I wanted to use it for StarCraft AI Development.

<!-- more -->

I tried to find a solution for coroutines in c++ that was simple and easy to
use. I found the [cross platform fibers/coroutines in
c++](http://www.subatomicglue.com/secret/coro/readme.html) library that seemed
nice but was old, so i continued looking. They mentioned
**boost::execution_context** as a portable alternative so I decided to give that
a try.

The interface for **boost::execution_context** seemed simple enough so I used
that to implement a simple coroutine class around it.

To implement a coroutine with my solution one just has to derive from the
`Coroutine` class and override the `run()` method. One can use the `yield()`
method to yield the current coroutine. When the coroutine is resumed with
`resume()` it continues after the last `yield()` call. A coroutine can also be
aborted with `abort()`. In that case the last `yield()` will throw
`CoroutineAbortedException()` and make sure that the stack is unwound and return
from the coroutine.

Example coroutine:

{% highlight cpp %}
class MyCoroutine : public Coroutine {
protected:
  virtual void run() override
  {
    std::cout << 1 << endl;
    yield();
    std::cout << 2 << endl;
    yield();
    std::cout << 3 << endl;
  }
};
{% endhighlight %}

Example usage: 

{% highlight cpp %}
int main(int argc, char *argv[])
{
  MyCoroutine c;
  
  while (c.resume() != CoroutineStatus::Running)
  {}

  c.abort();
}
{% endhighlight %}

Full source code for the `Coroutine` implementation:

{% highlight cpp %}
#pragma once


#include <boost/context/execution_context.hpp>


struct CoroutineAbortedException : std::exception
{
  const char *what() const throw()
  {
    return "CoroutineAbortedException";
  }
};

enum CoroutineStatus
{
  Running,
  Finished
};

class Coroutine {
private:
  CoroutineStatus status_;
  boost::context::execution_context<int> source_;
  boost::context::execution_context<int> sink_;

public:
  Coroutine()
  {
    source_ = boost::context::execution_context<int>([this](boost::context::execution_context<int> sink, int)
    {
      sink_ = std::move(sink);
      status_ = CoroutineStatus::Running;

      try { this->run(); }
      catch (CoroutineAbortedException&) { }

      status_ = CoroutineStatus::Finished;
      return std::move(sink_);
    });

    status_ = Running;
  }

  CoroutineStatus status() { return status_; }

  CoroutineStatus resume()
  {
    if (status_ != CoroutineStatus::Running)
      return status_;

    auto result = source_(0);
    source_ = std::move(std::get<0>(result));

    return status_;
  }

  void abort()
  {
    if (status_ != CoroutineStatus::Running)
      return;

    auto result = source_(1);
    source_ = std::move(std::get<0>(result));
  }

protected:
  void yield()
  {
    auto result = (sink_)(0);
    sink_ = std::move(std::get<0>(result));
    if (std::get<1>(result) == 1)
      throw CoroutineAbortedException();
  }

  virtual void run() = 0;
};
{% endhighlight %}

