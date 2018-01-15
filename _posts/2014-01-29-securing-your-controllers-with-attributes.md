---
layout: single
title: Securing your Controllers with Attribute's
date: 2014-01-29 20:19:57.000000000 +01:00
type: post
tags: ["ASP.NET MVC"]
---

Today, while working on an ASP.NET MVC site, I wanted to do something special when securing a controller with the AuthorizeAttribute class:

```csharp
[Authorize(Roles = Role.Administrator.Or(Role.Management).ToString())]
public class SecretController : Controller {
//...
```

Initially, Administrator and Management where `static readonly` instances of the Role class. The `Or` extension method would combine their internal names together in a new Role instance. Finally, because of the overridden `ToString` method, the Roles property would receive the correct end result. However, when compiling, attributes must know their exact value. Read: constant value. So I couldn't assign Role instances at all.
Also, having to end with a ToString() call didn't seem like a nice thing to do either.

So, back to the drawing board. This time, inside my Role class, I decided to declare my known roles as `const string`. But then I still can't use that Or extension method, because that would - again - result in a non-constant expression. I can, however, do this:

```csharp
[Roles(Role.Administrator, Role.Management)]
public class SecretController : Controller {
// ...
```

Roles is a custom attribute, and is very simple indeed:

```csharp
public class RolesAttribute : AuthorizeAttribute
{
   public RolesAttribute(params string[] roles)
   {
      if (roles == null)
         throw new ArgumentNullException("roles", "roles can not be a null reference.");

      Roles = string.Join(",", roles);
   }
}
```

So now, you can specify as many roles as you like. And I'm quite happy with this. But [@chrissie]("http://twitter.com/chrissie") pointed me towards [FluentSecurity](http://www.fluentsecurity.net), which offers an entirely different approach. Also worth a look!