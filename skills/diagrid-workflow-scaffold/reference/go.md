# Go

Add `github.com/dapr/go-sdk` v1.16.0 and let it choose `github.com/dapr/durabletask-go`:

```
go get github.com/dapr/go-sdk@v1.16.0
go mod tidy
```

go-sdk v1.16.0 requires durabletask-go v0.15.1 and Go 1.26.10. With an older Go
installed, `go get` downloads 1.26.10 on its own (`switching to go1.26.11`) and raises the
`go` line in `go.mod`. Say so to the user; it is not an error.

Do not pin durabletask-go to an older version before adding go-sdk. An older pin makes
`go get github.com/dapr/go-sdk@latest` fail, and the easy wrong fix is to downgrade
go-sdk instead.

Authoring is `github.com/dapr/durabletask-go/workflow`; the connection comes from
`github.com/dapr/go-sdk/client`. The `workflow` package that go-sdk had from v1.10.0 to
v1.13.0 was removed in v1.14.0, so stale samples that import it from go-sdk do not build.

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
