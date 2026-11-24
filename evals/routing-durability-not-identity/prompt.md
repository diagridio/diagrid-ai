---
name: routing-durability-not-identity
description: A request for crash recovery alone, with an MCP tool call in the agent, must not load the identity skill. Durability and identity are separate skills, and a durability-only run should not carry identity guidance.
tags: [routing]
plugins: ["../.."]
max_turns: 8
allowed_tools: [Read, Glob, Grep, Skill]
expected_outcome: >
  diagrid-mcp-on-behalf-of does not fire. An agent or harness skill handles the request.
---

My LangGraph agent on Diagrid calls a weather MCP server through Diagrid's MCP proxy. When
the process crashes half way through an answer, the whole conversation starts again and
the tools run twice. I want it to pick up from where it stopped after a restart. Nothing
else about it needs to change.
