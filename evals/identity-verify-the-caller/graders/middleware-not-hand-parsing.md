---
type: llm
focus: last_message
---
A Python FastAPI app on Diagrid must accept only requests from a signed-in user and read
who that user is. Diagrid puts a verified user token in the `X-Diagrid-User-Token` header,
and the `diagrid[identity]` package verifies it.

PASS if the response does all of these:
- installs `diagrid[identity]` (alone or next to another extra);
- adds `OAuthMiddleware` from `diagrid.identity.asgi` with an `OAuthConfig`;
- reads the caller in the handler with `verified_user(request)`.

FAIL if the response does any of these:
- decodes, splits or verifies the token itself (base64, `jwt.decode`, a hand-written
  JWKS fetch) instead of using the middleware;
- requires the `agent.invoke` scope, or any scope the user did not name;
- sets the issuer, audience or JWKS URI by hand without a reason the user gave;
- logs, prints or returns the raw token;
- asks the user to paste a token.
