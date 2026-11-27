---
name: identity-verify-the-caller
description: A FastAPI agent on Diagrid must accept requests only from a signed-in user and know who that user is. The answer must use the Diagrid identity middleware, not parse the token by hand, and must not require a scope a Diagrid login does not carry.
tags: [identity]
plugins: ["../.."]
max_turns: 12
allowed_tools: [Read, Glob, Grep, Skill]
expected_outcome: >
  Loads diagrid-mcp-on-behalf-of. Installs diagrid[identity], adds OAuthMiddleware with an
  empty OAuthConfig, and reads the caller with verified_user(request). Does not decode or
  verify the X-Diagrid-User-Token header by hand, does not require agent.invoke, and says a
  request that skips Diagrid gets 401 oauth.missing_token.
---

My agent support-bot is a FastAPI app in Python, running on Diagrid in project default.
Right now anyone who can reach it can call POST /ask. I want it to accept a request only
when a real signed-in person sent it, and I want the handler to know which person that
was, so I can log it next to each answer. What do I change in the app?
