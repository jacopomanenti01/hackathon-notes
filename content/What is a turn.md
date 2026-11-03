---
tags: [note]
topic: general
sources:
  - https://docs.langchain.com/oss/python/langchain/agents
  - https://docs.langchain.com/oss/python/langchain/middleware/built-in
  - https://docs.langchain.com/oss/python/langgraph/graph-api
status: current
created: 2026-10-30
updated: 2026-11-03
---
# What is a turn

> A turn is one round of the agent loop ([[What is a loop]]): **one model call**, plus running the tools that call asked for. A **graph step** is smaller: one of the internal steps a turn is made of.

## What it is

An agent doesn't answer in one go. It works in rounds, like a conversation with an assistant who can look things up: you ask, it decides it needs some data, it asks for it, reads it, and decides again. Each of those rounds is a **turn**:
1. the model is called once, with the whole conversation so far, and replies;
2. if the reply asks for tools, they run, and their results are added to the conversation ([[What is a tool]]).

Then the next turn starts with the next model call. A reply that asks for no tools is the answer, and that's the **last turn**.

The simplest way to count turns is to count **model calls**: one turn, one model call. That's also why turns matter: every turn is a model call that resends the whole conversation, so turns are what an agent is limited by, and paid for.

Inside the agent, each turn is made of smaller pieces: the **graph steps**. Both libraries run an agent as a graph, a set of steps connected by arrows: LangChain builds it for you, and in LangGraph you draw it. A **graph step** is one run of one of those steps: the model step, a tools step, or (in LangChain) a middleware hook. So a turn is usually several graph steps.

## Its components

| Unit | What it is | Size |
|---|---|---|
| **turn** | one model call + the tools it asked for | the agent's round |
| **tool call** | one request to run one tool | zero, one or several per turn |
| **graph step** | one run of one step of the agent's graph | several per turn |

A turn can hold **several tool calls**: if the model asks for two tools in one reply, both run before the next model call, still in the same turn.

## How it's implemented

### In LangChain

`create_agent` builds the agent as a graph ([[The agent loop in LangChain]]). What counts as a graph step:
- **`model`**: one step per turn, the model call;
- **`tools`**: one step **per tool call**, so a turn with two tool calls has two;
- **each node-style hook** (`before_model`, `after_model`, …): one step each, every turn ([[Hooks in LangChain]]);
- **wrap-style hooks** (`wrap_model_call`, `@dynamic_prompt`, …): no step of their own, they run inside the `model` or `tools` step.

Each unit has its own limit:
- **turns** → the model call limit (`ModelCallLimitMiddleware`);
- **tool calls** → the tool call limit (`ToolCallLimitMiddleware`);
- **graph steps** → the recursion limit.

What happens when each one is reached, and how to set them: [[Limits in LangChain]].

### In LangGraph

You draw the graph yourself, so the steps are your nodes ([[The agent loop in LangGraph]]):
- **the model node**: one step per turn;
- **the `ToolNode`**: one step per turn that asked for tools, however many calls it holds: it runs them all together;
- **any other node** you add: one step each time it runs.

There are no ready-made limits on turns or tool calls: you count them yourself in the model node. Only the graph steps have a built-in limit, the recursion limit ([[Errors and retries in LangGraph]]).

## Example

Take a researcher agent that must measure the returns and risk of three funds (equity, bond and cash), with two tools: `get_returns(days)`, for at most 365 days, and `get_volatility(fund)`. Its run takes **4 turns** and **4 tool calls**:

| Turn | The model call: it reads all messages and replies… | …then the tools run |
|---|---|---|
| 1 | "call `get_returns(days=500)`" | rejected: days must be ≤ 365; the error is added to the messages |
| 2 | "call `get_returns(days=90)`" | runs; the returns are added |
| 3 | "call `get_volatility("equity")` and `get_volatility("bond")`" | both run; both results are added |
| 4 | the notes, with no tool call | nothing to run, so the loop ends |

**Counted in graph steps**, the same run takes:
- **8 steps without middleware**: `model`, `tools` for turns 1 and 2; `model`, `tools`, `tools` for turn 3 (one `tools` step per tool call); `model` for turn 4;
- **20 steps with the researcher's middleware** (a model call limit and a tool call limit, which run as hooks before and after each model call): turn 1 alone is 5 steps, `before_model` → `model` → `after_model` → `after_model` → `tools`. Its prompt and retry middleware are wrap-style, so they add no steps.
- **7 steps in a hand-written LangGraph loop**: `researcher`, `tools` for each of turns 1–3 (turn 3's two calls run in one `tools` step), then `researcher` for turn 4.

So 4 turns can be 7 or 20 graph steps: a model call limit of 8 is a limit on turns, while the recursion limit has to leave room for all the steps those turns take ([[Limits in LangChain]]).

## Watch out
- **Each turn resends everything.** The model has no memory between calls: more turns cost more time and tokens.
- **Every hook you add makes each turn longer in graph steps.** If you add middleware, check that the recursion limit is still well above what your turns need.
- **"Turn" in chat apps** sometimes means one message from one side. In these notes, a turn is always one model call plus its tools.

## Related
[[LangChain Overview]] · [[LangGraph Overview]] · [[What is a loop]] · [[What is a tool]] · [[The agent loop in LangChain]] · [[The agent loop in LangGraph]] · [[Hooks in LangChain]] · [[Limits in LangChain]] · [[Errors and retries in LangGraph]]

## My notes & questions
