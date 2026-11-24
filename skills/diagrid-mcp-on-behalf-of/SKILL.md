---
name: diagrid-mcp-on-behalf-of
description: Verify which signed-in user is calling an app or agent on Diagrid, and run its MCP tool calls as that user (on-behalf-of). Use for user identity, OBO, X-Diagrid-User-Token, requireUser, or "only the signed-in user may do this".
---

# Verify the caller and call MCP tools as them

Goal: end this skill with the user's app checking who is calling it, and with its MCP tool
calls reaching the tool as that same person. And with proof: one call made as the user,
where the app reports the user and the MCP server reports the same user plus the app
acting for them, and one call that skips Diagrid and is refused. Identity that is only
wired, and never seen arriving at the tool, has not been shown to work.

Diagrid does the security work. On the way in, it verifies the caller's login, exchanges
it for a Diagrid-issued token, and puts that token in the `X-Diagrid-User-Token` header.
On the way out, the MCP proxy mints a new token for each tool call. That token names the
user (`sub`) and the app acting for them (`act.sub`). The app itself only does two things,
both with the Diagrid identity package for its language:

1. **Verify the caller.** Middleware checks `X-Diagrid-User-Token` on every request and
   hands the handler a verified user.
2. **Carry the caller onward.** An identity-aware HTTP client puts the caller's token on
   the MCP client's requests to the proxy.

Never parse, log, store, echo or print the token, and never set the header by hand when
the package does it. Never ask the user to paste a token.

## 1. Pick the language file

Read the project's manifest and load the matching file. Each one has the install line,
the middleware, the MCP client wiring and the traps for that SDK:

| Manifest | Load | Package |
| --- | --- | --- |
| `pyproject.toml`, `requirements.txt` | [reference/python.md](reference/python.md) | `diagrid[identity]` on PyPI |
| `*.csproj` | [reference/dotnet.md](reference/dotnet.md) | `Diagrid.AI.Identity` on NuGet |
| `pom.xml`, `build.gradle` | [reference/java.md](reference/java.md) | `io.diagrid:diagrid-spring-ai-identity` on Maven Central |
| `go.mod` | [reference/go.md](reference/go.md) | `github.com/diagridio/go-ai/identity` |
| `package.json` | [reference/typescript.md](reference/typescript.md) | `@diagrid/agent-core` on npm |

Rust has no identity package. Say so and stop. Do not write a token check by hand to
fill the gap.

## 2. Know what carries the user and what does not

The caller's token lives in request-scoped context while the inbound request is being
handled. Anything the handler sends with the identity-aware client, while that request is
still open, carries the user:

- **MCP tool calls through the proxy**, at `$DAPR_HTTP_ENDPOINT/v1.0/diagrid/mcp/<server>`.
  This is the main use.
- **Calls to sub-agents** through the app's own sidecar.

These do not carry the user. Say so plainly when the user asks for one, and do not wire
something that looks like it works:

- **A tool call that runs as a workflow activity.** A durable agent runner, the
  `diagrid-spring-ai-starter`, and go-ai's runner, `agent` and `mcp` packages all run
  tool calls as Dapr Workflow steps, after the inbound request has ended. There is no
  caller left to carry. Identity works on the request path only.
- **A call straight to the MCP server's own URL.** Only the proxy mints the user's token.
  A direct call reaches the server with no user.
- **Third-party APIs.** Call those with a plain HTTP client, so the user's Diagrid token
  is never sent outside Diagrid.
- **Human-in-the-loop approval.** It needs a workflow that can wait, so it is not
  available on this path.

A call with no user behind it, such as a cron, pub/sub or scheduled trigger, has nothing
to carry. The client then sends no user header.

## 3. Set up the MCP server so Diagrid passes the user on

Two settings on the server decide whether the tool ever sees a user. Read them first,
with `diagrid_get_mcp_server` and `diagrid_get_access_policy`. Change them with
`diagrid_apply`, after `diagrid_get_resource_schema` for the kind. A changed manifest is
sent back whole, because a field left out is dropped. Say what will change and get the
user's agreement first.

**The `MCPServer` auth profile must be SPIFFE JWT.** Only that profile lets Diagrid mint
the user's token for the server. The other profiles send an app credential with no user
in it. `scopes` lists the App IDs allowed to call the server:

```yaml
apiVersion: dapr.io/v1alpha1
kind: MCPServer
metadata:
  name: crm-mcp
spec:
  endpoint:
    streamableHTTP:
      url: https://crm.example.com/mcp
      auth:
        spiffe:
          jwt:
            header: Authorization
            headerValuePrefix: "Bearer "
scopes:
  - identity-agent
```

A server whose upstream runs on the user's machine reports `error` until its tunnel
opens. That is expected, not a failed apply.

**The `MCPServerAccessPolicy` must require a user.** By default Diagrid removes the
user's token before forwarding, and the tool sees no user. Requiring one turns
propagation on, and also rejects any call that has no user. Diagrid creates the policy
together with the server, with the same name as the server, so a new server must be
listed by `diagrid_get_mcp_server` before its policy can be applied. Read the policy with
`diagrid_get_access_policy`, which also gives its `apiVersion`, keep its existing
`rules`, and add `auth` to its spec:

