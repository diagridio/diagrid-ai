# Python

Package: `diagrid[identity]` on PyPI, 0.5.0 or later, on Python 3.11 to 3.13. Install
the latest:

```bash
pip install "diagrid[identity]"
```

Add it next to a framework extra when the app needs both, for example
`diagrid[langgraph,identity]`. The extra brings the JWT library, `httpx2` and `starlette`.
It brings no server, so the app also needs an ASGI server such as `uvicorn`. For the MCP
client, use `mcp` 2.x, whose `streamable_http_client` takes an `httpx2` client.

The middleware works with any ASGI app: FastAPI, Starlette, or a framework served over
ASGI.

## Verify the caller

```python
from fastapi import FastAPI, Request
from diagrid.identity import OAuthConfig
from diagrid.identity.asgi import OAuthMiddleware, verified_user

app = FastAPI()
app.add_middleware(OAuthMiddleware, config=OAuthConfig())

@app.post("/agent/run")
async def run(request: Request):
    user = verified_user(request)  # never None while require_auth is on
    ...
```

`OAuthConfig()` with no arguments requires a verified user and discovers everything else.
`verified_user(request)` returns a `VerifiedUser` with `subject`, `tenant`, `scopes`,
`claims` and `issuer_id`. Do not return `claims` to the caller. They are a real person's
identity data.

`OAuthConfig(require_auth=False)` lets tokenless requests through, for health probes.
Then `verified_user(request)` can be `None`, so check it.

## Call MCP tools as the caller

```python
import os

from diagrid.identity.http import AsyncClient
from mcp import ClientSession
from mcp.client.streamable_http import streamable_http_client

MCP_URL = f"{os.environ['DAPR_HTTP_ENDPOINT']}/v1.0/diagrid/mcp/crm-mcp"

# One shared client is safe: it reads the caller when each request is sent.
mcp_http = AsyncClient(headers={"dapr-api-token": os.environ.get("DAPR_API_TOKEN", "")})

async def account_summary(account_id: str) -> str:
    async with streamable_http_client(MCP_URL, http_client=mcp_http) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            result = await session.call_tool("account_summary", {"account_id": account_id})
            return result.content[0].text
```

Call this from inside the request handler, or from a tool the handler's model turn runs,
so the caller is still in context. `AsyncClient` takes every `httpx2.AsyncClient`
argument. Use `Client` for synchronous code. For a client the app cannot replace, add the
hook instead: `httpx2.AsyncClient(event_hooks={"request": [attach_identity_headers_async]})`,
imported from `diagrid.identity.http`.

## Traps

- On `mcp` 1.x a type checker rejects `http_client=`. Use `mcp` 2.x.
- Run on its own, or under plain `dapr run`, the app has no Diagrid connection. The first
  request with a token answers 503 `oauth.not_configured`. Start it with its Diagrid
  connection and tunnel, as `diagrid-app-tunnels` describes.

Reference: the `agents/langgraph/enterprise-identity` quickstart in
`diagridio/catalyst-quickstarts`, and the docs at
https://docs.diagrid.io/develop/agents/enterprise-identity.
