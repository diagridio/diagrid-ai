# .NET

Add the latest `Dapr.Workflow`, 1.19.1 or later, with `dotnet add package Dapr.Workflow`. The
current version and the API are in
[the .NET workflow SDK docs](https://docs.dapr.io/developing-applications/sdks/dotnet/dotnet-workflow/).
1.19.1 brings `Microsoft.Extensions.Hosting` 10.0.8 with it; a
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

## Constructs

The orchestrator below shows a branch, a parallel join and an event raced against a
timer, which are the shapes a diagram turns into. This section and the next two are
adapted from `prompts/languages/dotnet.md` in
[diagrid-labs/dapr-skills](https://github.com/diagrid-labs/dapr-skills) (MIT).

```csharp
using Dapr.Workflow;

internal sealed class OrderFulfilment : Workflow<Order, OrderResult>
{
    public override async Task<OrderResult> RunAsync(WorkflowContext context, Order order)
    {
        var valid = await context.CallActivityAsync<bool>(nameof(ValidateOrder), order);
        if (!valid)
        {
            return new OrderResult("rejected");
        }

        // gw_in_stock: exclusive, diverging
        var inStock = await context.CallActivityAsync<bool>(nameof(CheckInventory), order);
        if (inStock)                                     // stock_available
        {
            // implied parallel fan-out and join
            var charge = context.CallActivityAsync(nameof(ChargeCard), order);
            var label = context.CallActivityAsync(nameof(PrintShippingLabel), order);
            await Task.WhenAll(charge, label);
            await context.CallActivityAsync(nameof(ShipOrder), order);
            return new OrderResult("shipped");
        }

        // stock_unavailable
        await context.CallActivityAsync(nameof(NotifyBackorder), order);
        try
        {
            await context.WaitForExternalEventAsync<RestockNotice>(
                "restocked", TimeSpan.FromDays(7));
            return new OrderResult("retry");
        }
        catch (TaskCanceledException)                    // restock_wait_expired
        {
            await context.CallActivityAsync(nameof(CancelOrder), order);
            return new OrderResult("cancelled");
        }
    }
}

internal sealed class ValidateOrder : WorkflowActivity<Order, bool>
{
    public override Task<bool> RunAsync(WorkflowActivityContext context, Order order)
        => Task.FromResult(order.Lines.Count > 0);
}
```

| Need | Spelling |
| --- | --- |
| Orchestrator | `class X : Workflow<TIn, TOut>`, override `RunAsync(WorkflowContext, TIn)` |
| Activity | `class Y : WorkflowActivity<TIn, TOut>`, override `RunAsync(WorkflowActivityContext, TIn)` |
| Call an activity | `CallActivityAsync<T>(string name, object? input = null, WorkflowTaskOptions? options = null)` |
| Retry | `new WorkflowTaskOptions(RetryPolicy: new WorkflowRetryPolicy(maxNumberOfAttempts, firstRetryInterval, backoffCoefficient, maxRetryInterval, retryTimeout))` |
| Timer | `CreateTimer(TimeSpan delay)`, or `CreateTimer(DateTime fireAt, CancellationToken)` |
| External event | `WaitForExternalEventAsync<T>(name, TimeSpan timeout)`, or `(name, CancellationToken)` |
| Wait for all / first | plain TPL `Task.WhenAll` / `Task.WhenAny`. There is no SDK equivalent and none is needed |
| Child workflow | `CallChildWorkflowAsync<T>(name, input, new ChildWorkflowTaskOptions(InstanceId: ..., RetryPolicy: ..., TargetAppId: ...))` |
| Instance id | `context.InstanceId` |
| Replay-safe now | `context.CurrentUtcDateTime` |
| Replay-safe id | `context.NewGuid()` |
| Am I replaying | `context.IsReplaying` |
| Logger | `context.CreateReplaySafeLogger<T>()` — returns `ILogger`, **not** `ILogger<T>` |
| Register | `builder.Services.AddDaprWorkflow(options => { options.RegisterWorkflow<OrderFulfilment>(); options.RegisterActivity<ValidateOrder>(); });` |
| Start a run | `DaprWorkflowClient.ScheduleNewWorkflowAsync(name, instanceId, input)` |

## Traps

- **The SDK must support the target framework.** A project on `net10.0` needs the .NET 10
  SDK; with only the .NET 9 SDK the build fails with `NETSDK1045`. Run
  `dotnet --list-sdks` first, and look in `~/.dotnet` too, where `dotnet-install.sh`
  puts SDKs off the `PATH`. If the right SDK is missing, ask the user whether to install
  it or retarget the project. Do not install an SDK or change the target framework
  without asking.
- On the .NET 9 SDK the build warns `CS9057` for the Dapr.BinaryStore analyzer. The
  worker still runs.
- **Register explicitly.** The SDK also registers every `Workflow` and
  `WorkflowActivity` class on its own, and a parameterless `AddDaprWorkflow()` works.
  Every official sample still writes `RegisterWorkflow<T>()` and `RegisterActivity<T>()`,
  and neither is obsolete. With both, the worker logs `Workflow 'OrderWorkflow' is
  already registered` at startup; the warning is harmless.
- `Call canceled by the client` in the log after the run is the worker stopping.
- **A workflow class cannot take constructor injection.** The SDK ships an analyzer,
  DAPR1305, that fails the build for it. Activities may inject freely, which is where the
  dependency belongs anyway — the workflow orchestrates, the activity talks to things.
  Generating a workflow with an injected `HttpClient` is a build error, and generating one
  with an injected repository would have been a determinism bug if it had compiled.
- **`CreateTimer(DateTime)` has no default cancellation token.** `CreateTimer(fireAt)`
  does not compile; `CreateTimer(TimeSpan)` does. Prefer the `TimeSpan` overload, which is
  also what an ISO 8601 duration from the IR converts to directly.
- **`WaitForExternalEventAsync` with a timeout throws on expiry.** It raises
  `TaskCanceledException`, so the timeout branch of an event-or-timer wait is a `catch`,
  not a return value. Catch that specific type — a bare `catch` swallows genuine activity
  failures into the timeout path.

## Serialization

System.Text.Json across every boundary. Records are ideal for the models generated from
`data_object` records. Keep the property names stable — renaming one after instances are
in flight breaks their replay just as surely as renaming an activity does.
