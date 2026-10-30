---
tags: [note]
topic: langchain
template_files: [src/systems/langchain/agents.py, src/systems/langchain/tools.py, src/systems/langchain/middleware.py]
status: current
created: 2026-10-30
updated: 2026-10-30
---
# Tools in LangChain

> You write the tool as a normal Python function with `@tool`, hand it to `create_agent`, and the agent runs it whenever the model asks.

## Context

In the [[Agents in LangChain#Context|running example]], the model asks for `get_asset_statistics` with 90 days, and "the program runs it". Two questions come out of that:
- how does the model know the tool exists, and what `lookback_days` means?
- who is "the program" that runs it, and how does it know the date is 31 January 2025?

Both answers are in two places: the tool's definition (`tools.py`) and the agent's construction (`agents.py`).

## Defining the tool

```python
# tools.py (shortened)
@tool(args_schema=StatisticsInput)
def get_asset_statistics(lookback_days: int, runtime: ToolRuntime[AgentContext]) -> dict:
    """Mean daily return, volatility (std_dev) and a Sharpe-like score for every asset,
    over the last `lookback_days` calendar days up to the reporting date (e.g. 90 or 365)."""
    end = runtime.context.reporting_date          # 2025-01-31, set by our code
    prices = read_assets(end_date=end)
    ...
```

`@tool` turns the function into the description the model reads ([[What is a tool]]):
- the **name** is the function's name;
- the **description** is the docstring, so write it for the model: what it returns and when to use it;
- the **arguments** come from `StatisticsInput`: `lookback_days`, an integer between 5 and 3650 ([[Schemas in LangChain]]).

The `runtime` parameter is **not** shown to the model. `create_agent` fills it in when it runs the tool, and it carries the context passed to `agent.invoke(..., context=AgentContext("2025-01-31"))`. So the model chooses *how far back* to look, but never *up to which date*.

## Giving it to the agent

```python
# agents.py (analyst)
agent = create_agent(llm, tools=[get_asset_statistics], ...)
```

That one argument does all the work. `create_agent` sends the tool descriptions with every model call. When the reply contains a call, it runs the tool and sends the result back ([[The agent loop in LangChain]]). You never write `bind_tools`, a tools node or an edge, unlike LangGraph ([[Tools in LangGraph]]).

The final answer is a tool call too: with `response_format=ToolStrategy(Analysis)`, the model "calls" `Analysis` with its signals as arguments. That's how step 6 of the example becomes a checked object.

## Watch out
- **Limits are middleware.** `tool_call_limit()` refuses tool calls after `MAX_TOOL_CALLS`. `call_limit()` stops the agent after `MAX_MODEL_CALLS` ([[Middleware in LangChain]]).
- **Malformed calls.** If the model's call is broken JSON, the agent would end without an answer. `retry_invalid_tool_calls` sends the error back and asks again.
- **The docstring is a prompt.** A vague docstring gives vague calls.

## Related
[[What is a tool]] · [[Agents in LangChain]] · [[The agent loop in LangChain]] · [[Schemas in LangChain]] · [[Skills in LangChain]] · [[Tools in LangGraph]]


