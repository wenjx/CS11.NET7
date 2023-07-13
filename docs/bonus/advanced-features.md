# Implementing advanced features for web services

> This is an optional bonus section for Chapter 15. It is not required to complete the rest of the book.

- [Implementing advanced features for web services](#implementing-advanced-features-for-web-services)
  - [Implementing a Health Check API](#implementing-a-health-check-api)
  - [Implementing Open API analyzers and conventions](#implementing-open-api-analyzers-and-conventions)
  - [Implementing transient fault handling](#implementing-transient-fault-handling)
  - [Adding security HTTP headers](#adding-security-http-headers)
  - [Enabling HTTP/3 support for HttpClient](#enabling-http3-support-for-httpclient)

Now that you have seen the fundamentals of building a web service and then calling it from a client, let's look at some more advanced features.

## Implementing a Health Check API

There are many paid services that perform site availability tests that are basic pings, some with more advanced analysis of the HTTP response.
ASP.NET Core 2.2 and later make it easy to implement more detailed website health checks. For example, your website might be live, but is it ready? Can it retrieve data from its database?

Let's add basic health check capabilities to our web service:

1.	In the `Northwind.WebApi` project, add package references to enable Entity Framework Core database health checks, as shown in the following markup:
```xml
<PackageReference Include="AspNetCore.HealthChecks.SqlServer" 
                  Version="6.0.2" />
<PackageReference Include=  
  "Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore"   
  Version="7.0.0" />
```

> Warning! `AspNetCore.HealthChecks.SqlServer` is not officially supported by Microsoft.

2.	Build the `Northwind.WebApi` project.
3.	In `Program.cs`, before the call to the `Build` method, add a statement to add health checks, including to the Northwind database context, and if you are using SQL Server instead of SQLite, as shown in the following code:

```cs
builder.Services.AddHealthChecks()
  .AddDbContextCheck<NorthwindContext>()
  // execute SELECT 1 using the specified connection string
  .AddSqlServer("Data Source=.;Initial Catalog=Northwind;Integrated Security=true;");
```

By default, the database context check calls EF Core's `IDatabaseCreator.CanConnectAsync` method. You can customize the method that is run by setting its name as a parameter in the `AddDbContextCheck` method.

4.	In `Program.cs`, before the call to `MapControllers`, add a statement to use basic health checks, as shown in the following code:

```cs
app.UseHealthChecks(path: "/howdoyoufeel");
```

5.	Start the `Northwind.WebApi` web service project.
6.	Start Chrome.
7.	Navigate to https://localhost:5002/howdoyoufeel and note that the web service responds with a plain text response: `Healthy`.
8.	If you are using SQL Server, note the SQL statement that was executed to test the health of the database, as shown in the following output:

```
Level: Debug, Event Id: 20100, State: Executing DbCommand [Parameters=[], CommandType='Text', CommandTimeout='30']
SELECT 1
```

9.	Close Chrome and shut down the web server.

## Implementing Open API analyzers and conventions

In this chapter, you learned how to enable Swagger to document a web service by manually decorating a controller class with attributes.

In ASP.NET Core 2.2 or later, there are API analyzers that reflect over controller classes that have been annotated with the `[ApiController]` attribute to document them automatically. The analyzer assumes some API conventions. After installing the analyzers, controllers that have not been properly decorated should have warnings (green squiggles) and warnings when you compile the source code.

To use it, your project must enable the OpenAPI Analyzers:

1.	In the `Northwind.WebApi` project file, add the element that enables the OpenAPI Analyzers to a project that targets the web SDK, as shown highlighted in the following markup:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <IncludeOpenAPIAnalyzers>true</IncludeOpenAPIAnalyzers>
  </PropertyGroup>
```

2.	In `CustomersController.cs`, for the `Update` action method, comment out the three `[ProducesResponseType]` attributes, as shown highlighted in the following code:

```cs
// [ProducesResponseType(204)]
// [ProducesResponseType(400)]
// [ProducesResponseType(404)]
public async Task<IActionResult> Update(
  string id, [FromBody] Customer c)
```

3.	Note the warnings in **Error List** or **PROBLEMS**, as shown in *Figure 15.15*:
 
![Figure 15.15: Warnings from the OpenAPI Analyzers](images/B18856_15_15.png)
*Figure 15.15: Warnings from the OpenAPI Analyzers*

4.	Uncomment the three attributes to remove the warnings.

Automatic code fixes can then add the appropriate `[Produces]` and `[ProducesResponseType]` attributes, although this only currently works in Visual Studio. In Visual Studio Code, you will see warnings about where the analyzer thinks you should add attributes, but you must add them yourself.

## Implementing transient fault handling

When a client app or website calls a web service, it could be from across the other side of the world. Network problems between the client and the server could cause issues that are nothing to do with your implementation code. If a client makes a call and it fails, the app should not just give up. If it tries again, the issue may now have been resolved. We need a way to handle these temporary faults.

To handle these transient faults, Microsoft recommends that you use the third-party library **Polly** to implement automatic retries with exponential backoff. You define a policy, and the library handles everything else.

> **Good Practice**: You can read more about how Polly can make your web services more reliable at the following link: https://docs.microsoft.com/en-us/dotnet/architecture/microservices/implement-resilient-applications/implement-http-call-retries-exponential-backoff-polly

## Adding security HTTP headers
ASP.NET Core has built-in support for common security HTTP headers like HSTS. But there are many more HTTP headers that you should consider implementing.

The easiest way to add these headers is using a middleware class:

1.	In the `Northwind.WebApi` project/folder, create a file named `SecurityHeadersMiddleware.cs` and modify its statements, as shown in the following code:

```cs
using Microsoft.Extensions.Primitives; // StringValues

public class SecurityHeaders
{
  private readonly RequestDelegate next;

  public SecurityHeaders(RequestDelegate next)
  {
    this.next = next;
  }

  public Task Invoke(HttpContext context)
  {
    // add any HTTP response headers you want here
    context.Response.Headers.Add(
      "super-secure", new StringValues("enable"));

    return next(context);
  }
}
```

2.	In `Program.cs`, before the call to `MapControllers`, add a statement to register the middleware, as shown in the following code:

```cs
app.UseMiddleware<SecurityHeaders>();
```

3.	Start the web service.
4.	Start Chrome.
5.	Show Developer tools and its Network tab to record requests and responses.
6.	Navigate to https://localhost:5002/weatherforecast.
7.	Note the custom HTTP response header named `super-secure` that we added, as shown in *Figure 15.16*:

![Figure 15.16: Adding a custom HTTP header named super-secure](images/B18856_15_16.png)
*Figure 15.16: Adding a custom HTTP header named super-secure*

## Enabling HTTP/3 support for HttpClient

In *Chapter 13, Building Websites Using ASP.NET Core Razor Pages*, you learned how to enable HTTP/3 support in the Kestrel web server. Now we will see how to enable HTTP/3 on the client side. First, we need to enable all versions of HTTP in the web service project:

1.	In the `Northwind.WebApi` project/folder, in `Program.cs`, import the namespace for working with HTTP protocols, as shown in the following code:

```cs
using Microsoft.AspNetCore.Server.Kestrel.Core; // HttpProtocols
```

2.	In `Program.cs`, add statements before the call to `Build` to enable all three versions of HTTP, as shown in the following code:

```cs
builder.WebHost.ConfigureKestrel((context, options) =>
{
  options.ListenAnyIP(5002, listenOptions =>
  {
    listenOptions.Protocols = HttpProtocols.Http1AndHttp2AndHttp3;
    listenOptions.UseHttps(); // HTTP/3 requires secure connections
  });
});
```

3.	In `appSettings.json`, add an entry to show hosting diagnostics, as shown highlighted in the following configuration:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.AspNetCore.Hosting.Diagnostics": "Information"
    }
```

4.	In the `Northwind.Mvc` project/folder, add an element to enable HTTP/3 support, as shown in the following markup:

```xml
<ItemGroup>
  <RuntimeHostConfigurationOption 
    Include="System.Net.SocketsHttpHandler.Http3Support"
    Value="true" />
</ItemGroup>
```

5.	In `Program.cs`, import the namespace for setting the HTTP version, as shown in the following code:

```cs
using System.Net; // HttpVersion
```

6.	In `Program.cs`, add statements to the configuration of the HTTP client for the `Northwind.WebApi` service to set the default version to 3.0, and to fall back to earlier versions if HTTP/3 is not supported by the web service, as shown highlighted in the following code:

```cs
builder.Services.AddHttpClient(name: "Northwind.WebApi",
  configureClient: options =>
  {
    options.DefaultRequestVersion = HttpVersion.Version30;
    options.DefaultVersionPolicy = HttpVersionPolicy.RequestVersionOrLower;

    options.BaseAddress = new Uri("https://localhost:5002/");
    ...
  });
```

7.	Start the `Northwind.WebApi` project and confirm that the web service is listening only on port 5002, as shown in the following output:

```
info: Microsoft.Hosting.Lifetime[14]
  Now listening on: https://localhost:5002
```

8.	Start the `Northwind.Mvc` project.
9.	Start Chrome and navigate to https://localhost:5001/.
10.	On the home page, in the customer form, enter a country like `UK`, and then click **Submit**.
11.	In the command line or terminal for the web service listening on port 5002, note that the HTTP requests and responses are being made using HTTP/3, as shown in the following output:

```
info: Microsoft.AspNetCore.Hosting.Diagnostics[1]
      Request starting HTTP/3 GET https://localhost:5002/api/customers/?country=UK - -
```

12.	Close Chrome and shut down the two web servers.
