# TypeScript and JavaScript

Add the latest `@dapr/dapr`, 3.19.0 or later. The current version and the API are in
[the JavaScript workflow SDK docs](https://docs.dapr.io/developing-applications/sdks/js/js-workflow/).
The workflow runtime and client are in the main package; there
is no separate workflow package to install.

```json
{
  "type": "module",
  "scripts": { "build": "tsc", "start": "node dist/main.js" },
  "dependencies": { "@dapr/dapr": "^3.19.0" },
  "devDependencies": { "typescript": "^5.9.0", "@types/node": "^24.0.0" }
}
```

With `"module": "NodeNext"` and `"moduleResolution": "NodeNext"` in `tsconfig.json`, the
top-level `await` below compiles. This worker ran against Diagrid with `npm start` and the
three connection values in its environment (`diagrid-develop` section 3). It registers
the workflow, starts one run, waits for it and exits.

```typescript
import {
  DaprWorkflowClient,
  TWorkflow,
  WorkflowActivityContext,
  WorkflowContext,
  WorkflowRuntime,
  WorkflowRuntimeStatus,
} from "@dapr/dapr";

type Order = { id: string; quantity: number };

const reserve = async (_ctx: WorkflowActivityContext, order: Order): Promise<string> =>
  `reserved ${order.quantity}`;

const orderWorkflow: TWorkflow = async function* (ctx: WorkflowContext, order: Order): any {
  const reserved = yield ctx.callActivity(reserve, order);
  return `order ${order.id}: ${reserved}`;
};

const runtime = new WorkflowRuntime().registerWorkflow(orderWorkflow).registerActivity(reserve);
await runtime.start();

const client = new DaprWorkflowClient();
const runId = await client.scheduleNewWorkflow(orderWorkflow, { id: "o-1", quantity: 2 });
const state = await client.waitForWorkflowCompletion(runId, undefined, 60);
console.log(runId, state && WorkflowRuntimeStatus[state.runtimeStatus], state?.serializedOutput);

await runtime.stop();
await client.stop();
process.exit(0);
```

For plain JavaScript, drop the types and keep the same calls.

## Starting it from outside the app

- **Name:** the function's name, here `orderWorkflow`. An anonymous function has no
  name to start it by, so assign it to a named `const` or declare it with a name.
- **Input:** plain JSON, `{"id": "o-2", "quantity": 3}`. It arrives as a plain object.

## Traps

- `npm start` does not load `.env`. Use `node --env-file=.env dist/main.js`, or put the
  values in the process's environment another way (`diagrid-develop` section 3).
- Build before `npm start`; it runs the compiled `dist/`.
