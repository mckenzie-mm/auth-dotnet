# Dotnet Authentication / Authorization

To implement a simple username and password authentication mechanism in ASP.NET Core Web API, the standard approach is to create a custom Basic Authentication Handler. This decodes the Authorization: Basic <base64> header, validates the credentials, and secures endpoints using the standard [Authorize] attribute.

Here is a complete, minimal example for modern .NET 8 / .NET 9.

### 1. Create the Authentication Handler
This class intercepts the HTTP request, extracts the Authorization header, decodes the Base64 credentials (username:password), and verifies them.

Create a file named BasicAuthenticationHandler.cs:

csharp

```cs
using Microsoft.AspNetCore.Authentication;
using Microsoft.Extensions.Options;
using System.Net.Http.Headers;
using System.Security.Claims;
using System.Text;
using System.Text.RegularExpressions;

public class BasicAuthenticationHandler : AuthenticationHandler<AuthenticationSchemeOptions>
{
    public BasicAuthenticationHandler(
        IOptionsMonitor<AuthenticationSchemeOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder) : base(options, logger, encoder) { }

    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        // 1. Check if Authorization header exists
        if (!Request.Headers.ContainsKey("Authorization"))
            return AuthenticateResult.Fail("Missing Authorization Header");

        try
        {
            // 2. Parse and validate the scheme
            var authHeader = AuthenticationHeaderValue.Parse(Request.Headers["Authorization"]!);
            if (!authHeader.Scheme.Equals("Basic", StringComparison.OrdinalIgnoreCase))
                return AuthenticateResult.Fail("Not a Basic Authentication header");

            // 3. Decode the base64 string
            var credentialBytes = Convert.FromBase64String(authHeader.Parameter ?? "");
            var credentials = Encoding.UTF8.GetString(credentialBytes).Split(':', 2);
            
            if (credentials.Length != 2)
                return AuthenticateResult.Fail("Invalid Basic Authentication format");

            var username = credentials[0];
            var password = credentials[1];

            // 4. Validate the credentials (Hardcoded for this simple example)
            if (username == "admin" && password == "SecretPassword123")
            {
                // Create user identity claims upon successful login
                var claims = new[] { new Claim(ClaimTypes.Name, username) };
                var identity = new ClaimsIdentity(claims, Scheme.Name);
                var principal = new ClaimsPrincipal(identity);
                var ticket = new AuthenticationTicket(principal, Scheme.Name);

                return AuthenticateResult.Success(ticket);
            }

            return AuthenticateResult.Fail("Invalid Username or Password");
        }
        catch
        {
            return AuthenticateResult.Fail("Invalid Authorization Header Format");
        }
    }
}
```
### 2. Register the Services

You need to notify your application about the custom authentication scheme in Program.cs.Open Program.cs and add the authentication services before builder.Build():

csharp

```cs
using Microsoft.AspNetCore.Authentication;

var builder = WebApplication.CreateBuilder(args);

// Add Controllers
builder.Services.AddControllers();

// Register Custom Basic Authentication
builder.Services.AddAuthentication("BasicAuthentication")
    .AddScheme<AuthenticationSchemeOptions, BasicAuthenticationHandler>("BasicAuthentication", null);

var app = builder.Build();

app.UseHttpsRedirection();

// Ensure both Authentication and Authorization middlewares are enabled (order matters!)
app.UseAuthentication(); 
app.UseAuthorization();

app.MapControllers();

app.Run();
```

### 3. Protect Your Controllers

Decorate any controller or specific endpoint with the [Authorize] attribute to lock down access.Create a file named DataController.cs:

csharp
```cs
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
[Authorize] // <--- This protects every route inside this controller
public class DataController : ControllerBase
{
    [HttpGet]
    public IActionResult GetSecureData()
    {
        return Ok(new { Message = $"Hello {User.Identity?.Name}, you have successfully authenticated!" });
    }
}
```
### How to Test This in Postman / cURL

1. Send a GET request to https://localhost:<port>/api/data.

2. Without credentials, you will receive a 401 Unauthorized response.
3. Go to the Authorization tab in Postman, select Basic Auth, and enter:

    * Username: admin

    * Password: SecretPassword123

4. Postman automatically builds the header (Authorization: Basic YWRtaW46U2VjcmV0UGFzc3dvcmQxMjM=) and your request will return a 200 OK status code.


