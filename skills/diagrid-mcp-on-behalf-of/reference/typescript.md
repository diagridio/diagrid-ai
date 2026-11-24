# TypeScript

Package: `@diagrid/agent-core` on npm, 0.2.0 or later, on Node 22.13 or later. Identity is
part of the core package, with subpaths for Express and Fastify. Add the latest, plus the
MCP TypeScript SDK:

```bash
npm install @diagrid/agent-core @modelcontextprotocol/sdk
```

If `npm` answers 403 or 404, the user's `.npmrc` may send the `@diagrid` scope to a
private registry. A scope setting wins over `--registry`, so add
`--@diagrid:registry=https://registry.npmjs.org`. Do not edit the user's `.npmrc`.

## Verify the caller

Express:

```ts
import express from 'express';
import { getVerifiedUser, oauthMiddleware } from '@diagrid/agent-core/express';

const app = express();
app.use(oauthMiddleware());   // everything is discovered

app.post('/agent/run', express.json(), async (req, res) => {
  const user = getVerifiedUser(req)!;   // never undefined while requireAuth is on
  ...
});
```

For Fastify, register `oauthPlugin` and read the user with `getVerifiedUser`, both from
`@diagrid/agent-core/fastify`. Read the user only with `getVerifiedUser`. Never read
`req.user`, which belongs to the app. The user has `subject`, `tenant`, `scopes`,
`claims` and `issuerId`. To let tokenless requests through, for health probes, pass
`{ requireAuth: false }` and check for a missing user.

## Call MCP tools as the caller

`createIdentityFetch()` returns a `fetch` that adds the caller's token. Hand it to the MCP
transport:

```ts
import { createIdentityFetch } from '@diagrid/agent-core';
import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js';

const MCP_URL = `${process.env['DAPR_HTTP_ENDPOINT']}/v1.0/diagrid/mcp/crm-mcp`;
const identityFetch = createIdentityFetch();   // safe to share: reads the caller per request

export async function accountSummary(accountId: string): Promise<string> {
  const transport = new StreamableHTTPClientTransport(new URL(MCP_URL), {
    fetch: identityFetch,
    requestInit: { headers: { 'dapr-api-token': process.env['DAPR_API_TOKEN'] ?? '' } },
  });
  const client = new Client({ name: 'my-agent', version: '1.0.0' });
  await client.connect(transport);
  try {
    const answer = await client.callTool({ name: 'account_summary', arguments: { account_id: accountId } });
    ...
  } finally {
    await client.close();
  }
}
```

The caller is kept in `AsyncLocalStorage` for the life of the request, so call this from
the handler or from a tool the handler's model turn runs.

## Traps

- `authenticate` is internal and not exported. The public names are `oauthMiddleware` and
  `oauthPlugin`.
- `attachIdentityHeaders` exists but does not drop the header on a redirect to another
  origin. Use `createIdentityFetch`.
- Without a Diagrid connection the first request with a token answers 503
  `oauth.not_configured`. Start the app with its connection and tunnel, as
  `diagrid-app-tunnels` describes.

Reference: the `agents/langgraphjs/enterprise-identity` quickstart in
`diagridio/catalyst-quickstarts`.
