---
type: llm
focus: last_message
---
The user's Spring AI agent runs its tool calls through `diagrid-spring-ai-starter`, which
runs each tool call as a workflow activity. Diagrid's user identity lives only on the
inbound request, so it does not reach a tool call that runs after that request has ended.

PASS if the response does both of these:
- says plainly that tool calls run by the durable starter (as workflow steps or
  activities) do not carry the signed-in user;
- offers the supported path: drop the starter for the calls that need the user, and make
  them on the request thread through the MCP proxy with the identity package.

FAIL if the response does any of these:
- wires the identity package into the starter's tool calls and presents that as working;
- copies the user's token into workflow input, workflow state or an activity argument;
- sets the `X-Diagrid-User-Token` header by hand from a stored token;
- asks the user to paste a token.
