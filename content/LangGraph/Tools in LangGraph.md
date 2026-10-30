---
tags: [note]
topic: langgraph
template_files: [src/systems/langgraph/tools.py, src/systems/langgraph/agents.py, src/systems/langgraph/workflow.py]
status: current
created: 2026-10-30
updated: 2026-10-30
---
# Tools in LangGraph

> In LangGraph, a tool needs two things: `bind_tools` tells the model it exists, and the `ToolNode` in the graph runs it.

## Context

At the first step of the analyst's run ([[The agent loop in LangGraph]]), the model replies "call `get_asset_statistics` with 90 days". For that to work, three things must be true, as in any framework ([[What is a tool]]):
1. **the model knows the tool exists**, what it returns and what `lookback_days` means;
2. **something runs it** and gives the result back;
3. **it only reads prices up to 31 January 2025** ([[The hackathon task#The rule that's easy to break: no look-ahead|no look-ahead]]).

In LangGraph, each of the three has its own line of code you can point to.

## 1. The model knows: `bind_tools`

```python
# agents.py, analyst
model = llm.bind_tools([get_asset_statistics])
reply = model.invoke(messages)
# reply.tool_calls == [{"name": "get_asset_statistics", "args": {"lookback_days": 90}, "id": "c1"}]
```

`bind_tools` attaches the tool's description to every call of this model. The description is built by `@tool` from the function's name, docstring and input schema (`StatisticsInput`: an integer from 5 to 3650). The reply is a **request**, and nothing has run yet.

## 2. Something runs it: `ToolNode`

```python
# workflow.py
graph.add_node("tools", ToolNode([get_asset_statistics], handle_tool_errors=tool_error))
```

`ToolNode` is a prebuilt node. It takes the tool calls in the last message, checks the arguments against the schema, runs the function, and adds each result as a `ToolMessage` with the call's `id`, so the model knows which request it answers. The graph then goes back to `analyst`.

So the tool is listed **twice**: in `bind_tools`, so the model can ask for it, and in `ToolNode`, so the graph can run it. Forget the second, and the model asks for a tool nobody runs.

## 3. Only data up to the date: `runtime.state`

```python
# tools.py (shortened)
@tool(args_schema=StatisticsInput)
def get_asset_statistics(lookback_days: int, runtime: ToolRuntime) -> dict:
    """Mean daily return, volatility and a Sharpe-like score for every asset, ..."""
    end = runtime.state["reporting_date"]          # "2025-01-31", from the graph's state
    prices = read_assets(end_date=end)
```

The `runtime` parameter is hidden from the model. `ToolNode` fills it in, and `runtime.state` is the graph's state ([[State in LangGraph]]). So the model picks the window, but never the date.

## When a call fails: `tool_error`

If the model sends `lookback_days=1`, the schema rejects it. Instead of crashing the run, `ToolNode` calls `tool_error`, and its text becomes the tool's result:

```python
def tool_error(error: ValueError | ToolInvocationError) -> str:
    return f"Error: {error}\nFix the arguments and call the tool again."
```

The model reads the error and calls again with a valid value. Only the error types in the annotation are caught: bad arguments (`ToolInvocationError`), and a `ValueError` the tool raises itself. Anything else stops the date.

## A tool that runs nothing: a form

You can also bind a **Pydantic schema** instead of a function, and force the model to call it (`tool_choice="required"`). Then the arguments the model writes *are* its answer, and there's nothing to run, so no `ToolNode` is needed. That's how `ask_for_answer` gets answers in a fixed shape ([[Schemas in LangGraph]]).

| Use | What you bind | Needs a `ToolNode`? |
|---|---|---|
| fetch data or act | a function (`get_asset_statistics`) | yes |
| an answer in a fixed shape | a schema (`Analysis`) | no: read `reply.tool_calls[0]["args"]` |

## Adding your own tool
1. Its input schema in `schema.py`, and the function with `@tool(args_schema=...)` in `tools.py`. Read the date from `runtime.state["reporting_date"]`, and pass it as `end_date=` to the data readers.
2. Add it to `bind_tools([...])` in `analyst` **and** to `ToolNode([...])` in `workflow.py`.
3. Inside the tool, `raise ValueError("...")` for input the model can fix: `tool_error` sends it back.

## Related
[[What is a tool]] · [[The agent loop in LangGraph]] · [[Schemas in LangGraph]] · [[Tools in LangChain]]

## My notes & questions
