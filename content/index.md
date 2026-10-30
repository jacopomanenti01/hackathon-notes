---
tags: [index]
created: 2026-10-30
updated: 2026-10-30
---
# Index

Every note in the vault, by topic. Decks are in `Slides/`.

## Start here
- [[The hackathon task]] — the running example: analyst → portfolio manager → risk, one reporting date step by step

## Both libraries
- [[What is a turn]] — one model call + the tools it asked for; what each limit counts
- [[What is a tool]] — describe → model asks → your code runs → result back; regardless of framework

## LangGraph
- [[Tools in LangGraph]] — the model asks, a `ToolNode` runs; schema-as-tool for structured answers

## LangChain
- [[Agents in LangChain]] — what an agent is; the parts of `create_agent`; running it from a node
- [[Tools in LangChain]] — `@tool` + `tools=[...]`: what the model sees, who runs it, adding your own
- [[The agent loop in LangChain]] — the `model` ⇄ `tools` graph inside `create_agent`; when it ends
- [[Skills in LangChain]] — methods in `SKILL.md`, added to the system prompt; how to add one
- [[Hooks in LangChain]] — the moments in the loop where your code runs; step hooks vs wrap hooks
- [[Middleware in LangChain]] — hooks into the agent's loop; prebuilt, decorator or class
- [[Schemas in LangChain]] — instruction + check: tool inputs, agent outputs, runtime context
- [[State in LangChain]] — the graph's state vs each agent's state; return only what changes
- [[Workflow in LangChain]] — three agents in a line; `StateGraph` → nodes → edges → compile

## General
