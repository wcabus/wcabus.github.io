---
layout: single
title: Cors.ConfigProfiles is here!
date: 2014-06-13 12:40:34.000000000 +02:00
type: post
tags: ["C&num;", "ASP.NET", "Web API"]
---

After working on [my previous blog post](http://wesleycabus.be/2014/06/adding-an-mvc-layer-on-top-of-a-web-api-backend/), where I had to use the EnableCorsAttribute, I thought: "Why can't I make profiles for this in the web.config file, like you can for the OutputCacheAttribute?"

Enter my brand new, shiny NuGet package: [Cors.ConfigProfiles](https://www.nuget.org/packages/Cors.ConfigProfiles/)! It enables you to use the EnableCorsAttribute just like you normally would, but you can also just give it one string parameter which then matches a profile inside the web.config file. So, for example, if you put this attribute on an API controller:

```csharp
[EnableCors("MyApiProfile")]
public class MyApiController : ApiController {
    // ...
}
```

Then you can configure that profile in web.config like so:

```csharp
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <configSections>
    <section name="cors" type="Cors.ConfigProfiles.Configuration.CorsSection, Cors.ConfigProfiles, Version=5.1.3.0, Culture=neutral, PublicKeyToken=253caed373cef676" requirePermission="false" />
  </configSections>
  <cors>
    <corsProfiles>
      <add name="MyApiProfile" 
           origins="*" 
           methods="*" 
           headers="*"
      />
    </corsProfiles>
  </cors>
</configuration>
```

And if you want to contribute or just inspect what I did (no unicorns involved though), the code is available on [GitHub](https://github.com/wcabus/mvc-cors-profile).

