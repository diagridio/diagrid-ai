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
