---
tags: [note]
topic: langgraph
template_files: [src/systems/langgraph/workflow.py, src/systems/langgraph/agents.py, src/systems/langgraph/tools.py, src/config.py]
status: current
created: 2026-10-30
updated: 2026-10-30
---
# Errors and retries in LangGraph

> Four different things can go wrong in a run, and the template has a different guard for each. Knowing which is which tells you where to look when a date fails.

## Context

On 31 January 2025, imagine four bad moments in the LangGraph system's run:
1. The model asks for statistics with `lookback_days=1`, below the minimum of 5.
2. `analyst_signals` answers with a missing asset.
3. The model server times out while `portfolio_manager` is waiting.
4. The analyst keeps asking for new windows and never stops.

None of them should crash the date immediately. Each has its own fix:

| Problem | Guard | Where | What happens | When it gives up |
|---|---|---|---|---|
| 1. a **tool call fails** (bad arguments, or the tool raises `ValueError`) | `tool_error` | `ToolNode(handle_tool_errors=tool_error)` | the error becomes the tool's result, and the model calls again ([[Tools in LangGraph]]) | the analyst's round limit (row 4) |
| 2. an **answer is invalid** | corrections in `ask_for_answer` | `AGENT_MAX_CORRECTIONS` = 4 | the error goes back, and the model answers again ([[Schemas in LangGraph]]) | after 4 corrections: the date fails |
| 3. a **node raises** (timeout, server error) | `RetryPolicy` | `retry_policy=LLM_RETRY` on each LLM node | LangGraph waits and runs **the whole node** again | after 3 attempts: the date fails |
| 4. the model **keeps calling tools** | round limit | `MAX_TOOL_ROUNDS` = 4 | the tool is no longer offered, so the model must answer ([[The agent loop in LangGraph]]) | — |
| any loop with no other limit | `recursion_limit` | `RECURSION_LIMIT` = 40 node runs | `GraphRecursionError` | immediately: the date fails |

## `RetryPolicy`: running a node again

```python
# workflow.py
LLM_RETRY = RetryPolicy(max_attempts=3)
graph.add_node("portfolio_manager", partial(portfolio_manager, llm=llm), retry_policy=LLM_RETRY)
```

If the node raises, LangGraph waits a little (longer each time) and calls the node function again from the start, up to 3 attempts. It's for problems that a second try can fix: the network, an overloaded server.

By default it does **not** retry a `ValueError`. That's deliberate. When `ask_for_answer` runs out of corrections it raises `ValueError`, and repeating the whole node would just repeat the same failure. Pass `retry_on=` to choose which exceptions are retried.

## Two kinds of error, two reactions

- **The model can fix it** (bad arguments, invalid answer): send the error *to the model* as a message. That's rows 1 and 2.
- **The model can't fix it** (network, server): *run the step again*. That's row 3.

That's also the rule for your own code. In a tool or a validator, raise `ValueError` with a clear message for anything the model can correct. Any other exception stops the date.

## Watch out
- **Retries multiply.** One node with 3 attempts × 5 answers each can mean 15 model calls. Keep the limits small.
- **A failed date shows in Langfuse** as an error on that date's trace: open it to see which guard gave up.

## Related
[[Tools in LangGraph]] · [[Schemas in LangGraph]] · [[The agent loop in LangGraph]] · [[Workflow in LangGraph]] · [[Middleware in LangChain]]

## My notes & questions
