# Go

Package: the `identity` package of `github.com/diagridio/go-ai`, v0.2.0 or later. It does
not exist in earlier versions. Get the latest, plus the MCP Go SDK for the client:

```bash
go get github.com/diagridio/go-ai@latest
go get github.com/modelcontextprotocol/go-sdk
```

go-ai v0.2.0 declares Go 1.26.4, so the module needs at least that.

Import only `github.com/diagridio/go-ai/identity`. go-ai's runner, its `agent` package
and its `mcp` package all run tool calls as Dapr Workflows, after the request has ended,
so the caller cannot reach them.

## Verify the caller

```go
import "github.com/diagridio/go-ai/identity"

handler := identity.Middleware(identity.OAuthConfig{})(mux)   // everything is discovered

mux.HandleFunc("POST /agent/run", func(w http.ResponseWriter, r *http.Request) {
	user, ok := identity.UserFromContext(r.Context())
	...
})
```

`VerifiedUser` has `Subject`, `Tenant`, `Scopes`, `Claims` and `IssuerID`. `RequireAuth`
is a `*bool`, and nil means true. To let tokenless requests through, for health probes,
set it to a pointer to `false` and check `ok`.

## Call MCP tools as the caller

The identity transport reads the caller from the request's `context.Context`. Wrap the
transport that adds the sidecar token, not the other way round:

```go
import "github.com/modelcontextprotocol/go-sdk/mcp"

func newMCPHTTPClient() *http.Client {
	return identity.NewHTTPClient(&http.Client{
		Transport: apiTokenTransport{token: os.Getenv("DAPR_API_TOKEN")},
	})
}

type apiTokenTransport struct{ token string }

func (t apiTokenTransport) RoundTrip(req *http.Request) (*http.Response, error) {
	clone := req.Clone(req.Context())   // a RoundTripper must not modify its request
	clone.Header.Set("dapr-api-token", t.token)
	return http.DefaultTransport.RoundTrip(clone)
}

// ctx must be the inbound request's context.
func accountSummary(ctx context.Context, client *http.Client, accountID string) (string, error) {
	transport := &mcp.StreamableClientTransport{
		Endpoint:             strings.TrimRight(os.Getenv("DAPR_HTTP_ENDPOINT"), "/") + "/v1.0/diagrid/mcp/crm-mcp",
		HTTPClient:           client,
		DisableStandaloneSSE: true,
	}
	session, err := mcp.NewClient(&mcp.Implementation{Name: "my-agent", Version: "0.1.0"}, nil).
		Connect(ctx, transport, nil)
	if err != nil {
		return "", err
	}
	defer session.Close()
	result, err := session.CallTool(ctx, &mcp.CallToolParams{
		Name:      "account_summary",
		Arguments: map[string]any{"account_id": accountID},
	})
	...
}
```

## Traps

- A session or request built from `context.Background()` goes out with no user. Pass
  `r.Context()` all the way down, and build the MCP session per request.
- Without a Diagrid connection the first request with a token answers 503
  `oauth.not_configured`. Start the app with its connection and tunnel, as
  `diagrid-app-tunnels` describes.

Reference: the `agents/langchaingo/enterprise-identity` quickstart in
`diagridio/catalyst-quickstarts`.
