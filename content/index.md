---
tags: [index]
created: 2026-10-30
updated: 2026-10-30
---
# Index

Every note in the vault, by topic. Decks are in `Slides/`.

## Start here
- [[The hackathon task]] — the running example: analyst → portfolio manager → risk, one reporting date step by step

## Working with Hermes
- [[Hermes coding agent]] — the files it reads (`SOUL.md`, `.hermes.md`), its skills, the commands, and how to steer it

## Both libraries
- [[What is a turn]] — one model call + the tools it asked for; what each limit counts
- [[What is a tool]] — describe → model asks → your code runs → result back; regardless of framework
- [[Tracing and Langfuse]] — what a trace is, the terminology, how the template sets it up, what Langfuse shows, how to debug with it

## LangGraph
Folder `LangGraph/`. Every step is a node you draw; suggested reading order:
- [[Workflow in LangGraph]] — the whole run as a graph: nodes, edges, `compile`
- [[State in LangGraph]] — one shared dictionary; replace vs reducers (`add_messages`)
- [[Agents in LangGraph]] — hand-written LLM nodes: `ask_for_text`, `ask_for_answer`
- [[The agent loop in LangGraph]] — `analyst ⇄ tools`, drawn in the graph; how it stops
- [[Tools in LangGraph]] — `bind_tools` + `ToolNode`; hidden date; `tool_error`; adding your own
- [[Schemas in LangGraph]] — forms the model fills in; the correction loop
- [[Branches in LangGraph]] — routers and conditional edges: `risk_review` → `rebalance`
- [[Errors and retries in LangGraph]] — four things that go wrong and the guard for each

## LangChain
Folder `LangChain/`. `create_agent` runs the loop for you.
- [[Agents in LangChain]] — what an agent is; the parts of `create_agent`; running it from a node
- [[Tools in LangChain]] — `@tool` + `tools=[...]`: what the model sees, who runs it, adding your own
- [[The agent loop in LangChain]] — the `model` ⇄ `tools` graph inside `create_agent`; when it ends
- [[Skills in LangChain]] — methods in `SKILL.md`, added to the system prompt; how to add one
- [[Hooks in LangChain]] — the moments in the loop where your code runs; step hooks vs wrap hooks
- [[Middleware in LangChain]] — hooks into the agent's loop; prebuilt, decorator or class
- [[Schemas in LangChain]] — instruction + check: tool inputs, agent outputs, runtime context
- [[State in LangChain]] — the graph's state vs each agent's state; return only what changes
- [[Workflow in LangChain]] — three agents in a line; `StateGraph` → nodes → edges → compile
