# Go

Add the latest `github.com/dapr/go-sdk`, v1.16.0 or later, and let it choose
`github.com/dapr/durabletask-go`. The Go SDK has no workflow page of its own; the Go tab of
[the workflow authoring guide](https://docs.dapr.io/developing-applications/building-blocks/workflow/howto-author-workflow/)
shows the current API.

```
go get github.com/dapr/go-sdk@latest
go mod tidy
```

go-sdk v1.16.0 requires durabletask-go v0.15.1 and Go 1.26.10. With an older Go
installed, `go get` downloads 1.26.10 on its own (`switching to go1.26.11`) and raises the
`go` line in `go.mod`. Say so to the user; it is not an error.

Do not pin durabletask-go to an older version before adding go-sdk. An older pin makes
`go get github.com/dapr/go-sdk@latest` fail, and the easy wrong fix is to downgrade
go-sdk instead.

Authoring is `github.com/dapr/durabletask-go/workflow`; the connection comes from
`github.com/dapr/go-sdk/client`. Do not import the `workflow` package from go-sdk; see
Traps.

This worker ran against Diagrid with the three connection values in its environment
(`diagrid-develop` section 3). It registers the workflow, starts one run, waits for it
and exits.

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/dapr/durabletask-go/workflow"
	"github.com/dapr/go-sdk/client"
)

type Order struct {
	ID       string `json:"id"`
	Quantity int    `json:"quantity"`
}

func OrderWorkflow(ctx *workflow.WorkflowContext) (any, error) {
	var order Order
	if err := ctx.GetInput(&order); err != nil {
		return nil, err
	}
	var reserved string
	if err := ctx.CallActivity(Reserve, workflow.WithActivityInput(order)).Await(&reserved); err != nil {
		return nil, err
	}
	return fmt.Sprintf("order %s: %s", order.ID, reserved), nil
}

func Reserve(ctx workflow.ActivityContext) (any, error) {
	var order Order
	if err := ctx.GetInput(&order); err != nil {
		return nil, err
	}
	return fmt.Sprintf("reserved %d", order.Quantity), nil
}

func main() {
	r := workflow.NewRegistry()
	if err := r.AddWorkflow(OrderWorkflow); err != nil {
		log.Fatal(err)
	}
	if err := r.AddActivity(Reserve); err != nil {
		log.Fatal(err)
	}

	wf, err := client.NewWorkflowClient()
	if err != nil {
		log.Fatal(err)
	}
	ctx := context.Background()
	if err := wf.StartWorker(ctx, r); err != nil {
		log.Fatal(err)
	}

	id, err := wf.ScheduleWorkflow(ctx, "OrderWorkflow", workflow.WithInput(Order{ID: "o-1", Quantity: 2}))
	if err != nil {
		log.Fatal(err)
	}
	waitCtx, cancel := context.WithTimeout(ctx, time.Minute)
	defer cancel()
	meta, err := wf.WaitForWorkflowCompletion(waitCtx, id)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(id, meta.String(), meta.Output.GetValue())
}
```

## Starting it from outside the app

- **Name:** the function name, here `OrderWorkflow`. Use `AddWorkflowN` to register
  under another name.
- **Input:** plain JSON, `{"id": "o-2", "quantity": 3}`. `GetInput` unmarshals it with
  the struct tags.

## Constructs

The orchestrator below shows a branch, a parallel join and an event raced against a
timer, which are the shapes a diagram turns into. This section and the next two are
adapted from `prompts/languages/go.md` in
[diagrid-labs/dapr-skills](https://github.com/diagrid-labs/dapr-skills) (MIT).

```go
import (
	"time"

	"github.com/dapr/durabletask-go/workflow"
	dapr "github.com/dapr/go-sdk/client"
)

func OrderFulfilment(ctx *workflow.WorkflowContext) (any, error) {
	var order Order
	if err := ctx.GetInput(&order); err != nil {
		return nil, err
	}

	var valid bool
	if err := ctx.CallActivity(ValidateOrder,
		workflow.WithActivityInput(order)).Await(&valid); err != nil {
		return nil, err
	}

	// implied parallel fan-out: create both tasks BEFORE awaiting either
	charge := ctx.CallActivity(ChargeCard, workflow.WithActivityInput(order))
	label := ctx.CallActivity(PrintShippingLabel, workflow.WithActivityInput(order))
	if err := charge.Await(nil); err != nil {
		return nil, err
	}
	if err := label.Await(nil); err != nil {
		return nil, err
	}

	// wait_for_restock: a message event with a P7D timer, expressed as the timeout
	var restock RestockNotice
	if err := ctx.WaitForExternalEvent("restocked", 7*24*time.Hour).Await(&restock); err != nil {
		// restock_wait_expired
		if err := ctx.CallActivity(CancelOrder,
			workflow.WithActivityInput(order)).Await(nil); err != nil {
			return nil, err
		}
		return OrderResult{Status: "cancelled"}, nil
	}
	return OrderResult{Status: "shipped"}, nil
}

