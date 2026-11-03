---
tags: [note]
topic: langgraph
sources:
  - https://docs.langchain.com/oss/python/langchain/tools
  - https://docs.langchain.com/oss/python/langgraph/workflows-agents
status: current
created: 2026-10-30
updated: 2026-11-03
---
# Tools in LangGraph

> In LangGraph, a tool needs two things: `bind_tools` tells the model it exists, and a `ToolNode` in the graph runs it.

## What it is

A model can't run code: it can only *ask* for a tool, and your program runs it and sends the result back ([[Introduction#What is a tool|What is a tool]]). LangChain's `create_agent` does both halves for you. In LangGraph you do each one yourself, so each has its own line of code: one so the model knows the tool exists, and one so the graph runs it.

## Its components

- **The tool**: a function with a name, a description and typed arguments.
- **Binding**: giving the model the tools' descriptions, so its reply can ask for them.
- **The tools node**: a node that runs the calls in the model's last reply and adds the results to the conversation ([[The agent loop in LangGraph]]).
- **Hidden inputs**: values the tool needs but the model must not choose, filled in by your code.
- **Error handling**: what happens when a call fails.

## How LangGraph implements it

- **The tool**: `@tool`, from `langchain.tools`. The function's name becomes the tool's name, its docstring the description, and its arguments the input schema. `@tool(args_schema=...)` gives a Pydantic class instead, with limits on the values ([[Schemas in LangGraph]]).
- **Binding**: `model.bind_tools([...])`, inside the model's node ([[Agents in LangGraph]]). The reply's `tool_calls` lists what it asks for, each with an `id`. Nothing has run yet.
- **The tools node**: `ToolNode([...])`, from `langgraph.prebuilt`. It checks each call's arguments, runs the function, and adds each result as a `ToolMessage` with the call's `id`.
- **Hidden inputs**: an argument `runtime: ToolRuntime` is left out of the description the model sees, and `ToolNode` fills it in:
  - `runtime.context`: the run's fixed values, passed with `workflow.invoke(..., context=...)` ([[Runtime in LangGraph]]);
  - `runtime.state`: the graph's state.
- **Error handling**: `ToolNode(..., handle_tool_errors=...)`:
  - by default, only **bad arguments** are caught: the error goes back to the model as the tool's result, and the model can call again;
  - any other error raised inside the tool stops the run, unless you pass `handle_tool_errors=True` (every error goes back), or your own function that writes the message.

## Example

Take a researcher agent that measures the returns and risk of three funds (equity, bond and cash). The funds come from a **mandate** the model must not change, so they aren't an argument: the tool reads them from the context.

```python
from langchain.tools import ToolRuntime, tool
from pydantic import BaseModel, Field

class ReturnsInput(BaseModel):
    days: int = Field(ge=5, le=365, description="How many past days to measure, from 5 to 365")

@tool(args_schema=ReturnsInput)
def get_returns(days: int, runtime: ToolRuntime[Mandate]) -> dict:
    """Total return of every fund in the mandate over the last `days` days, in %."""
    return {f: RETURNS[f] for f in runtime.context.funds}       # the funds: hidden from the model
```

The tool is listed **twice**: once where the model is called, and once in the graph.

```python
reply = model.bind_tools([get_returns, get_volatility]).invoke(messages)   # in the researcher node
graph.add_node("tools", ToolNode([get_returns, get_volatility]))           # in the graph
```

The model's first call asks for `days=500`. The schema rejects it, and the `ToolNode` sends the error back as the result: *"days: Input should be less than or equal to 365 … Please fix the error and try again."*. In the next [[Introduction#What is a turn|turn]] the model asks for `days=90`, and gets the returns of all three funds without ever naming them: equity +4%, bond +1%, cash +0.5%.

## Watch out
- **Bound but not in the `ToolNode`** (or the other way round): the model asks for a tool nobody runs. List each tool in both places.
- **Errors inside the tool stop the run by default.** If the model could fix it (e.g. "use a longer window"), raise a clear error and set `handle_tool_errors=True` ([[Errors and retries in LangGraph]]).
- **Calling a tool by hand** fails if it needs a runtime: `ToolNode` fills it in only inside the graph.

## References
- [LangChain docs: Tools](https://docs.langchain.com/oss/python/langchain/tools)
- [LangGraph docs: Workflows and agents](https://docs.langchain.com/oss/python/langgraph/workflows-agents)

## Related
[[A Brief Overview on LangGraph]] · [[Introduction]] · [[Runtime in LangGraph]] · [[The agent loop in LangGraph]] · [[Agents in LangGraph]] · [[Schemas in LangGraph]] · [[Tools in LangChain]]
