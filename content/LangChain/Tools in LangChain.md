---
tags: [note]
topic: langchain
sources:
  - https://docs.langchain.com/oss/python/langchain/tools
  - https://docs.langchain.com/oss/python/langchain/runtime
  - https://huggingface.co/learn/agents-course/unit1/tools
status: current
created: 2026-10-30
updated: 2026-11-02
---
# Tools in LangChain

> You write a tool as a normal Python function with `@tool`, give it to `create_agent`, and the agent runs it whenever the model asks.

## Context

In [[The running example]], the researcher asks for `get_returns(days=90)`, and "our program runs it". Two questions come out of that:
- how does the model know the tool exists, and what `days` means?
- the result covers the Equity, Bond and Cash funds, but the model never named them. Where does the list of funds come from?

## Defining the tool

```python
from langchain.tools import tool, ToolRuntime
from pydantic import BaseModel, Field

class ReturnsInput(BaseModel):   # the arguments the model fills in
    days: int = Field(ge=5, le=365, description="How many past days to measure, from 5 to 365")

@tool(args_schema=ReturnsInput)
def get_returns(days: int, runtime: ToolRuntime[Mandate]) -> dict:
    """Total return of every fund in the mandate over the last `days` days, in %."""
    funds = runtime.context.funds    # ("equity", "bond", "cash"), set by our code
    ...
```

`@tool` turns the function into the description the model reads ([[What is a tool]]):
- the **name** is the function's name;
- the **description** is the docstring, so write it for the model: what the tool returns and when to use it;
- the **arguments** come from the schema: `days`, an integer from 5 to 365, with its description ([[Schemas in LangChain]]).

The `runtime` argument is **hidden from the model**: LangChain fills it in when it runs the tool, with the mandate your code passed in. So the model chooses *how many days*, but never *which funds*. What the runtime holds, and how to pass it, is in [[Runtime in LangChain]].

## Giving it to the agent

```python
researcher = create_agent(model, tools=[get_returns, get_volatility], context_schema=Mandate, ...)
```

That one argument does the rest: `create_agent` sends the tool descriptions with every model call, runs the tools the model asks for, and sends the results back ([[The agent loop in LangChain]]).

## When a call goes wrong

At turn 1 the model asks for `days=500`. The arguments don't match the schema, so **the tool doesn't run**. LangChain sends the error back to the model as the tool's result:

```text
Error invoking tool 'get_returns' with kwargs {'days': 500} with error:
 days: Input should be less than or equal to 365
 Please fix the error and try again.
```

At turn 2 the model reads it and asks for `days=90`.

That only covers **bad arguments**. If the function itself raises an error (say, the price database is down), the agent stops. To turn those errors into messages too, add middleware: `ToolErrorMiddleware`, or `ToolRetryMiddleware` to try again ([[Middleware in LangChain]]).

## Watch out
- **The docstring is a prompt.** A vague description gives vague calls.
- **Keep results small.** Every result is added to the conversation and resent on every turn ([[What is a turn]]).
- **Limit the calls.** Nothing stops a model that keeps calling tools, unless you add a limit (`ToolCallLimitMiddleware`).

## Related
[[LangChain Overview]] · [[The running example]] · [[What is a tool]] · [[Runtime in LangChain]] · [[Agents in LangChain]] · [[The agent loop in LangChain]] · [[Schemas in LangChain]] · [[Middleware in LangChain]]

## My notes & questions
