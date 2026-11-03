---
tags: [note]
topic: langgraph
sources:
  - https://docs.langchain.com/oss/python/langgraph/fault-tolerance
  - https://docs.langchain.com/oss/python/langgraph/use-graph-api
  - https://docs.langchain.com/oss/python/langgraph/graph-api
  - https://docs.langchain.com/oss/python/langgraph/errors/GRAPH_RECURSION_LIMIT
  - https://docs.langchain.com/oss/python/langchain/tools
status: current
created: 2026-10-30
updated: 2026-11-03
---
# Errors and retries in LangGraph

> Four different things can go wrong in an agent's run, and each needs its own guard. In LangGraph, you add every guard yourself: what you don't write doesn't happen.

## What it is

A model makes mistakes, a server fails, and a loop may never end. Each problem needs a different reaction:
- if **the model can fix it** (a bad argument, an invalid answer), send the error *to the model*, as a message, and let it try again;
- if **the model can't fix it** (a timeout, a server error), *run the step again*;
- if **nothing stops it** (a model that keeps asking for more), *cap it* with a limit.

## Its components

- **Tool errors**: a tool call with bad arguments, or a tool that raises an error.
- **Invalid answers**: an answer that fails its schema ([[Schemas in LangGraph]]).
- **Node failures**: a node raises, e.g. the model's server times out.
- **Endless loops**: a model that never stops calling tools.
- **The recursion limit**: the backstop for any loop nothing else has stopped.

## How LangGraph implements it

- **Tool errors**: the `ToolNode` turns the error into the tool's result, and the model calls again ([[Tools in LangGraph]]). By default it does this only for bad arguments; `handle_tool_errors=True` does it for every error the tool raises.
- **Invalid answers**: the correction loop inside the node. On a failed check, the error goes back to the model, up to a maximum number of attempts; then the node raises.
- **Node failures**: `graph.add_node(..., retry_policy=RetryPolicy(max_attempts=3))`, from `langgraph.types`. If the node raises, LangGraph waits (a little longer each time) and runs **the whole node** again. By default it retries network and server errors, but not errors that a second try won't fix, like a `ValueError`.
- **Endless loops**: your own count in the model's node: after a maximum number of tool calls, stop binding the tools ([[The agent loop in LangGraph]]).
- **The recursion limit**: `workflow.invoke(..., config={"recursion_limit": 30})` caps the number of graph steps in a run. When it's reached, LangGraph raises `GraphRecursionError` (*"Recursion limit of 30 reached without hitting a stop condition"*). Its default is 10,007, so on its own it won't stop a runaway loop for a long time: set your own.

## Example

Take a system that builds a portfolio from three funds (equity, bond and cash): a researcher with two tools, `get_returns(days)` (at most 365 days) and `get_volatility(fund)`, then a strategist that must give a weight to every fund. Imagine four bad moments, and the guard for each:

1. **The researcher asks for `days=500`.** The `ToolNode` sends back *"days: Input should be less than or equal to 365 … Please fix the error and try again."*, and the next call asks for 90 days.
2. **The strategist forgets the Cash Fund.** Its node sends back *"Every fund needs a weight; missing: ['cash']"*, and the second answer passes. If it kept forgetting, the node would give up after 3 attempts with an error.
3. **The model's server times out** during the strategist's call. Its retry policy runs the node again, from the start:

   ```python
   from langgraph.types import RetryPolicy

   retry = RetryPolicy(max_attempts=3)
   graph.add_node("researcher", researcher, retry_policy=retry)
   graph.add_node("strategist", strategist, retry_policy=retry)
   graph.add_node("writer", writer, retry_policy=retry)
   ```
4. **The researcher is never satisfied**: after 90 days it asks for 120, then 180, then 200… After 6 tool calls, its node stops binding the tools, so the model must write its notes with what it has.

And the recursion limit? The whole run normally takes 11 graph steps, and a limit of 11 already fails it. So set it with room to spare, e.g. `config={"recursion_limit": 30}`, and it will only fire for a loop nothing else stopped.

## Watch out
- **Retries multiply.** A node with 3 attempts, each with 3 answers, can mean 9 model calls. Keep each limit small.
- **A retry reruns the whole node**, including what it did before it failed. Keep a node that calls a model free of other side effects.
- **A failed run shows in your traces** as an error on that run: open it to see which guard gave up.

## Related
[[LangGraph Overview]] · [[Tools in LangGraph]] · [[Schemas in LangGraph]] · [[The agent loop in LangGraph]] · [[Workflow in LangGraph]] · [[Limits in LangChain]]

## My notes & questions