```yaml
kind: MCPServerAccessPolicy
metadata:
  name: crm-mcp
spec:
  mcpServerRef: crm-mcp
  rules:
    - callers:
        - appID: identity-agent
      grants:
        - capability: tools
          names: ["account_summary"]
  auth:
    spiffeJWT:
      requireUser: true
```

Grant the app its tools with `diagrid_grant_access`, as for any MCP server.

Reading the policy back may not show `auth`, even when the setting took. The proof that
it works is the tool seeing the user, in section 6. A full read of an MCP server or an
App ID can include the App ID's API token. Never show those fields to the user.

Requiring a user applies to every caller of that server, including the user's own cron
jobs. Before you set it, check who the policy's `rules` name. If a tool must serve both
people and jobs, register the same endpoint as two MCP servers: one that requires a user,
granted to the agents a person invokes, and one that does not, granted to the jobs. Keep
the two sets of callers apart, or a caller in both can skip the requirement.

The policy can also list scopes the user's token must carry. Add them only for scopes
the user's identity provider really issues, as the next section explains.

## 4. Wire the app

Follow the language file. In every language:

- **Leave the middleware config empty.** Issuer, audience and signing keys are discovered
  from the sidecar on the first authenticated request. Setting them by hand is the usual
  cause of `oauth.invalid_issuer` and `oauth.invalid_signature`.
- **Do not require a scope by default.** The examples in the SDK READMEs show
  `agent.invoke`, but no Diagrid Cloud login carries it. A user signed in to Diagrid
  Cloud carries `openid`, `profile`, `email` and `offline_access` and nothing else, so
  requiring `agent.invoke` answers 403 `oauth.missing_scope` to everyone. Require a scope
  only when the user names one their identity provider issues.
- **The default rejects any request without a user,** health probes included. Either
  turn off the app health check, or set the option that lets tokenless requests through
  and check for a missing user in the handler.
- **The MCP client needs two headers.** `dapr-api-token: $DAPR_API_TOKEN` says the app
  may use its sidecar. The identity client adds `X-Diagrid-User-Token`, which says whose
  behalf it acts on. Add the first yourself and never the second.
- **Open the MCP session while the inbound request is being handled**, so the caller's
  context is there for every MCP request, the `initialize` handshake included. Never
  connect at startup: there is no caller then. A client object that is shared across
  requests is fine when it reads the caller per request, as each language file shows.

## 5. Read the user on the MCP server

If the user also owns the MCP server, its tools read the user from the token Diagrid
mints. It arrives in `X-Diagrid-User-Token`. `sub` is the end user and `act.sub` is the
app acting for them. It is a standard RS256 JWT with OIDC discovery, so validate it with
the language's JWT library. Do not fold it into the check on the `Authorization` header:
that one proves the request came from Diagrid, and this one says whose permissions to
apply.

## 6. Prove it

The app has to receive requests from Diagrid, so it runs on the user's machine with a
tunnel. Load `diagrid-app-tunnels` for that. Run the MCP server the same way if it is
local too.

**As the user.** No `diagrid_*` tool can call an app as the signed-in user yet. Ask the
user to send one request to the app through Diagrid as themselves. The "Ask the agent who
you are" step of the enterprise-identity quickstart for their language shows the command.
That command prints the request it sent as well, token included, so ask the user to share
only its `Status Code` and `Data` lines. Expect 200, the app reporting the user's subject,
and the MCP server's log reporting that same subject plus the app's identity as
`act.sub`.

**Without Diagrid.** Call the app's local port directly, with no header. Expect `401` and
`{"error":"oauth.missing_token"}`. That proves the app refuses a caller Diagrid did not
verify.

A Diagrid Cloud login works only on Diagrid Cloud. Self-managed BYOC and Enterprise
Server regions trust the customer's own identity provider instead. There the caller sets
`X-Diagrid-User-Token: Bearer <token from their provider>` on the request itself. See
https://docs.diagrid.io/develop/agents/enterprise-identity/custom-idp-tokens.

## Errors

The same `oauth.*` code means a different thing depending on where it shows up. On the
response to the app's own request it came from the middleware. On the response to an
MCP call it came from the server's access policy.

| Code | Where | Meaning and fix |
| --- | --- | --- |
| `oauth.missing_token` 401 | app | No user on the request. Called directly instead of through Diagrid, or a health probe. |
| `oauth.missing_token` 401 | MCP call | The server requires a user and the call had none: a cron or pub/sub trigger, a plain HTTP client instead of the identity one, or a call made after the request ended. |
| `oauth.missing_scope` 403 | either | A required scope is not in the token. For a Diagrid Cloud user, remove the scope. |
| `oauth.invalid_issuer`, `oauth.invalid_audience`, `oauth.invalid_signature` 401 | app | The middleware was given an issuer, audience or JWKS by hand. Remove them. |
| `oauth.not_configured` 503 | app | No Diagrid connection to discover the issuer from. Start the app with its connection and tunnel, not on its own or under plain `dapr run`. |
| `oauth.verifier_unavailable` 503 | either | Signing keys not loaded yet. Retry. |
| `obo.exchange_failed` 502 | MCP call | Diagrid could not mint the user's token. Retry, then raise a support case. |
| `invalid_grant` 401 | before the app | The login's issuer is not trusted in this region, or the login expired. Ask the user to sign in again. |

**The tool ran but saw no user.** The server does not require a user, or its auth profile
is not SPIFFE JWT. See section 3.

Docs: https://docs.diagrid.io/develop/agents/enterprise-identity
