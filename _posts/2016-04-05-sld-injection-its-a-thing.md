---
layout: single
title: SLD Injection, it's a thing
date: 2016-04-05 12:00:47.000000000 +02:00
type: post
categories:
- C&num;
- Security
tags: []
---

<p>I'm in the middle of preparing a session about security, and one topic you regularly bump into when thinking about security and writing code is SQL Injection. Although it is the year 2016, there are still people writing code which is vulnerable against SQL Injection.</p>
<p>But I'm not going to focus on SQL Injection here, I'm going to take it one step further. If you want to read up on SQL Injection, <a href="http://www.bing.com/search?q=sql+injection" target="_blank">Bing</a>/<a href="https://www.google.be/#q=sql+injection" target="_blank">Google</a> is your friend. Or just take a look at good old <a href="http://bobby-tables.com/" target="_blank">Bobby Tables</a>.</p>
<p>People often think: "It's called SQL Injection, but I'm not using SQL here, so I'm safe."</p>
<p>Wrong.</p>
<p>From the moment you're using any type of query or filtering language construct, you can introduce some form of SQL Injection in your code:</p>
<ul>
<li>Using (n)Hibernate with <a href="https://docs.jboss.org/hibernate/orm/3.3/reference/en/html/queryhql.html" target="_blank">HQL</a>? Are you using parameters in those queries <a href="http://blog.h3xstream.com/2014/02/hql-for-pentesters.html" target="_blank">or not</a>?</li>
<li>You're using Azure Table Storage or another NoSQL DB. <a href="https://securityintelligence.com/does-nosql-equal-no-injection/" target="_blank">But does NoSQL mean No SQL Injection</a>?</li>
</ul>
<p>Another fun example uses the <a href="https://www.nuget.org/packages/System.Linq.Dynamic/" target="_blank">System.Linq.Dynamic</a> NuGet package. This little gem allows you to use string expressions to filter on IEnumerable's. And because it makes our developer life easier, we could use this to filter data in a Web API, for example. Take a look at this piece of code:</p>

```csharp
public async Task<IEnumerable<User>> GetUsersAsync(string filter)
{
	// Lets say the current user can only access data with odd ID's
	var fullFilter = "(id % 2) = 1";

	if (!string.IsNullOrEmpty(filter))
	{
		fullFilter += $" && {filter}";
	}

	return (await DbContext.Users.ToListAsync()).Where(fullFilter);
}
```

<p>Yes, I know this is not the best code because you're fetching all users from the database and then you reduce the result set, but bear with me. I'm making a statement here. Besides, this kind of code is still being written in production as well!</p>
<p>As you can see, you're applying a filter to a list of User instances using System.Linq.Dynamic. That filter could be, for example, this:</p>
`"name.StartsWith(\"n\")"`
<p>This would reduce the results down to all users with an odd ID and where the surname starts with "n". But someone could circumvent our fixed "odd ID" filter part, which would ignore our little security measure, by changing the filter a bit:</p>
`1 = 1 || 1 = 1`
<p>This filter value would make fullFilter look like this:</p>
`(id % 2) = 1 && 1 = 1 || 1 = 1`
<p>Because we're missing parentheses around the user filter part, the ID filter will be ignored. One solution could be to add these parentheses.</p>

```csharp
public async Task<IEnumerable<User>> GetUsersAsync(string filter)
{
	// Lets say the current user can only access data with odd ID's
	var fullFilter = "(id % 2) = 1";

	if (!string.IsNullOrEmpty(filter))
	{
		fullFilter += $" && ({filter})";
	}

	return (await DbContext.Users.ToListAsync()).Where(fullFilter);
}
```

<p>Actually, you could solve that problem by moving the fixed filter part. That way, no matter what your user is requesting, he or she can't get by the fixed filter anymore:</p>

```csharp
public async Task<IEnumerable<User>> GetUsersAsync(string filter)
{
   // Lets say the current user can only access data with odd ID's
   var query = DbContext.Users.Where(x => x.Id % 2 == 1);

   if (!string.IsNullOrEmpty(filter))
   {
      return (await query.ToListAsync()).Where(filter);
   }

   return await query.ToListAsync();
}
```

