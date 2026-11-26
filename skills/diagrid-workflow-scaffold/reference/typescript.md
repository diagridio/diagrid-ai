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

## Constructs

The orchestrator below shows a branch, a parallel join and an event raced against a
timer, which are the shapes a diagram turns into. This section and the next two are
adapted from `prompts/languages/javascript.md` in
[diagrid-labs/dapr-skills](https://github.com/diagrid-labs/dapr-skills) (MIT).

```ts
import {
  DaprWorkflowClient,
  WorkflowActivityContext,
  WorkflowContext,
  WorkflowRuntime,
  TWorkflow,
} from "@dapr/dapr";

const validateOrder = async (ctx: WorkflowActivityContext, order: Order): Promise<boolean> =>
  order.lines.length > 0;

const orderFulfilment: TWorkflow = async function* (ctx: WorkflowContext, order: Order): any {
  const valid = yield ctx.callActivity(validateOrder, order);
  if (!valid) {
    return { status: "rejected" };
  }

  // implied parallel fan-out and join
  const charge = ctx.callActivity(chargeCard, order);
  const label = ctx.callActivity(printShippingLabel, order);
  yield ctx.whenAll([charge, label]);
  yield ctx.callActivity(shipOrder, order);

  // wait_for_restock: message racing a P7D timer
  const restocked = ctx.waitForExternalEvent("restocked");
  const deadline = ctx.createTimer(new Date(ctx.getCurrentUtcDateTime().getTime() + 7 * 86_400_000));
  const winner = yield ctx.whenAny([restocked, deadline]);
  if (winner === deadline) {                          // restock_wait_expired
    yield ctx.callActivity(cancelOrder, order);
    return { status: "cancelled" };
  }

  return { status: "shipped" };
};
```

| Need | Spelling |
| --- | --- |
| Orchestrator | `const wf: TWorkflow = async function* (ctx, input) { ... }` |
| Activity | `(ctx: WorkflowActivityContext, input: TIn) => TOut`, sync or async |
| Call an activity | `ctx.callActivity(activityFnOrName, input?)` — that is the whole signature |
| Retry | **there is no retry parameter.** See below |
| Timer | `ctx.createTimer(fireAt: Date)`, or a bare `number` meaning **seconds** |
| External event | `ctx.waitForExternalEvent(name)`, then `task.getResult()` for the payload |
| Wait for all | `ctx.whenAll(tasks)` — a **method on the context** |
| Wait for first | `ctx.whenAny(tasks)` |
| Child workflow | `ctx.callChildWorkflow(wfOrName, input?, instanceId?)`. `callSubWorkflow` is deprecated |
| Instance id | `ctx.getWorkflowInstanceId()` |
| Replay-safe now | `ctx.getCurrentUtcDateTime()` |
| Am I replaying | `ctx.isReplaying()` |
| Register | `const runtime = new WorkflowRuntime(); runtime.registerWorkflow(wf); runtime.registerActivity(fn); await runtime.start();` |
| Start a run | `await new DaprWorkflowClient().scheduleNewWorkflow(wf, input, instanceId)` |

Every context accessor is a **method**, not a property. `ctx.instanceId` is `undefined`,
silently, and an idempotency key built from it is the string `"undefined"`.

## Traps

- `npm start` does not load `.env`. Use `node --env-file=.env dist/main.js`, or put the
  values in the process's environment another way (`diagrid-develop` section 3).
- Build before `npm start`; it runs the compiled `dist/`.

## The trap that matters more than the rest

**The orchestrator must be `async function*`. A plain `function*` fails silently.**

The executor checks for `Symbol.asyncIterator`. A synchronous generator does not have
one, so it takes the other branch: the generator *object* is serialized as the
workflow's output and the instance is marked **COMPLETED without running a single
activity**. No error, no failed run, no empty status — a green workflow that did
nothing.

Guidance in circulation says the opposite, mandating plain `function*`. It is wrong, and
it produces exactly this. Generate `async function*` and nothing else.

## No retry policy

`callActivity` has no options argument, so a retry from the diagram becomes an explicit
loop in the orchestrator with a timer between attempts:

```ts
let attempt = 0;
while (true) {
  try {
    return yield ctx.callActivity(chargeCard, order);
  } catch (err) {
    if (++attempt >= 3) throw err;
    yield ctx.createTimer(new Date(ctx.getCurrentUtcDateTime().getTime() + attempt * 5_000));
  }
}
```

That is deterministic — the attempt counter is workflow-local state and the delay comes
from the replay-safe clock — but it is code, so it needs the bound written down. Take
the bound from the diagram if it says one, and state the bound you chose if it does not.

## Imports

Import from `@dapr/dapr` and nowhere else. `@dapr/durabletask-js` is a declared
dependency and the authoring surface resolves through a vendored copy inside
`@dapr/dapr`, so importing it directly gets you a second, unrelated copy of the runtime
types. There is no `@dapr/workflow` package.

## Serialization

JSON, both directions. Interfaces and plain object types are ideal for the models
generated from `data_object` records. `Date` becomes a string and does not come back a
`Date`, so keep timestamps as ISO 8601 strings in the payload types and parse at the
edges. `Map`, `Set`, `BigInt` and class instances with methods do not survive.
