---
layout: single
title: Fluent API with Nullable this time :)
date: 2014-03-28 09:48:18.000000000 +01:00
type: post
tags: ["C&num;"]
---

Yesterday, I attended [Kevin's session about building a fluent API](http://blog.kevindockx.com/post/Fluent-API-Session-Code-Available.aspx). One of the things he mentioned while building the API, was that he would have liked to constrain the use of the NotNull extension method to cases where it makes sense. And I thought I knew how, but it was already past 8 PM...

After he released the code for the session this morning, I took another go at it. And behold:
![FluentAPI - Nullable]({{ "/assets/FluentAPI.png" | absolute_url }})

Now, to achieve this is actually really simple once you know the magic. The original code for the NotNull extension method was this:

```csharp
public static RuleBuilder<T, TProp>
    NotNull<T, TProp>(this RuleBuilder<T, TProp> ruleBuilder)
    where T : class
{
    return ruleBuilder.AddValidator(new NotNullValidator());
}
```

Let's first fix this one, so it only allows reference types. Easy: put another type constraint to make sure that TProp is a reference type:
```csharp
public static RuleBuilder<T, TProp>
    NotNull<T, TProp>(this RuleBuilder<T, TProp> ruleBuilder)
    where T : class
    where TProp : class
{
    return ruleBuilder.AddValidator(new NotNullValidator());
}
```

Now, for Nullable types, we need to add another extension method. One where TProp is something nullable, of course.

```csharp
public static RuleBuilder<T, TValue?>
    NotNull<T, TValue>(this RuleBuilder<T, TValue?> ruleBuilder)
    where T : class
    where TValue: struct
{
    return ruleBuilder.AddValidator(new NotNullValidator());
}
```

The magic here is in the difference between `TValue` and `TValue?` (or `Nullable<TValue>`). When calling NotNull, we want to pass a struct or value type for the TValue type parameter, but the RuleBuilder will use the nullable one as property type. While we can't define `NotNull<T, TValue?>` or use Nullable in the type constraint, we can pass it along to the RuleBuilder. The compiler is smart enough to deduce the rest using type inference :)