func ValidateOrder(ctx workflow.ActivityContext) (any, error) {
	var order Order
	if err := ctx.GetInput(&order); err != nil {
		return nil, err
	}
	return len(order.Lines) > 0, nil
}
```

| Need | Spelling |
| --- | --- |
| Orchestrator | `func(ctx *workflow.WorkflowContext) (any, error)` — the input is **not** a parameter |
| Activity | `func(ctx workflow.ActivityContext) (any, error)` — by value, not a pointer |
| Call an activity | `ctx.CallActivity(fn, opts...)` — functional options, not `(name, input)` |
| Activity input | `workflow.WithActivityInput(v)` |
| Retry | `workflow.WithActivityRetryPolicy(&workflow.RetryPolicy{MaxAttempts, InitialRetryInterval, BackoffCoefficient, MaxRetryInterval, RetryTimeout})` |
| Timer | `ctx.CreateTimer(d time.Duration, opts...)` |
| External event | `ctx.WaitForExternalEvent(name string, timeout time.Duration)` |
| Child workflow | `ctx.CallChildWorkflow(fn, workflow.WithChildWorkflowInput(v), workflow.WithChildWorkflowInstanceID(id))` |
| Await anything | `.Await(&out)`, or `.Await(nil)` to discard |
| Instance id | `ctx.ID()` |
| Replay-safe now | `ctx.CurrentTimeUTC()` |
| Am I replaying | `ctx.IsReplaying()` |
| Register | `r := workflow.NewRegistry()`, then `r.AddWorkflow(fn)` / `r.AddActivity(fn)`, or the `...N` variants to name them |
| Run the worker | `c, _ := dapr.NewWorkflowClient()`, then `c.StartWorker(ctx, r)` |
| Start a run | `c.ScheduleWorkflow(ctx, "OrderFulfilment", workflow.WithInput(order))` |

## Traps

**There is no `when_any` and no `when_all`.** The module exports neither, under any
name. Two consequences for generated code:

- A parallel gateway is *create every task, then await each in turn*. Calling `Await`
  on the first task before creating the second silently serialises the branches — the
  workflow is correct and the parallelism is gone, which is the hardest kind of bug to
  see in a diff.
- An event raced against a timer uses the `timeout` argument on
  `WaitForExternalEvent`. A negative duration waits forever; **zero cancels
  immediately**, so never pass a zero you computed. Racing two *events* against each
  other cannot be expressed — say so instead of approximating it.

**Do not import `github.com/dapr/go-sdk/workflow`.** It was real once, in go-sdk
v1.10.0 through v1.13.0, and it was removed in v1.14.0 — which is why stale samples and
even some documentation still say `go get` on it. Code written against it does not
compile now: `ctx.InstanceID()` is `ctx.ID()`, and `worker.RegisterActivity()` is
`registry.AddActivity()`.

**The activity context is passed by value.** `func(ctx *workflow.ActivityContext)` will
not satisfy the registry.

<!-- The removed import path is named here in order to steer away from it, which is the
     opposite of instructing anyone to use it.
     lint-allow-banned: go-sdk/workflow — taught as an import path removed in v1.14.0 -->

## Serialization

Everything crossing an activity or child-workflow boundary is JSON. Design the structs
for it from the start.

```go
type Order struct {
	ID        string   `json:"id"`
	Lines     []Line   `json:"lines"`
	Total     float64  `json:"total"`
	CreatedAt string   `json:"created_at"`  // ISO 8601, not time.Time
	Note      *string  `json:"note,omitempty"`
}
```

What does not survive: `func` fields, `chan`, `sync.Mutex`, anything unexported, and
`time.Time` — which technically marshals but round-trips inconsistently enough to be
worth avoiding in workflow payloads. Tag them `json:"-"` or keep them out of the
payload type. An unexported field simply vanishes, with no error at any point.
