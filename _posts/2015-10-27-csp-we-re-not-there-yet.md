---
layout: single
title: 'CSP: we''re not there, yet.'
date: 2015-10-27 22:00:27.000000000 +01:00
type: post
tags: ["ASP.NET", "Security"]
---

CSP, or Content Security Policy, is an added layer of security that helps to detect and mitigate certain types of attacks, including Cross-Site Scripting (XSS) and data injection attacks. In short, you can add this policy using either a <meta> tag or by setting additional headers in your HTTP response, and using the policy you'll limit the origins for resources for that HTTP request. For example:

* `script-src 'self'` allows loading scripts from the current domain
* `child-src https://youtube.com` allows the use of <iframe> tags pointing towards YouTube
* `style-src 'self' https://maxcdn.bootstrapcdn.com` allows loading CSS files from the current source and from a CDN.

If you want to read up on CSP, I suggest heading over to [http://www.html5rocks.com/en/tutorials/security/content-security-policy/](http://www.html5rocks.com/en/tutorials/security/content-security-policy/).

Now, I was trying out some things concerning CSP and inline scripts. You could add 'unsafe-inline' to the script-src source list to allow all inline scripting (except eval and other bad stuff, you need 'unsafe-eval' in that case), but that is exactly what you would want to avoid. Instead, you can opt to do one of these things:

1. Move all inline JavaScript into .js files and load them using <script src=""></script> tags, while adding 'self' to script-src if that wasn't already the case. The easiest solution, but not always possible.
2. Generate a nonce for each inline script. A nonce means: a unique and hard to predict stream of bytes, different for each script, each page **and** each request!
A nonce could be base64 encoded and look like *RJ6bGjm5EO/X8pImZjvjeAexFVei9IvzNFCGw5lQUa0=*
You would need to add that nonce to the script using the nonce attribute, and also to the script-src source list using 'nonce-RJ6bGjm5EO/X8pImZjvjeAexFVei9IvzNFCGw5lQUa0='
3. Calculate the SHA256, SHA384 or SHA512 digest of each inline script (including all whitespace, but excluding the opening and closing script tags). Like the nonce, you'd need to add these digests to the script-src source list using 'sha256-0ZMofb7eMqkB9IFHnGVZ6Z4chjplAavgi09shinNehs=' for example.


Options 2 and 3 however are only defined in CSP v2, which is currently only implemented by Chrome, Opera and Firefox.
The safest method would be to use the hash digest method, option 3. Alas, when you choose option 3, you'll quickly run into the following issues:

* Chrome. I'm running version 46.0.2490.80 m on a Windows PC and Chrome keeps telling me this:
```
Refused to execute inline script because it violates the following Content Security Policy directive: "script-src 'self' 'sha256-0ZMofb7eMqkB9IFHnGVZ6Z4chjplAavgi09shinNehs='". Either the 'unsafe-inline' keyword, a hash ('sha256-G3FpTn2EFU91Umz2fz6NyjnqeGDTr8SUdmWbsvxzfbY='), or a nonce ('nonce-...') is required to enable inline execution.
```
The issue? My script block contains carriage returns for readability. You know, those '\r\n' thingies which tend to differ sometimes, depending on the OS you're running. But I'm running on Windows and hosting my website locally. Yet Chrome has stripped the '\r' characters from the script before calculating the SHA256 hash to verify my script block... Ugh.

* Firefox. Version 41.0.2, catching up to Chrome apparently. Loading my test site and...
```
Content Security Policy: The page's settings blocked the loading of a resource at self ("script-src http://localhost:21275 'sha256-0ZMofb7eMqkB9IFHnGVZ6Z4chjplAavgi09shinNehs='")
```

* Edge. The newest browser from Microsoft, which I really want to start to use. But then I bump, as expected, into:
```
CSP14304: Unknown source ''sha256-0ZMofb7eMqkB9IFHnGVZ6Z4chjplAavgi09shinNehs='' for directive 'script-src' in Content-Security-Policy - source will be ignored.
```

* Let's try Internet Explorer, version 11.0.10240.16431. I'll dump the entire console output to share this with you:
```
Navigation occurred.
localhost:21275
test
```
Amazing, no? Wait, let's try something else here. Let's remove the hash from the header, that should stop the script from running. I'll dump the console output again:
```
Navigation occurred.
localhost:21275
test
```
So don't be fooled here. Internet Explorer doesn't support CSP v2, and even v1 is only partially supported.

Let's try option 2 instead: making use of a nonce for every script block. This time, the results are looking a *little bit* better:
* Chrome is happily executing the script now.
* So is Firefox, but Firefox keeps complaining about another script. I have no clue which one. I could try to find out, using the CSP reporting options, but I'm not going to bother.

So there you have it: Content Security Policy is still not fully there yet. You can use it though, to restrict the use of other content, like webfonts, style sheets, media, ..., but for scripts our options are currently limited to either opening up 'unsafe-inline' or moving all JavaScript into files. And if you're doing ASP.NET projects, have a look at [https://www.nuget.org/packages/NWebsec](https://www.nuget.org/packages/NWebsec). This NuGet package will make implementing CSP - and other security headers - a **lot** easier for you :)
