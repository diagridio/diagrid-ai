---
name: identity-mcp-call-as-the-user
description: A Go agent on Diagrid calls an MCP server and the tool must know which signed-in user asked. The answer must call through the MCP proxy with the go-ai identity client, require a user on the server's access policy, and use a SPIFFE JWT auth profile.
tags: [identity, mcp]
plugins: ["../.."]
max_turns: 14
allowed_tools: [Read, Glob, Grep, Skill]
expected_outcome: >
  Loads diagrid-mcp-on-behalf-of. Verifies the caller with identity.Middleware, builds the
  MCP client on identity.NewHTTPClient wrapping a transport that adds dapr-api-token, and
  points it at $DAPR_HTTP_ENDPOINT/v1.0/diagrid/mcp/crm with the request's context. Sets
  requireUser on the crm MCPServerAccessPolicy and a SPIFFE JWT auth profile on the
  MCPServer. Does not call the CRM's own URL and does not set X-Diagrid-User-Token by hand.
---

My Go agent sales-agent runs on Diagrid in project default. It calls the tool
lookup_account on an MCP server I registered there called crm. People talk to the agent,
and the CRM must answer with only the accounts the person asking is allowed to see, so
the CRM has to know who that person is on every call. How do I wire the agent and the
MCP server so the tool knows who asked?
