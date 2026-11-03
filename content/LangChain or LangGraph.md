---
tags: [note]
topic: both
sources:
  - https://docs.langchain.com/oss/python/langchain/agents
  - https://docs.langchain.com/oss/python/langchain/multi-agent/custom-workflow
status: current
created: 2026-11-02
updated: 2026-11-02
---
# LangChain or LangGraph?

> LangChain gives you a ready-made agent: you describe it, it runs the loop. LangGraph gives you the pieces: you draw every step yourself. Pick by how much of the loop you want to control.

## Context

Take an agentic system that builds a portfolio from three funds: a researcher calls tools for 4 turns, a strategist is corrected once, and a risk check fixes the weights. Both libraries can build this. The question is who writes the loop *"call the model → run the tools → call the model again"*: the library, or you.

## The two in one table

| | LangChain (`create_agent`) | LangGraph (`StateGraph`) |
|---|---|---|
| **You write** | the agent's parts: model, prompt, tools, answer form, middleware | every step as a node, and the edges between them |
| **It hides** | the loop: the model and tools steps, when to stop, error handling | nothing: what you don't draw doesn't happen |
| **Good at** | getting an agent with tools running in a few lines | custom control flow: branches, loops with your own rules, parallel steps |
| **Your own logic goes** | in middleware, at fixed moments of the loop ([[Hooks in LangChain]]) | in any node, anywhere in the graph |
| **The risk check** | middleware after the strategist answers | its own node, with a branch after it |
| **When it's hard** | you need control *between* the loop's fixed moments | you rewrite things LangChain gives for free (retries on a bad answer, limits) |

They aren't rivals: `create_agent` **builds a LangGraph graph**, and a common setup uses both. The workflow (researcher → strategist → writer) is a LangGraph graph, and each step runs a LangChain agent ([[Workflow in LangChain]]).

## How to choose

- **Start with LangChain** if each step is "an agent that uses some tools and returns an answer". You'll have it running quickly, and middleware covers most custom needs.
- **Use LangGraph** if the order of steps depends on results (send the strategist back with feedback, run two researchers in parallel), or you want every step visible in the graph.
- **Mix them** when the overall flow is custom but each step is a normal agent: a LangGraph workflow whose nodes call `create_agent` agents.

## What to read

| LangChain | LangGraph |
|---|---|
| **[[LangChain Overview]]** (start here) | [[Workflow in LangGraph]] |
| [[Agents in LangChain]] | [[State in LangGraph]] |
| [[The agent loop in LangChain]] | [[Agents in LangGraph]] |
| [[Tools in LangChain]] | [[The agent loop in LangGraph]] |
| [[Runtime in LangChain]] | [[Tools in LangGraph]] |
| [[Schemas in LangChain]] | [[Schemas in LangGraph]] |
| [[Skills in LangChain]] | [[Branches in LangGraph]] |
| [[Hooks in LangChain]] | [[Errors and retries in LangGraph]] |
| [[Middleware in LangChain]] | |
| [[Limits in LangChain]] | |
| [[State in LangChain]] | |
| [[Workflow in LangChain]] | |

Both: [[What is a loop]], [[What is a tool]], [[What is a turn]].

## Related
[[Agents in LangChain]] · [[Workflow in LangChain]] · [[Workflow in LangGraph]]

## My notes & questions
