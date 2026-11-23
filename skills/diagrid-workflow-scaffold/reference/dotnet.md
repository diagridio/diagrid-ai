# .NET

Add `Dapr.Workflow` 1.19.1. It brings `Microsoft.Extensions.Hosting` 10.0.8 with it; a
direct reference to an older Hosting version fails the restore with `NU1605`, so leave
Hosting out or match it.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Dapr.Workflow" Version="1.19.1" />
  </ItemGroup>
</Project>
```

This worker ran against Diagrid on `net10.0` with the .NET 10 SDK, and on `net9.0` with
the .NET 9 SDK, with the three connection values in its environment (`diagrid-develop`
section 3). It registers the workflow, starts one run, waits for it and exits.

```csharp
using Dapr.Workflow;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddDaprWorkflow(options =>
{
    options.RegisterWorkflow<OrderWorkflow>();
    options.RegisterActivity<ReserveActivity>();
});
using var host = builder.Build();
await host.StartAsync();

var client = host.Services.GetRequiredService<DaprWorkflowClient>();
var runId = await client.ScheduleNewWorkflowAsync(nameof(OrderWorkflow), input: new Order("o-1", 2));
var state = await client.WaitForWorkflowCompletionAsync(runId);
Console.WriteLine($"{runId} {state.RuntimeStatus} {state.ReadOutputAs<string>()}");
await host.StopAsync();

public record Order(string Id, int Quantity);

public sealed class OrderWorkflow : Workflow<Order, string>
{
    public override async Task<string> RunAsync(WorkflowContext context, Order order)
    {
        var reserved = await context.CallActivityAsync<string>(nameof(ReserveActivity), order);
        return $"order {order.Id}: {reserved}";
    }
}

public sealed class ReserveActivity : WorkflowActivity<Order, string>
{
    public override Task<string> RunAsync(WorkflowActivityContext context, Order order) =>
        Task.FromResult($"reserved {order.Quantity}");
}
```

## Starting it from outside the app

- **Name:** the class name, here `OrderWorkflow`.
- **Input:** plain JSON, `{"id": "o-2", "quantity": 3}`. System.Text.Json maps it onto
  the record.

## Traps

- **The SDK must support the target framework.** A project on `net10.0` needs the .NET 10
  SDK; with only the .NET 9 SDK the build fails with `NETSDK1045`. Run
  `dotnet --list-sdks` first, and look in `~/.dotnet` too, where `dotnet-install.sh`
  puts SDKs off the `PATH`. If the right SDK is missing, ask the user whether to install
  it or retarget the project. Do not install an SDK or change the target framework
  without asking.
- On the .NET 9 SDK the build warns `CS9057` for the Dapr.BinaryStore analyzer. The
  worker still runs.
- **Registration:** 1.19.1 registers every `Workflow` and `WorkflowActivity` class on its
  own. With the explicit `Register*` calls above it logs `Workflow 'OrderWorkflow' is
  already registered` at startup. Both forms run; the warning is harmless.
- `Call canceled by the client` in the log after the run is the worker stopping.
