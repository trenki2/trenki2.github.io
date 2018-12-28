---
layout: post
title:  "Activator.CreateInstance Alternative"
date:   2018-12-28 12:00:00 +0200
date:   2018-12-28 12:00:00 +0200
feature_image: "https://unsplash.it/1200/400?image=500"
category: Development
tags: [csharp, c#, activator, compiled expression tree]
---

I noticed that the performance of the C# Activator.CreateInstance method can be
a bit slow. Using Compiled Expression Trees it is possible to speed things up a
lot.

<!-- more -->

I created a solution that can be used as a replacement for
Activator.CreateInstance. The InstanceFactory can create objects by calling the
constructor directly. Up tp three constructor parameters are supported.

**Example:**
{% highlight csharp %}
var myInstance = InstanceFactory.CreateInstance(typeof(MyClass));
var myArray = InstanceFactory.CreateInstance(typeof(int[]), 1024);
{% endhighlight %}

**Code:**
{% highlight csharp %}
public static class InstanceFactory
{
  public static object CreateInstance(Type type)
  {
    return InstanceFactoryGeneric<TypeToIgnore, TypeToIgnore, TypeToIgnore>.CreateInstance(type, null, null, null);
  }

  public static object CreateInstance<TArg1>(Type type, TArg1 arg1)
  {
    return InstanceFactoryGeneric<TArg1, TypeToIgnore, TypeToIgnore>.CreateInstance(type, arg1, null, null);
  }

  public static object CreateInstance<TArg1, TArg2>(Type type, TArg1 arg1, TArg2 arg2)
  {
    return InstanceFactoryGeneric<TArg1, TArg2, TypeToIgnore>.CreateInstance(type, arg1, arg2, null);
  }

  public static object CreateInstance<TArg1, TArg2, TArg3>(Type type, TArg1 arg1, TArg2 arg2, TArg3 arg3)
  {
    return InstanceFactoryGeneric<TArg1, TArg2, TArg3>.CreateInstance(type, arg1, arg2, arg3);
  }
}

public static class InstanceFactoryGeneric<TArg1, TArg2, TArg3>
{
  private static ConcurrentDictionary<Type, Func<TArg1, TArg2, TArg3, object>> cachedFuncs = new ConcurrentDictionary<Type, Func<TArg1, TArg2, TArg3, object>>();

  public static object CreateInstance(Type type, TArg1 arg1, TArg2 arg2, TArg3 arg3)
  {
    if (cachedFuncs.TryGetValue(type, out Func<TArg1, TArg2, TArg3, object> func))
      return func(arg1, arg2, arg3);
    else
      return CacheFunc(type, arg1, arg2, arg3)(arg1, arg2, arg3);
  }

  private static Func<TArg1, TArg2, TArg3, object> CacheFunc(Type type, TArg1 arg1, TArg2 arg2, TArg3 arg3)
  {
    var constructorTypes = new List<Type>();
    if (typeof(TArg1) != typeof(TypeToIgnore))
      constructorTypes.Add(typeof(TArg1));
    if (typeof(TArg2) != typeof(TypeToIgnore))
      constructorTypes.Add(typeof(TArg2));
    if (typeof(TArg3) != typeof(TypeToIgnore))
      constructorTypes.Add(typeof(TArg3));

    var parameters = new List<ParameterExpression>()
    {
      Expression.Parameter(typeof(TArg1)),
      Expression.Parameter(typeof(TArg2)),
      Expression.Parameter(typeof(TArg3)),
    };

    var constructor = type.GetConstructor(constructorTypes.ToArray());
    var constructorParameters = parameters.Take(constructorTypes.Count).ToList();
    var newExpr = Expression.New(constructor, constructorParameters);
    var lambdaExpr = Expression.Lambda<Func<TArg1, TArg2, TArg3, object>>(newExpr, parameters);
    var func = lambdaExpr.Compile();
    cachedFuncs.TryAdd(type, func);
    return func;
  }
}

public class TypeToIgnore
{
}

{% endhighlight %}