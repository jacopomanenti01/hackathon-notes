---
tags: [note]
topic: langchain
sources:
  - https://docs.langchain.com/oss/python/langchain/runtime
  - https://docs.langchain.com/oss/python/langchain/tools
  - https://docs.langchain.com/oss/python/langchain/context-engineering
status: current
created: 2026-11-02
updated: 2026-11-02
---
# Runtime in LangChain

> The runtime is what your code can see during a run but the model can't: the **context** you passed in, the agent's **state**, and a long-term **store**. Tools and middleware receive it as an extra, hidden argument.

## Context

In [[The running example]], the mandate (the three funds, and at most 40% in equity) must reach three places:
- `get_returns` needs the **list of funds** to measure, but the model only gives `days=90`;
- the prompt must tell the model the mandate, so it proposes weights for those funds;
- the risk check needs the **limit**, 40%, to compare with the strategist's 55%.

If the mandate were a tool argument, the model would fill it in, and it could get it wrong (or change it). It must come from **our code**, and reach the tools and middleware without passing through the model. That's the job of the runtime.

## What's in the runtime

| Part | What it holds | Lasts | In the example |
|---|---|---|---|
| `context` | data you pass to `invoke(..., context=...)`, read-only by intent | one run | `Mandate(funds=("equity", "bond", "cash"), max_equity=0.40)` |
| `state` | the agent's state: `messages`, plus any keys you added ([[State in LangChain]]) | one run | the conversation so far |
| `store` | long-term memory you give to `create_agent(store=...)` | across runs | not used |

Tools also get `tool_call_id`, the id of the call they're answering.

## Example

**Declare** the context's shape, and **pass** it in at each run:

```python
from dataclasses import dataclass

@dataclass(frozen=True)          # frozen: nobody can change the mandate during the run
class Mandate:
    funds: tuple[str, ...]
    max_equity: float

researcher = create_agent(model, tools=[get_returns, get_volatility], context_schema=Mandate)
researcher.invoke({"messages": [...]}, context=Mandate(("equity", "bond", "cash"), 0.40))
```

**Read** it wherever you need it. The way in depends on where your code runs:

```python
@tool
def get_returns(days: int, runtime: ToolRuntime[Mandate]) -> dict:   # a tool: a ToolRuntime argument
    funds = runtime.context.funds                                       # hidden from the model

@dynamic_prompt
def mandate_prompt(request) -> str:                                    # a wrap hook: request.runtime
    m = request.runtime.context
    return f"{request.system_prompt}\n\nThe funds: {', '.join(m.funds)}. At most {m.max_equity:.0%} in equity."

class EquityCapGuardrail(AgentMiddleware):
    def after_agent(self, state, runtime):                             # a node hook: a runtime argument
        cap = runtime.context.max_equity
```

The `runtime` argument of a tool never appears in the tool's description: the model chooses `days`, never the funds ([[Tools in LangChain]]).

## Context or state?

| Put it in… | if… | Example |
|---|---|---|
| the **context** | it's fixed for the whole run, and the model must not change it | the funds, the equity limit, a user id |
| the **state** | it changes during the run, or a later step needs it | the messages, the approved weights |
| the **prompt** | the model must read it | the mandate, *shown* to the model by `mandate_prompt` |

The mandate is in the context *and* shown in the prompt: the model reads it, but can't change the value your code uses.

## Watch out
- **The context isn't checked by the model's schema**, and nothing stops your own code from changing it. Use `frozen=True` if it must never change.
- **Calling a tool by hand** (outside an agent) fails if it needs a runtime: LangChain fills it in only inside the agent ([[The agent loop in LangChain]]).

## Related
[[LangChain Overview]] · [[The running example]] · [[Tools in LangChain]] · [[Middleware in LangChain]] · [[Hooks in LangChain]] · [[State in LangChain]] · [[Agents in LangChain]]

## My notes & questions
