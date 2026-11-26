# .NET

Package: `Diagrid.AI.Identity` on NuGet, 1.2.0 or later, for net8.0, net9.0 and net10.0.
Add the latest:

```bash
dotnet add package Diagrid.AI.Identity
dotnet add package ModelContextProtocol
```

The package marks its whole API experimental. Without this in the project file, the build
fails:

```xml
<PropertyGroup>
  <NoWarn>$(NoWarn);DGRDID001</NoWarn>
</PropertyGroup>
```

Tell the user the API is experimental and may change outside a major release.

## Verify the caller

```csharp
using Diagrid.AI.Identity;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDiagridIdentity();   // issuer, audience and keys are discovered

var app = builder.Build();
app.UseDiagridIdentity();

app.MapPost("/agent/run", (HttpContext context) =>
{
    var user = context.GetVerifiedUser()!;   // never null while RequireAuth is on
    ...
});
```

`GetVerifiedUser()` returns the user with `Subject`, `Tenant`, `Scopes` and `Claims`. To
let tokenless requests through, for health probes, pass
`AddDiagridIdentity(config => config.RequireAuth = false)` and check for null.

## Call MCP tools as the caller

Register the identity client once, with the sidecar token:

```csharp
builder.Services.AddDiagridIdentityHttpClient(configure: client =>
{
    var daprApiToken = Environment.GetEnvironmentVariable("DAPR_API_TOKEN");
    if (!string.IsNullOrEmpty(daprApiToken))
    {
        client.DefaultRequestHeaders.Add("dapr-api-token", daprApiToken);
    }
});
```

Then build the MCP client on it, inside the request. Call this from the handler above,
which gets `IHttpClientFactory` and `ILoggerFactory` by injection:

```csharp
using ModelContextProtocol.Client;
using ModelContextProtocol.Protocol;

static async Task<string> AccountSummary(
    IHttpClientFactory httpClientFactory, ILoggerFactory loggerFactory, string accountId)
{
    var http = httpClientFactory.CreateClient(
        DiagridIdentityHttpClientServiceCollectionExtensions.DefaultClientName);

    await using var transport = new HttpClientTransport(
        new HttpClientTransportOptions
        {
            Endpoint = new Uri($"{Environment.GetEnvironmentVariable("DAPR_HTTP_ENDPOINT")}/v1.0/diagrid/mcp/crm-mcp"),
            TransportMode = HttpTransportMode.StreamableHttp,
        },
        http,
        loggerFactory,
        ownsHttpClient: false);

    await using var client = await McpClient.CreateAsync(transport, loggerFactory: loggerFactory);
    var result = await client.CallToolAsync(
        "account_summary", new Dictionary<string, object?> { ["account_id"] = accountId });
    if (result.IsError == true)
    {
        throw new InvalidOperationException("account_summary was refused");
    }
    return result.Content.OfType<TextContentBlock>().FirstOrDefault()?.Text ?? "";
}
```

The argument names are the tool's own, from its schema. They are not converted to C#
style.

The handler reads the caller from `AsyncLocal` when each request is sent, so one shared
client is safe under concurrency. `ownsHttpClient: false` keeps the transport from
disposing the factory's client.

## Traps

- `ModelContextProtocol` 2.4 logs `Tool 'account_summary' not found in cache during
  tools/call` when the client calls a tool it has not listed. It is harmless. Call
  `ListToolsAsync` first to silence it.
- The registration turns off `AllowAutoRedirect`, because the handler follows redirects
  itself and drops the user header when a redirect leaves the origin. Do not turn it back
  on.
- Without a Diagrid connection the first request with a token answers 503
  `oauth.not_configured`. Start the app with its connection and tunnel, as
  `diagrid-app-tunnels` describes.

Reference: the `agents/microsoft-dotnet/enterprise-identity` quickstart in
`diagridio/catalyst-quickstarts`.
