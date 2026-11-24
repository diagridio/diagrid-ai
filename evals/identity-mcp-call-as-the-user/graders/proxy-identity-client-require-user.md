---
type: llm
focus: last_message
---
A Go agent on Diagrid calls the MCP server `crm`, and the tool must know which signed-in
user asked. Diagrid passes the user on only when the call goes through the MCP proxy with
the caller's token, the server's access policy requires a user, and the server uses the
SPIFFE JWT auth profile.

PASS if the response does all of these:
- verifies the inbound caller with go-ai's `identity.Middleware`;
- builds the MCP client's HTTP client with `identity.NewHTTPClient` (or
  `identity.Transport`), and also sends `dapr-api-token`;
- calls `$DAPR_HTTP_ENDPOINT/v1.0/diagrid/mcp/crm` with the inbound request's context,
  not `context.Background()`;
- turns on `requireUser` on the `crm` access policy;
- uses the SPIFFE JWT auth profile on the `crm` MCPServer, or checks that it already does.

FAIL if the response does any of these:
- calls the CRM's own upstream URL directly;
- sets the `X-Diagrid-User-Token` header by hand, or passes the user's identity as a tool
  argument for the CRM to trust;
- uses go-ai's runner, `agent` or `mcp` package for this call;
- decodes or verifies the token by hand in the agent;
- asks the user to paste a token.
