---
tags: [index]
created: 2026-10-30
updated: 2026-11-02
---
# Index

Notes on the ideas behind LangChain and LangGraph, explained on one small example. Decks are in `Slides/`.

## Start here
- [[LangChain Overview]] — a guided tour of LangChain, told as the story of one agentic portfolio system, with a link to each detail note
- [[LangChain or LangGraph]] — what each is good at, what each hides from you, and which pages to read for each

## Both libraries
- [[What is a loop]] — the agent loop: model decides → tools run → model reads, until it answers; what every loop needs
- [[What is a tool]] — describe → model asks → your code runs → result back; regardless of framework
- [[What is a turn]] — one model call + the tools it asked for; what each limit counts
- [[Tracing and Langfuse]] — what a trace is, the terminology, what Langfuse shows, how to debug with it

## LangChain
Folder `LangChain/`. `create_agent` runs the loop for you.

**Start with [[LangChain Overview]]**: a guided tour of how the ideas fit together, told as the story of the running example, with a link to the detail note for each idea. Detail notes, in reading order:
- [[Agents in LangChain]] — what an agent is; the parts of `create_agent`; running it
- [[The agent loop in LangChain]] — the model ⇄ tools loop inside `create_agent`; when it ends
- [[Tools in LangChain]] — `@tool`: what the model sees, what happens on a bad call
- [[Runtime in LangChain]] — context, state and store: what tools and middleware see, and the model doesn't
- [[Schemas in LangChain]] — instruction + check: tool inputs, the answer's form, the context
- [[Skills in LangChain]] — methods in their own files: in the system prompt, or loaded on demand
- [[Hooks in LangChain]] — the moments in the loop where your code runs; node-style vs wrap-style
- [[Limits in LangChain]] — model calls, tool calls, recursion: what each limit counts, and what happens when it's reached
- [[Middleware in LangChain]] — your code in the loop: built-in, decorator or class; guardrails and limits
- [[State in LangChain]] — the workflow's state vs each agent's state; adding keys
- [[Workflow in LangChain]] — agents in a line with `StateGraph`; other ways to combine agents

## LangGraph
Folder `LangGraph/`. Every step is a node you draw; suggested reading order:
- [[Workflow in LangGraph]] — the whole run as a graph: nodes, edges, `compile`
- [[State in LangGraph]] — one shared dictionary; replace vs reducers (`add_messages`)
- [[Agents in LangGraph]] — hand-written LLM nodes
- [[The agent loop in LangGraph]] — the model ⇄ tools loop, drawn in the graph; how it stops
- [[Tools in LangGraph]] — `bind_tools` + `ToolNode`; hidden values; tool errors
- [[Schemas in LangGraph]] — forms the model fills in; the correction loop
- [[Branches in LangGraph]] — routers and conditional edges
- [[Errors and retries in LangGraph]] — four things that go wrong and the guard for each

## Working with Hermes
- [[Hermes coding agent]] — how to start it, the files it reads, its skills, the commands, and how to steer it
