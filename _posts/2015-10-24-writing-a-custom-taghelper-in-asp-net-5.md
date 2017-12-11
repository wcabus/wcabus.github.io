---
layout: single
title: Writing a custom TagHelper in ASP.NET 5
date: 2015-10-24 21:10:34.000000000 +02:00
type: post
tags: ["ASP.NET", "ASP.NET MVC"]
---

ASP.NET 5 brings some new features to MVC. One of these features is *TagHelpers*, which allow us to add new attributes to HTML tags (or create custom tags altogether) to add behavior to these tags. At this moment - beta8 - we receive the following TagHelpers out of the box:
- AnchorTagHelper: allows you to write `<a asp-controller="Home" asp-action="Index">Back to home</a>` instead of `@Html.ActionLink("Back to home", "Index", "Home")`
- LabelTagHelper, InputTagHelper, TextAreaTagHelper, SelectTagHelper, etc.: instead of writing Razor like `@Html.LabelFor(m => m.Property1, new { @class = "control-label col-md-2"})`, you can use a syntax which is much cleaner: `<label asp-for="Property1" class="control-label col-md-2"></label>`
- EnvironmentTagHelper: adds a new tag, `<environment>`, giving you the possibility to include `<link>` or `<script>` tags specific for Development or Production environments.
- LinkTagHelper, ScriptTagHelper: adds attributes to `<link>` and `<script>` to allow the use of CDN's and fallback urls.
This list is incomplete, but you can [head over to GitHub](https://github.com/aspnet/Mvc/tree/dev/src/Microsoft.AspNetCore.Mvc.TagHelpers) to see every available TagHelper class currently available.

You can also create your own TagHelper classes. I'll show you an example which enables you to do this:
```html
<img asp-controller="Users" asp-action="ProfileImage" asp-route-id="@currentUserId" />
```
That *img* tag would load a users profile image using the ProfileImage action method in a UsersController class: userful for retrieving a profile image from a blob container on Azure, for example.


To create a custom TagHelper class, start by adding a dependency to the Microsoft.AspNet.Razor.Runtime NuGet package in project.json. You can add this dependency to the top list of global dependencies:
```js
{
  "webroot": "wwwroot",
  "version": "1.0.0-*",

  "dependencies": {
    "Microsoft.AspNet.Diagnostics": "1.0.0-beta8",
    "Microsoft.AspNet.IISPlatformHandler": "1.0.0-beta8",
    "Microsoft.AspNet.Mvc": "6.0.0-beta8",
    "Microsoft.AspNet.Mvc.TagHelpers": "6.0.0-beta8",
    "Microsoft.AspNet.Server.Kestrel": "1.0.0-beta8",
    "Microsoft.AspNet.StaticFiles": "1.0.0-beta8",
    "Microsoft.AspNet.Tooling.Razor": "1.0.0-beta8",
    "Microsoft.Framework.Configuration.Json": "1.0.0-beta8",
    "Microsoft.Framework.Logging": "1.0.0-beta8",
    "Microsoft.Framework.Logging.Console": "1.0.0-beta8",
    "Microsoft.Framework.Logging.Debug": "1.0.0-beta8",
    "Microsoft.VisualStudio.Web.BrowserLink.Loader": "14.0.0-beta8",
    "Microsoft.AspNet.Razor.Runtime":  "4.0.0-beta8"
  },

...
```
Next, add a new Class to your MVC project. Let's call this class **ImageTagHelper**, and let the class derive from the **TagHelper** class which lives in the *Microsoft.AspNet.Razor.Runtime.TagHelpers* namespace.
We're going to add three new attributes to the <img> tag:
* asp-controller: the MVC Controller
* asp-action: the action method within the controller
* asp-route-*: route values for the action method

For each of these attributes, add a class level HtmlTargetElementAttribute:
```csharp
[HtmlTargetElement("img", Attributes = ActionAttributeName)]
[HtmlTargetElement("img", Attributes = ControllerAttributeName)]
[HtmlTargetElement("img", Attributes = RouteValuesPrefix + "*")]
public class ProfileImageTagHelper : TagHelper
{
    private const string ActionAttributeName = "asp-action";
    private const string ControllerAttributeName = "asp-controller";
    private const string RouteValuesPrefix = "asp-route-";
}
```
I'm using constants here to define the attribute names: I can reuse these later when adding the properties which will receive the values of the HTML attributes. Let's add these properties now:
```csharp
[HtmlTargetElement("img", Attributes = ActionAttributeName)]
[HtmlTargetElement("img", Attributes = ControllerAttributeName)]
[HtmlTargetElement("img", Attributes = RouteValuesPrefix + "*")]
public class ProfileImageTagHelper : TagHelper
{
    private const string ActionAttributeName = "asp-action";
    private const string ControllerAttributeName = "asp-controller";
    private const string RouteValuesPrefix = "asp-route-";

    /// <summary>
    /// The name of the action method.
    /// </summary>
    [HtmlAttributeName(ActionAttributeName)]
    public string Action { get; set; }

    /// <summary>
    /// The name of the controller.
    /// </summary>
    [HtmlAttributeName(ControllerAttributeName)]
    public string Controller { get; set; }

    /// <summary>
    /// Additional parameters for the route.
    /// </summary>
    [HtmlAttributeName(DictionaryAttributePrefix = RouteValuesPrefix)]
    public IDictionary<string, string> RouteValues { get; set; } =
        new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase);
}
```
By using a * at the end of the HtmlTargetElementAttribute, we can map multiple values into an IDictionary<string, string>. Think of this as a way to specify only one property to hold all data-* attributes.
It is now time to add the logic which will make this TagHelper work. Let's override the Process method:
```csharp
public override void Process(TagHelperContext context, TagHelperOutput output)
{
	if (context == null)
	{
		throw new ArgumentNullException(nameof(context));
	}

	if (output == null)
	{
		throw new ArgumentNullException(nameof(output));
	}

	// If "src" is already set, it means the user is attempting to use a normal img tag.
	if (output.Attributes.ContainsName("src"))
	{
		if (Action != null ||
			Controller != null ||
			RouteValues.Count != 0)
		{
			// User specified an src and one of the bound attributes; can't determine the src attribute.
			throw new InvalidOperationException(
				"Cannot override the src attribute of an <img> tag if the src attribute has a value.");
		}
	}
	else
	{
		// Convert from Dictionary<string, string> to Dictionary<string, object>.
		var routeValues = RouteValues.ToDictionary(
			kvp => kvp.Key,
			kvp => (object) kvp.Value,
			StringComparer.OrdinalIgnoreCase);

		var tagBuilder = new TagBuilder("img");
		tagBuilder.MergeAttribute("src", _urlHelper.Action(Action, Controller, routeValues));

		output.MergeAttributes(tagBuilder);
	}
}
```
First of all, we check if the consumer of this custom TagHelper hasn't set a value for the *src* attribute, because then the <img> tag would just need to look at the original URI to retrieve the image. We will however show an error when the *src* attribute and one of our custom attributes has been set together.
If the developer is using our TagHelper without a *src* attribute, then we'll fill in the attribute ourselves using _urlHelper.Action(): that's the @Url.Action method you've been using in Razor all along. Of course, we still need to add that _urlHelper instance to our class and inject it. Here is the final version of our TagHelper class code:
```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using Microsoft.AspNet.Mvc;
using Microsoft.AspNet.Mvc.Rendering;
using Microsoft.AspNet.Mvc.TagHelpers;
using Microsoft.AspNet.Razor.Runtime.TagHelpers;

namespace CustomTagHelperDemo.TagHelpers
{
    [HtmlTargetElement("img", Attributes = ActionAttributeName)]
    [HtmlTargetElement("img", Attributes = ControllerAttributeName)]
    [HtmlTargetElement("img", Attributes = RouteValuesPrefix + "*")]
    public class ProfileImageTagHelper : TagHelper
    {
        private readonly IUrlHelper _urlHelper;

        private const string ActionAttributeName = "asp-action";
        private const string ControllerAttributeName = "asp-controller";
        private const string RouteValuesPrefix = "asp-route-";
        
        public ProfileImageTagHelper(IUrlHelper urlHelper)
        {
            _urlHelper = urlHelper;
        }

        /// <summary>
        /// The name of the action method.
        /// </summary>
        /// <remarks>Must be <c>null</c> if <see cref="Route"/> is non-<c>null</c>.</remarks>
        [HtmlAttributeName(ActionAttributeName)]
        public string Action { get; set; }

        /// <summary>
        /// The name of the controller.
        /// </summary>
        /// <remarks>Must be <c>null</c> if <see cref="Route"/> is non-<c>null</c>.</remarks>
        [HtmlAttributeName(ControllerAttributeName)]
        public string Controller { get; set; }

        /// <summary>
        /// Additional parameters for the route.
        /// </summary>
        [HtmlAttributeName(DictionaryAttributePrefix = RouteValuesPrefix)]
        public IDictionary<string, string> RouteValues { get; set; } =
            new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase);

        /// <summary>
        /// Synchronously executes the <see cref="T:Microsoft.AspNet.Razor.Runtime.TagHelpers.TagHelper"/> with the given <paramref name="context"/> and
        ///             <paramref name="output"/>.
        /// </summary>
        /// <param name="context">Contains information associated with the current HTML tag.</param><param name="output">A stateful HTML element used to generate an HTML tag.</param>
        /// <remarks>Does nothing if user provides an <c>src</c> attribute.</remarks>
        /// <exception cref="InvalidOperationException">
        /// Thrown if <c>src</c> attribute is provided and <see cref="Action"/> or <see cref="Controller"/> are
        /// non-<c>null</c> or if the user provided <c>asp-route-*</c> attributes.
        /// </exception>
        public override void Process(TagHelperContext context, TagHelperOutput output)
        {
            if (context == null)
            {
                throw new ArgumentNullException(nameof(context));
            }

            if (output == null)
            {
                throw new ArgumentNullException(nameof(output));
            }

            // If "src" is already set, it means the user is attempting to use a normal anchor.
            if (output.Attributes.ContainsName("src"))
            {
                if (Action != null ||
                    Controller != null ||
                    RouteValues.Count != 0)
                {
                    // User specified an href and one of the bound attributes; can't determine the href attribute.
                    throw new InvalidOperationException(
                        "Cannot override the src attribute of an <img> tag if the src attribute has a value.");
                }
            }
            else
            {
                // Convert from Dictionary<string, string> to Dictionary<string, object>.
                var routeValues = RouteValues.ToDictionary(
                    kvp => kvp.Key,
                    kvp => (object) kvp.Value,
                    StringComparer.OrdinalIgnoreCase);

                var tagBuilder = new TagBuilder("img");
                tagBuilder.MergeAttribute("src", _urlHelper.Action(Action, Controller, routeValues));

                output.MergeAttributes(tagBuilder);
            }
        }
    }
}
```
But we're not done, yet. To be able to use custom TagHelper classes, you'll need to let Razor know about them.  Open the _ViewImports.cshtml file in the Views/Shared folder and add the following line:
```csharp
@addTagHelper "*, CustomTagHelperDemo"
```
This line will tell Razor to load all TagHelper derived classes from the assembly CustomTagHelperDemo. You can also use the name of a specific TagHelper class instead of an asterisk. Also, don't forget to replace *CustomTagHelperDemo* with the name of your project or class library.
And now, you can open up a Razor file and tryout the new features you've added to the <img> tag:
```html
<ul class="nav navbar-nav navbar-right">
	@if (User.Identity?.IsAuthenticated == true)
	{
		<li>
			<img asp-controller="Users" asp-action="ProfileImage" asp-route-id="@User.FindFirst("sub").Value" />
			<a asp-controller="Account" asp-action="Logoff">Sign out</a>
		</li>
	}
	else
	{
		<li>
			<a asp-controller="Account" asp-action="Logon">Sign in</a>
		</li>
	}
</ul>
```
