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