<p>And to complete the story, you can look at using parameterized queries, <a href="http://dynamiclinq.azurewebsites.net/GettingStarted" target="_blank">which is also supported by System.Linq.Dynamic</a>. Using parameters in your filter would take away some of the filtering power from the consumer of your API, but you could reintroduce that power by defining your own filters. This example shows a <strong>basic attempt</strong> at implementing this:</p>

```csharp
public async Task<IEnumerable<User>> GetUsersAsync(string filter)
{
	// Lets say the current user can only access data with odd ID's
	var query = DbContext.Users.Where(x => x.Id % 2 == 1);

	if (!string.IsNullOrEmpty(filter))
	{
		var parsed = ParseFilter(filter);
		return (await query.ToListAsync()).Where(parsed.Filter, parsed.ParamList.ToArray());
	}

	return await query.ToListAsync();
}

private ParsedFilter ParseFilter(string filter)
{
	var newFilter = new StringBuilder(filter.Length);
	var paramList = new List<object>();

	var filterParts = filter.Split(new[] { ' ' }, StringSplitOptions.RemoveEmptyEntries);
	var i = 0;
	var paramIndex = 0;

	while (i < filterParts.Length)
	{
		string @operator = "", property = "";
		object literal = null;
		var parts = new[] {filterParts[i++], filterParts[i++], filterParts[i++]};

		foreach (var part in parts)
		{
			if (IsStringLiteral(part))
			{
				literal = part.Substring(1, part.Length - 2);
				continue;
			}

			if (IsBoolLiteral(part))
			{
				literal = bool.Parse(part);
				continue;
			}

			// No support for float/double/decimal/... yet
			if (IsNumericLiteral(part))
			{
				literal = int.Parse(part);
				continue;
			}

			var propOrOp = part.ToLowerInvariant();
			if (IsOperator(propOrOp))
			{
				@operator = propOrOp;
				continue;
			}

			property = propOrOp;
		}

		newFilter.Append(UseOperator(@operator, property, paramIndex++));
		paramList.Add(literal);

		// We only support "and" and "or". No ! or () allowed.
		if (i < filterParts.Length)
		{
			newFilter.Append(string.Equals(filterParts[i++], "and", StringComparison.OrdinalIgnoreCase) ? " &amp;&amp; " : " || ");
		}
	}

	return new ParsedFilter(newFilter.ToString(), paramList);
}

private bool IsNumericLiteral(string literal)
{
	var rxNumeric = new Regex(@"^-?\d+(,\d+)*(\.\d+(e\d+)?)?$");
	return rxNumeric.IsMatch(literal);
}

private bool IsBoolLiteral(string literal)
{
	return string.Equals(literal, "true", StringComparison.OrdinalIgnoreCase) ||
		   string.Equals(literal, "false", StringComparison.OrdinalIgnoreCase);
}

private bool IsStringLiteral(string literal)
{
	return literal.StartsWith("\"", StringComparison.Ordinal) &&
		   literal.EndsWith("\"", StringComparison.Ordinal);
}

private bool IsOperator(string @operator)
{
	switch (@operator)
	{
		case "eq":
		case "ne":
		case "lt":
		case "le":
		case "gt":
		case "ge":
		case "sw":
		case "ew":
		case "co":
			return true;
	}

	return false;
}

private string UseOperator(string @operator, string property, int paramIndex)
{
	switch (@operator)
	{
		case "eq":
			return $"{property} = @{paramIndex}";
		case "ne":
			return $"{property} != @{paramIndex}";
		case "lt":
			return $"{property} < @{paramIndex}";
		case "le":
			return $"{property} <= @{paramIndex}";
		case "gt":
			return $"{property} > @{paramIndex}";
		case "ge":
			return $"{property} >= @{paramIndex}";
		case "sw":
			return $"{property}.StartsWith(@{paramIndex})";
		case "ew":
			return $"{property}.EndsWith(@{paramIndex})";
		case "co":
			return $"{property}.Contains(@{paramIndex})";
	}

	return "";
}

private struct ParsedFilter
{
	public ParsedFilter(string filter, IEnumerable<object> paramList)
	{
		Filter = filter;
		ParamList = paramList;
	}

	public string Filter { get; }
	public IEnumerable<object> ParamList { get; }
}
```
