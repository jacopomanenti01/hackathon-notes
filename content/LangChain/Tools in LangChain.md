---
tags: [note]
topic: langchain
sources:
  - https://docs.langchain.com/oss/python/langchain/tools
  - https://docs.langchain.com/oss/python/langchain/runtime
  - https://huggingface.co/learn/agents-course/unit1/tools
status: current
created: 2026-10-30
updated: 2026-11-03
---
# Tools in LangChain

> You write a tool as a normal Python function with `@tool`, give it to `create_agent`, and the agent runs it whenever the model asks.

## What it is

A **tool** is a function the model can't run but can *ask* you to run: the model writes a request (this tool, with these arguments), your program runs it, and the result goes back to the model ([[What is a tool]]). This note is about how you write tools in LangChain, and what happens when a call goes wrong.

## Its components

| Component | What it is | Seen by the model? |
|---|---|---|
| **name** | how the model refers to the tool | yes |
| **description** | what the tool does and when to use it | yes |
| **arguments** | the inputs the model fills in, with their types and limits | yes |
| **hidden inputs** | values your code provides, like a list of funds ([[Runtime in LangChain]]) | no |
| **result** | what the function returns, sent back to the model | yes, after the call |

## How LangChain implements it

| Piece | Where it comes from |
|---|---|
| `@tool` | turns a Python function into a tool |
| name | the function's name |
| description | the function's docstring |
| arguments | the type hints, or a Pydantic schema passed as `@tool(args_schema=...)` ([[Schemas in LangChain]]) |
| hidden inputs | a `runtime: ToolRuntime[...]` argument, filled in by LangChain and never shown to the model |
| giving it to the agent | `create_agent(model, tools=[...])` |

`create_agent` then sends the tool descriptions with every model call, runs the tools the model asks for, and adds the results to the conversation ([[The agent loop in LangChain]]).

**When a call goes wrong:**
- **bad arguments** (outside the schema): the tool doesn't run, and the error goes back to the model as the tool's result, so it can fix its call;
- **an error inside the function** (e.g. a database is down): the agent stops, unless you add middleware that turns errors into messages (`ToolErrorMiddleware`) or retries the call (`ToolRetryMiddleware`) ([[Middleware in LangChain]]).

## Example

Take a researcher agent that measures the returns of three funds (equity, bond and cash). Its tool:

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

researcher = create_agent(model, tools=[get_returns, get_volatility], context_schema=Mandate, ...)
```

The model reads the name `get_returns`, the docstring, and `days` (an integer from 5 to 365). It never sees `runtime`, so it chooses *how many days*, never *which funds*.

On its first [[What is a turn|turn]], the model asks for `days=500`. The arguments don't match the schema, so the tool doesn't run, and LangChain sends back:

```text
Error invoking tool 'get_returns' with kwargs {'days': 500} with error:
 days: Input should be less than or equal to 365
 Please fix the error and try again.
```

On the next turn, the model asks for `days=90` and gets equity +4%, bond +1%, cash +0.5%.

## Watch out
- **The docstring is a prompt.** A vague description gives vague calls.
- **Keep results small.** Every result is added to the conversation and resent on every turn ([[What is a turn]]).
- **Limit the calls.** Nothing stops a model that keeps calling tools, unless you add a limit (`ToolCallLimitMiddleware`: [[Limits in LangChain]]).

## Related
[[LangChain Overview]] · [[What is a tool]] · [[Runtime in LangChain]] · [[Agents in LangChain]] · [[The agent loop in LangChain]] · [[Schemas in LangChain]] · [[Middleware in LangChain]]

## My notes & questions
