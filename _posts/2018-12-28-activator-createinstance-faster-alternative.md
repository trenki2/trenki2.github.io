---
layout: post
title:  "Activator.CreateInstance Alternative"
date:   2018-12-28 12:00:00 +0200
last_modified_at:   2018-12-28 12:00:00 +0200
feature_image: "https://unsplash.it/1200/400?image=500"
category: Development
tags: [csharp, c#, activator, compiled expression tree]
---

I noticed that the performance of the C# Activator.CreateInstance method can be
a bit slow. Using Compiled Expression Trees it is possible to speed things up a
lot.

<!-- more -->

I created a solution that can be used as a drop in replacement for
Activator.CreateInstance. The InstanceFactory can create objects by calling the
constructor directly using Expression Trees. Up tp three constructor parameters
are supported. If more parameters are used it automatically falls back to using
Activator.

**Example:**
{% highlight csharp %}
var myInstance = InstanceFactory.CreateInstance(typeof(MyClass));
var myArray1 = InstanceFactory.CreateInstance(typeof(int[]), 1024);
var myArray2 = InstanceFactory.CreateInstance(typeof(int[]), new object[] { 1024 });
{% endhighlight %}

**Code:**
{% highlight csharp %}
public static class InstanceFactory
{
  private delegate object CreateDelegate(Type type, object arg1, object arg2, object arg3);

  private static ConcurrentDictionary<Tuple<Type, Type, Type, Type>, CreateDelegate> cachedFuncs = new ConcurrentDictionary<Tuple<Type, Type, Type, Type>, CreateDelegate>();

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

  public static object CreateInstance(Type type, params object[] args)
  {
    if (args == null)
      return CreateInstance(type);

    if (args.Length > 3 || 
      (args.Length > 0 && args[0] == null) ||
      (args.Length > 1 && args[1] == null) ||
      (args.Length > 2 && args[2] == null))
    {
        return Activator.CreateInstance(type, args);   
    }

    var arg0 = args.Length > 0 ? args[0] : null;
    var arg1 = args.Length > 1 ? args[1] : null;
    var arg2 = args.Length > 2 ? args[2] : null;

    var key = Tuple.Create(
      type,
      arg0?.GetType() ?? typeof(TypeToIgnore),
      arg1?.GetType() ?? typeof(TypeToIgnore),
      arg2?.GetType() ?? typeof(TypeToIgnore));
    
    if (cachedFuncs.TryGetValue(key, out CreateDelegate func))
      return func(type, arg0, arg1, arg2);
    else
      return CacheFunc(key)(type, arg0, arg1, arg2);
  }

  private static CreateDelegate CacheFunc(Tuple<Type, Type, Type, Type> key)
  {
    var types = new Type[] { key.Item1, key.Item2, key.Item3, key.Item4 };
    var method = typeof(InstanceFactory).GetMethods()
                                        .Where(m => m.Name == "CreateInstance")
                                        .Where(m => m.GetParameters().Count() == 4).Single();
    var generic = method.MakeGenericMethod(new Type[] { key.Item2, key.Item3, key.Item4 });

    var paramExpr = new List<ParameterExpression>();
    paramExpr.Add(Expression.Parameter(typeof(Type)));
    for (int i = 0; i < 3; i++)
      paramExpr.Add(Expression.Parameter(typeof(object)));

    var callParamExpr = new List<Expression>();
    callParamExpr.Add(paramExpr[0]);
    for (int i = 1; i < 4; i++)
      callParamExpr.Add(Expression.Convert(paramExpr[i], types[i]));
    
    var callExpr = Expression.Call(generic, callParamExpr);
    var lambdaExpr = Expression.Lambda<CreateDelegate>(callExpr, paramExpr);
    var func = lambdaExpr.Compile();
    cachedFuncs.TryAdd(key, func);
    return func;
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