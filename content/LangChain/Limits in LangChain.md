---
tags: [note]
topic: langchain
sources:
  - https://docs.langchain.com/oss/python/langchain/middleware/built-in
  - https://docs.langchain.com/oss/python/langchain/agents
status: current
created: 2026-11-03
updated: 2026-11-03
---
# Limits in LangChain

> A limit stops an agent's loop after a maximum number of model calls, tool calls or graph steps, so a model that never decides it's done can't run forever.

## What it is

The agent loop ends only when the model answers without asking for a tool ([[What is a loop]]). Nothing guarantees it will: a model can keep asking for "one more" result, or keep failing to give a valid answer. Every extra turn resends the whole conversation, so a runaway loop gets slower and more expensive, and may never finish. A **limit** puts a ceiling on it.

## Its components

There are three limits, each counting a different unit ([[What is a turn]]):

| Limit | Counts | When it's reached |
|---|---|---|
| **model call limit** | model calls, i.e. turns | the agent stops, **without an answer** |
| **tool call limit** | tool calls, across all turns | further tool calls are refused, and the model must answer with what it has |
| **recursion limit** | graph steps: every model step, tools step and node-style hook | the run stops with an **error** |

They play different roles:
- the **tool call limit** is the gentle one: the agent can still answer;
- the **model call limit** is the firm one: the agent ends, possibly with no answer at all;
- the **recursion limit** is a safety net, for a loop nothing else has stopped.

## How LangChain implements it

- **Model call limit**: `ModelCallLimitMiddleware(run_limit=8)`, in `create_agent(middleware=[...])` ([[Middleware in LangChain]]).
  - `run_limit`: the maximum per run; `thread_limit`: across a whole conversation (needs a checkpointer).
  - `exit_behavior="end"` (the default) ends the agent quietly; `"error"` raises an exception.
  - When it ends the agent, the last message is *"Model call limits exceeded: run limit (8/8)"*, and `structured_response` is `None`.
- **Tool call limit**: `ToolCallLimitMiddleware(run_limit=6)`.
  - `tool_name` limits one tool only; without it, the limit covers all tools.
  - With `exit_behavior="continue"` (the default), each refused call gets the result *"Tool call limit exceeded. Do not make additional tool calls."*, and the loop goes on.
- **Recursion limit**: not middleware, but a setting of the run: `agent.invoke(..., config={"recursion_limit": 30})`.
  - `create_agent` sets it to **9,999** by default, so on its own it almost never fires: set your own if you want a backstop.
  - When it's reached, LangGraph raises `GraphRecursionError` (*"Recursion limit of 30 reached without hitting a stop condition"*).
  - Every node-style hook adds graph steps to each turn ([[Hooks in LangChain]]), so the recursion limit must be well above the number of turns you allow.

## Example

Take a researcher agent that measures the returns and risk of three funds (equity, bond and cash), and a strategist that turns its notes into weights in a fixed form ([[Schemas in LangChain]]). Both get limits:

```python
from langchain.agents.middleware import ModelCallLimitMiddleware, ToolCallLimitMiddleware

researcher = create_agent(model, tools=[get_returns, get_volatility], context_schema=Mandate,
    middleware=[ToolCallLimitMiddleware(run_limit=6), ModelCallLimitMiddleware(run_limit=8)])
strategist = create_agent(model, response_format=ToolStrategy(Allocation), context_schema=Mandate,
    middleware=[ModelCallLimitMiddleware(run_limit=8)])
```

In a normal run, the researcher uses 4 turns and 4 tool calls, and the strategist 2 turns: nothing is reached. Now imagine two runaway cases:
- **The researcher is never satisfied**: after the 90-day returns it asks for 120 days, then 180, then 200… After 6 tool calls, the tool call limit refuses every new one and tells the model to stop calling tools, so it should write its notes with the data it already has. If it keeps trying anyway, the model call limit ends it after 8 turns.
- **The strategist keeps forgetting the Cash Fund**: every answer fails its schema, and it tries again. After 8 model calls, the model call limit ends it with no weights at all, so the next step must handle a missing answer.

And the recursion limit? The researcher's normal run takes 4 turns, but 20 graph steps with these two limits (each turn passes through their hooks before and after the model call). So a recursion limit of 20 is the bare minimum for it; set it with room to spare, e.g. `config={"recursion_limit": 60}`.

## Watch out
- **Every agent needs a model call limit.** Without one, nothing stops a model that keeps calling tools or keeps failing its schema.
- **A limit can mean no answer.** After the model call limit, check `structured_response` before using it.
- **Don't count on the recursion limit.** With `create_agent`'s default of 9,999 it won't save you from a runaway loop; the model call limit will.

## Related
[[LangChain Overview]] · [[What is a turn]] · [[What is a loop]] · [[Middleware in LangChain]] · [[Hooks in LangChain]] · [[The agent loop in LangChain]]

## My notes & questions
