---
layout: single
title: A way to deal with HTTP error responses
date: 2018-01-13 23:15:00.000000000 +02:00
type: post
tags: ["C#", "Design"]
---

When I read [Ken's blog post](https://kenbonny.net/2018/01/08/constructor-fun) about an issue he had with a certain class in a project he is working on, his solution didn't sit quite right with me. I know that in Ken's case, the code couldn't be improved any further because of dependencies on other bits. Nothing can stop us however from having a second look at this design with a fresh codebase, right?



# v1

We want to wrap HTTP responses into something like `Response<T>`. But, you know, the web is finicky and sometimes stuff breaks somewhere, so we might get a 4xx or 5xx response back. We need to represent these cases as well.

So, without further ado, here is version 1:

```csharp
public class Response<T>
{
    public Response(T value)
    {
        Value = value;
        IsError = false;
    }
    
    public Response(string error)
    {
        Error = error;
        IsError = true;
    }
    
    public T Value { get; }
    public string Error { get; }
    public bool IsError { get; }
}
```

As Ken already mentioned in his post: what if `T` is a `string`? The only way in that case to specify the correct constructor, is by using named parameters:

`var valueResponse = new Response<string>(value: "text");`

This is too confusing for any developer in my opinion: the differentiation between a successful response and an erroneous one is completely lost.

# v2

First of all, let's introduce contracts rather than classes to model the success and error responses. These make it easier to introduce tests in the system and allow for extensibility.

```csharp
public interface IResponse<out T>
{
    int StatusCode { get; }
    T Value { get; }
}

public interface IErrorResponse<out T> : IResponse<T> 
{
    string Error { get; }
    string OriginalData { get; }
    Exception Exception { get; }
    bool HasException { get; }
}
```

In case you're wondering why I opted for an `int` instead of `System.Net.HttpStatusCode` for the property `StatusCode`: not all status code possibilities exist in the enum, like 429 - Too Many Requests. But don't worry, the implementation will allow you to pass in HttpStatusCode's as well.

Let's now implement a class so we can return successful responses from an HTTP service:

```csharp
public class Response<T> : IResponse<T> 
{
    internal Response(HttpStatusCode statusCode, T value)
        : this((int)statusCode, value)
    {
    }
    
    internal Response(int statusCode, T value)
    {
        StatusCode = statusCode;
        Value = value;
    }
    
    public int StatusCode { get; }
    public T Value { get; }
}
```

And for an error response:

```csharp
public class ErrorResponse<T> : IErrorResponse<T>
{
    internal ErrorResponse(HttpStatusCode statusCode, string error)
        : this((int)statusCode, error)
    {
    }
    
    internal ErrorResponse(int statusCode, string error)
    {
        StatusCode = statusCode;
        Error = error;
    }
    
    public int StatusCode { get; }
    public string Error { get; }
    
    public string OriginalData { get; internal set; }
    public Exception Exception { get; internal set; }
    public bool HasException => Exception != null;
    
    T IResponse<T>.Value => default; // Needs C# 7.1
}
```

That last line will hide the `Value` property - which is only interesting when the response was successfull anyway - when you are working with an instance of a class that implements `IErrorResponse<T>`. It will however still show up when you cast to the interface(s), but will be either `null` or the default value for a value type. Note that you could still write `default(T)`, C# 7.1 just provides a slightly shorter way.

## Bringing success and error together: the HTTP service

When retrieving data from an HTTP service, we will either return a `Response<T>` when everything's OK (2xx response) and the data we get back can actually be converted back to what we expect to find (something `T`-ish). If anything else happens, we will return an `ErrorResponse<T>` instead, containing as much data as we can supply to the caller.

```csharp
protected async Task<IResponse<T>> Get<T>(string uri) 
{
    HttpResponseMessage response = null;
    string content = null;

    try
    {
        response = await Client.GetAsync(uri)
            .ConfigureAwait(false); // Client is a static System.Net.HttpClient instance

        content = await response.Content.ReadAsStringAsync()
            .ConfigureAwait(false);

        response.EnsureSuccessStatusCode();
        return CreateResponse<T>(response.StatusCode, content);
    }
    catch (Exception e)
    {
        return new ErrorResponse<T>(response?.StatusCode ?? HttpStatusCode.InternalServerError, e.Message)
        {
            OriginalData = content,
            Exception = e
        };
    }
}

private IResponse<T> CreateResponse<T>(HttpStatusCode statusCode, string json)
{
    try
    {
        var data = JsonConvert.DeserializeObject<T>(json);
        return new Response<T>(statusCode, data);
    }
    catch (Exception e)
    {
        return new ErrorResponse<T>(statusCode, "Couldn't deserialize the received data to the requested type.")
        {
            OriginalData = json,
            Exception = e
        };
    }
}

```

The majority of the `Get<T>` method should be familiar if you've ever called HTTP services using an `HttpClient` instance: 
* We perform a GET /uri and await the response
* We read the returned data as a string. This contains either the actual data (2xx) or information (4xx, 5xx) about what went wrong.
* Using `response.EnsureSuccessStatusCode();`, we check for 2xx responses. If the response was good, we call `CreateResponse<T>`. If not, we end up in the exception handler.
* In the exception handler, we return an `ErrorResponse<T>`

Inside the `CreateResponse<T>` method, we only need to keep in mind that the returned data might not be what we expect it to be, that is something that can be converted into a `T`. So when that happens, we catch the deserialization exception, and return an `ErrorResponse<T>` instead.
I also need to mention that, in the [full sample source code](https://github.com/wcabus/response-errorresponse), I've set a default Accept header to only accept JSON from the HTTP service. It would be rather strange to call `JsonConvert.DeserializeObject<T>` with XML, right?

## Inspecting the result

We now have a way to represent success and error responses, and an HTTP service to get both of these. How can we now call that service and test if we have an error or not?

In the sample, I've provided a dummy CustomerService that uses the HttpService class:

```csharp
public class CustomerService : HttpService
{
    public Task<IResponse<Customer>> GetCustomerById(int id)
    {
        return Get<Customer>($"https://api.myapp.local/customers/{id}");
    }
}
```

When we call that service, we can inspect the response:

```csharp
var client = new CustomerService();
var response = await client.GetCustomerById(404);
if (response is IErrorResponse<Customer> error)
{
    Console.ForegroundColor = ConsoleColor.DarkRed;
    Console.WriteLine($"{error.StatusCode} - {error.Error}");
    if (error.HasException)
    {
        Console.WriteLine(error.Exception);
    }

    Console.ResetColor();
   
    return;
}

Console.WriteLine(response.Value);
```

Using a C# 7.0 feature, pattern matching, we can easily check if `response` is in fact an `IErrorResponse<Customer>`, rather than a successful response. If it is, the `error` instance is immediately available inside the if-block.

If the call to `GetCustomerById` worked just fine, we can access `response.Value` just like we would expect.

## Wrapping it up

So, instead of dealing with both success and error in one class, I've provided two separate contracts to model them. Because `IErrorResponse<T>` implements `IResponse<T>`, we can use the latter as our base return type for all HTTP related methods. 
And thanks to pattern matching, it is now easier than ever to quickly check if the returned response is in fact an error.


Full sample code can be found on [GitHub](https://github.com/wcabus/response-errorresponse). Improvements and further discussions are welcome!