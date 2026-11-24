# Java

Add the latest `io.dapr:dapr-sdk-workflows`, 1.19.1 or later, and put that version in the
`pom.xml` below. The current version and the API are in
[the Java workflow SDK docs](https://docs.dapr.io/developing-applications/sdks/java/java-workflow/).
It needs Java 17 or newer.

```xml
<properties>
  <maven.compiler.release>17</maven.compiler.release>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>
<dependencies>
  <dependency>
    <groupId>io.dapr</groupId>
    <artifactId>dapr-sdk-workflows</artifactId>
    <version>1.19.1</version>
  </dependency>
</dependencies>
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.13.0</version>
    </plugin>
    <plugin>
      <groupId>org.codehaus.mojo</groupId>
      <artifactId>exec-maven-plugin</artifactId>
      <version>3.5.0</version>
      <configuration>
        <mainClass>example.Main</mainClass>
      </configuration>
    </plugin>
  </plugins>
</build>
```

This worker ran against Diagrid with `mvn -q exec:java` and the three connection values
in its environment (`diagrid-develop` section 3). It registers the workflow, starts one
run, waits for it and exits.

```java
package example;

import io.dapr.workflows.Workflow;
import io.dapr.workflows.WorkflowActivity;
import io.dapr.workflows.WorkflowActivityContext;
import io.dapr.workflows.WorkflowStub;
import io.dapr.workflows.client.DaprWorkflowClient;
import io.dapr.workflows.client.WorkflowState;
import io.dapr.workflows.runtime.WorkflowRuntime;
import io.dapr.workflows.runtime.WorkflowRuntimeBuilder;
import java.time.Duration;

public class Main {

  public record Order(String id, int quantity) {}

  public static class OrderWorkflow implements Workflow {
    @Override
    public WorkflowStub create() {
      return ctx -> {
        Order order = ctx.getInput(Order.class);
        String reserved = ctx.callActivity(ReserveActivity.class.getCanonicalName(), order, String.class).await();
        ctx.complete("order " + order.id() + ": " + reserved);
      };
    }
  }

  public static class ReserveActivity implements WorkflowActivity {
    @Override
    public Object run(WorkflowActivityContext ctx) {
      Order order = ctx.getInput(Order.class);
      return "reserved " + order.quantity();
    }
  }

  public static void main(String[] args) throws Exception {
    WorkflowRuntime runtime = new WorkflowRuntimeBuilder()
        .registerWorkflow(OrderWorkflow.class)
        .registerActivity(ReserveActivity.class)
        .build();
    runtime.start(false);

    try (DaprWorkflowClient client = new DaprWorkflowClient()) {
      String runId = client.scheduleNewWorkflow(OrderWorkflow.class, new Order("o-1", 2));
      WorkflowState state = client.waitForWorkflowCompletion(runId, Duration.ofSeconds(60), true);
      System.out.println(runId + " " + state.getRuntimeStatus() + " " + state.readOutputAs(String.class));
    }
    runtime.close();
    System.exit(0);
  }
}
```

## Starting it from outside the app

- **Name:** `registerWorkflow(Class)` registers the class's canonical name, here
  `example.Main.OrderWorkflow`. Pass that name, or register with
  `registerWorkflow("OrderWorkflow", OrderWorkflow.class)` to choose a shorter one.
- **Input:** plain JSON, `{"id": "o-2", "quantity": 3}`. Jackson maps it onto the record.

## Constructs

The orchestrator below shows a branch, a parallel join and an event raced against a
timer, which are the shapes a diagram turns into. This section and the next two are
adapted from `prompts/languages/java.md` in
[diagrid-labs/dapr-skills](https://github.com/diagrid-labs/dapr-skills) (MIT).

```java
import io.dapr.durabletask.Task;
import io.dapr.workflows.Workflow;
import io.dapr.workflows.WorkflowActivity;
import io.dapr.workflows.WorkflowActivityContext;
import io.dapr.workflows.WorkflowStub;
import java.time.Duration;
import java.util.List;

public class OrderFulfilment implements Workflow {
  @Override
  public WorkflowStub create() {
    return ctx -> {
      Order order = ctx.getInput(Order.class);

      boolean valid = ctx.callActivity(
          ValidateOrder.class.getCanonicalName(), order, Boolean.class).await();
      if (!valid) {
        ctx.complete(new OrderResult("rejected"));
        return;
      }

      // implied parallel fan-out and join
      Task<Void> charge = ctx.callActivity(ChargeCard.class.getCanonicalName(), order, Void.class);
      Task<Void> label = ctx.callActivity(PrintShippingLabel.class.getCanonicalName(), order, Void.class);
      ctx.allOf(List.of(charge, label)).await();

      // wait_for_restock: message racing a P7D timer
      Task<RestockNotice> restocked =
          ctx.waitForExternalEvent("restocked", RestockNotice.class);
      Task<Void> deadline = ctx.createTimer(Duration.ofDays(7));
      Task<?> winner = ctx.anyOf(restocked, deadline).await();
      if (winner == deadline) {                       // restock_wait_expired
        ctx.callActivity(CancelOrder.class.getCanonicalName(), order, Void.class).await();
        ctx.complete(new OrderResult("cancelled"));
        return;
      }

      ctx.complete(new OrderResult("shipped"));
    };
  }
}

public class ValidateOrder implements WorkflowActivity {
  @Override
  public Object run(WorkflowActivityContext ctx) {
    Order order = ctx.getInput(Order.class);
    return !order.getLines().isEmpty();
  }
}
```

| Need | Spelling |
| --- | --- |
| Orchestrator | `class X implements Workflow` — an **interface**, not a superclass — with `public WorkflowStub create()` returning a lambda over `ctx` |
| Returning output | `ctx.complete(obj)`. The `WorkflowStub` lambda is `void`; a `return` returns nothing |
| Activity | `class Y implements WorkflowActivity` with `public Object run(WorkflowActivityContext ctx)` |
| Call an activity | `ctx.callActivity(String name, Object input, WorkflowTaskOptions options, Class<V> returnType)`, plus five shorter overloads |
| Retry | `new WorkflowTaskOptions(WorkflowTaskRetryPolicy.newBuilder().setMaxNumberOfAttempts(n).setFirstRetryInterval(Duration).build())` — a builder, with no `new` on the policy |
| Timer | `ctx.createTimer(Duration)` or `ctx.createTimer(ZonedDateTime)`. There is **no `Instant` overload** |
| External event | `ctx.waitForExternalEvent(name, Duration, Class<V>)`, plus shorter overloads |
| Wait for all | `ctx.allOf(List<Task<V>>)` returning `Task<List<V>>` — an **instance method**, not static |
| Wait for first | `ctx.anyOf(Task<?>...)` or `ctx.anyOf(List<Task<?>>)` returning `Task<Task<?>>` |
| Child workflow | `ctx.callChildWorkflow(name, input, instanceId, options, Class<V>)`, plus shorter overloads |
| Await | `.await()` on the `Task` |
| Instance id | `ctx.getInstanceId()` |
| Replay-safe now | `ctx.getCurrentInstant()` |
| Replay-safe id | `ctx.newUuid()` |
| Logger | `ctx.getLogger()` |
| Register | `new WorkflowRuntimeBuilder().registerWorkflow(X.class).registerActivity(Y.class)`, then `WorkflowRuntime rt = builder.build(); rt.start();` |
| Start a run | `new DaprWorkflowClient().scheduleNewWorkflow(X.class, input)`, returning the instance id as a `String` |

Import surface: `io.dapr.workflows.{Workflow, WorkflowStub, WorkflowContext,
WorkflowActivity, WorkflowActivityContext, WorkflowTaskOptions,
WorkflowTaskRetryPolicy}`, `io.dapr.workflows.runtime.{WorkflowRuntime,
WorkflowRuntimeBuilder}`, `io.dapr.workflows.client.{DaprWorkflowClient, WorkflowState,
WorkflowRuntimeStatus}`.

## Traps

- **Keep `maven-compiler-plugin` pinned.** Without it Maven 3.8 uses compiler plugin 3.1,
  which ignores `maven.compiler.release`, and the build fails with `Source option 5 is no
  longer supported`.
- **Activity names are canonical names.** `registerActivity(ReserveActivity.class)`
  registers `example.Main.ReserveActivity`, but `ReserveActivity.class.getName()` is
  `example.Main$ReserveActivity` for a nested class. Calling the activity by `getName()`
  fails the run with `No activity task named 'example.Main$ReserveActivity' is
  registered`. Use `getCanonicalName()`, a top-level class, or an explicit name on both
  sides.
- **`Task` comes from `io.dapr.durabletask`, not from `io.dapr.workflows`.** The SDK
  vendors a fork of the durable-task client, so `io.dapr.durabletask.Task`,
  `TaskFailedException`, `TaskCanceledException` and `CompositeTaskFailedException` are
  the types to import. Before 1.14 they lived under `com.microsoft.durabletask`, which is
  why older snippets do not compile.
- **`TaskOrchestration`, `TaskActivity`, `TaskOrchestrationContext` and
  `DurableTaskGrpcWorker` are not the authoring API.** They exist inside the SDK as
  internal adapters. A workflow written against them will not compile against
  `io.dapr.workflows`, and this is the single commonest wrong shape in circulating Java
  Dapr Workflow examples.
- **`ctx.callSubOrchestrator(...)` will not compile.** It is the private delegation
  target of `callChildWorkflow`. Call `callChildWorkflow`.
- **`start()` is on the runtime, not the builder.** `builder.build()` returns a
  `WorkflowRuntime`, which is `AutoCloseable` and which you then `start()`.
- **`ctx.getCurrentInstant()` does exist.** Some circulating guidance claims it does not
  and steers people toward a wall clock instead, which is the exact determinism bug the
  method exists to prevent.

## Serialization

Jackson, everywhere. Data classes need a **default constructor** and getters, or use
records, which Jackson has handled since 2.12 and which suit generated models well:

```java
public record RestockNotice(String sku, int quantity, String receivedAt) {}
```

What breaks: a class with only an all-args constructor, a lambda field, a circular
reference, a nested type that is not itself serializable. And client and worker must
use the same `DataConverter` — if one is customised, both are.
