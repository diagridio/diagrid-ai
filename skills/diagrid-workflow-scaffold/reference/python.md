# Python

Install the latest `dapr[workflow]`, 1.19.0 or later. The current version and the API are in
[the Python workflow SDK docs](https://docs.dapr.io/developing-applications/sdks/python/python-sdk-extensions/python-workflow-ext/).
From 1.19 the workflow SDK ships inside `dapr`, under
`dapr.ext.workflow`. The separate `dapr-ext-workflow` distribution stops at 1.18.3, so do
not add it to a 1.19 project.

```toml
[project]
requires-python = ">=3.10"
dependencies = ["dapr[workflow]>=1.19.0"]
```

This worker ran against Diagrid with the three connection values in its environment
(`diagrid-develop` section 3). It registers the workflow, starts one run, waits for it
and exits.

```python
from dataclasses import dataclass

import dapr.ext.workflow as wf

wfr = wf.WorkflowRuntime()


@dataclass
class Order:
    id: str
    quantity: int


@wfr.workflow(name="order_workflow")
def order_workflow(ctx: wf.DaprWorkflowContext, order: Order):
    reserved = yield ctx.call_activity(reserve, input=order)
    return f"order {order.id}: {reserved}"


@wfr.activity(name="reserve")
def reserve(ctx: wf.WorkflowActivityContext, order: Order) -> str:
    return f"reserved {order.quantity}"


if __name__ == "__main__":
    wfr.start()
    client = wf.DaprWorkflowClient()
    run_id = client.schedule_new_workflow(workflow=order_workflow, input=Order("o-1", 2))
    state = client.wait_for_workflow_completion(run_id, timeout_in_seconds=60)
    print(run_id, state.runtime_status.name, state.serialized_output)
    wfr.shutdown()
```

A long-running worker keeps the process alive after `wfr.start()` instead of scheduling a
run itself.

## Starting it from outside the app

- **Name:** the `name=` in `@wfr.workflow`, here `order_workflow`. Without it, the
  function name.
- **Input:** the SDK turns a JSON object back into an object with attributes only when
  the object carries `"__durabletask_autoobject__": true`. It adds that key itself when
  the app passes a dataclass, as above. `diagrid_start_workflow` sends plain JSON, so the
  workflow gets a `dict`, and `order.id` fails with `'dict' object has no attribute
  'id'`.
  - For a workflow that reads attributes, add the key to the input:
    `{"__durabletask_autoobject__": true, "id": "o-2", "quantity": 3}`.
  - For new code that will be started from outside, take a `dict` and read keys
    (`order["id"]`), which works from every caller.
  - Ints, strings and lists need nothing extra.

## Constructs

The orchestrator below shows a branch, a parallel join and an event raced against a
timer, which are the shapes a diagram turns into. This section and the next two are
adapted from `prompts/languages/python.md` in
[diagrid-labs/dapr-skills](https://github.com/diagrid-labs/dapr-skills) (MIT).

```python
from datetime import timedelta

import dapr.ext.workflow as wf

wfr = wf.WorkflowRuntime()


@wfr.workflow(name="order_fulfilment")
def order_fulfilment(ctx: wf.DaprWorkflowContext, order: dict):
    valid = yield ctx.call_activity(validate_order, input=order)
    if not valid:
        return {"status": "rejected"}

    # gw_in_stock: exclusive, diverging
    in_stock = yield ctx.call_activity(check_inventory, input=order)
    if in_stock:                                   # stock_available
        charge = ctx.call_activity(charge_card, input=order)
        label = ctx.call_activity(print_shipping_label, input=order)
        yield wf.when_all([charge, label])          # implied parallel join
        yield ctx.call_activity(ship_order, input=order)
        return {"status": "shipped"}

    # stock_unavailable
    yield ctx.call_activity(notify_backorder, input=order)
    restocked = ctx.wait_for_external_event("restocked")
    deadline = ctx.create_timer(timedelta(days=7))
    winner = yield wf.when_any([restocked, deadline])
    if winner is deadline:                          # restock_wait_expired
        yield ctx.call_activity(cancel_order, input=order)
        return {"status": "cancelled"}
    return {"status": "retry"}


@wfr.activity(name="validate_order")
def validate_order(ctx: wf.WorkflowActivityContext, order: dict) -> bool:
    return bool(order.get("lines"))
```

| Need | Spelling |
| --- | --- |
| Orchestrator | `@wfr.workflow(name=...)` on a **generator** function taking `(ctx, input)` |
| Activity | `@wfr.activity(name=...)` on a function taking `(ctx, input)` |
| Call an activity | `ctx.call_activity(fn, input=..., retry_policy=...)` — everything after the first argument is keyword-only |
| Retry | `wf.RetryPolicy(first_retry_interval=, max_number_of_attempts=, backoff_coefficient=, max_retry_interval=, retry_timeout=)` |
| Timer | `ctx.create_timer(timedelta(...))`, or a `datetime` to fire at |
| External event | `ctx.wait_for_external_event(name)` |
| Wait for all | `wf.when_all([...])` — **module-level**, not on the context |
| Wait for first | `wf.when_any([...])` — module-level |
| Child workflow | `ctx.call_child_workflow(fn, input=, instance_id=, retry_policy=)` |
| Instance id | `ctx.instance_id` |
| Replay-safe now | `ctx.current_utc_datetime` |
| Am I replaying | `ctx.is_replaying` |
| Register | the decorators, or `wfr.register_workflow(fn, name=)` / `wfr.register_activity(fn, name=)` |
| Run the worker | `wfr.start()`, `wfr.shutdown()` |
| Start a run | `wf.DaprWorkflowClient().schedule_new_workflow(fn, input=, instance_id=)` |

Import surface, all from `dapr.ext.workflow`: `WorkflowRuntime`,
`DaprWorkflowClient`, `DaprWorkflowContext`, `WorkflowActivityContext`, `RetryPolicy`,
`when_all`, `when_any`, `WorkflowState`, `WorkflowStatus`.

## Traps

**Every await is a `yield`.** A `call_activity` whose result is never yielded returns a
task object, and comparing a task to a boolean is truthy — so a gateway generated
without the `yield` takes the same branch every time and nothing errors.

**`when_any` returns the winning task, so compare by identity.** Keep a reference to
each task before racing them, then test `winner is deadline`. Comparing results instead
breaks the moment two branches can return the same value.

**Race an event against a timer with `when_any`.** `wait_for_external_event` also
takes `timeout=` and raises `TimeoutError`, but the `when_any` over the event and a
`create_timer`, as above, tells the two outcomes apart without an exception.

**The activity context is not the workflow context.** It exposes `ctx.workflow_id` and
`ctx.task_id`, and there is no `instance_id` on it. Key idempotency on
`ctx.workflow_id` plus the activity name.

## Serialization

Activity inputs and outputs go through JSON. Dataclasses and Pydantic models are fine
if every field is JSON-representable; keep timestamps as ISO 8601 strings rather than
`datetime`, and keep anything with a socket, a lock or a file handle out of the payload
entirely. If a `data_object` in the IR has no obvious JSON shape, generate a `TypedDict`
or a dataclass for it and say in the summary that the field types were inferred.
