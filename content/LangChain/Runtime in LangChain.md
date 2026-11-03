---
tags: [note]
topic: langchain
sources:
  - https://docs.langchain.com/oss/python/langchain/runtime
  - https://docs.langchain.com/oss/python/langchain/tools
  - https://docs.langchain.com/oss/python/langchain/context-engineering
status: current
created: 2026-11-02
updated: 2026-11-03
---
# Runtime in LangChain

> The runtime is what your code can see during a run but the model can't: the **context** you passed in, the agent's **state**, and a long-term **store**. Tools and middleware receive it as an extra, hidden argument.

## What it is

Some values a run needs should never be chosen by the model. If they were tool arguments, the model would fill them in, and it could get them wrong or change them. The **runtime** carries them from your code to the tools and middleware directly, without passing through the model.

## Its components

| Part | What it holds | Lasts | Set by |
|---|---|---|---|
| `context` | fixed data for the run (read-only by intent) | one run | your code, when it starts the agent |
| `state` | the agent's state: `messages`, plus any keys you added ([[State in LangChain]]) | one run | the agent, as it runs |
| `store` | long-term memory | across runs | your code, via `create_agent(store=...)` |

Tools also get `tool_call_id`, the id of the call they're answering.

## How LangChain implements it

Using the context takes three steps:
1. **Declare its shape** with a dataclass, and give it to the agent: `create_agent(context_schema=...)`.
2. **Pass a value** each time you run the agent: `agent.invoke(..., context=...)`.
3. **Read it** where you need it. How you reach it depends on where your code runs:
   - **in a tool**: add a `runtime: ToolRuntime[...]` argument and read `runtime.context`. LangChain fills it in and hides it from the model ([[Tools in LangChain]]);
   - **in a node-style hook**, e.g. `after_agent`: the hook already receives a `runtime` argument, so read `runtime.context` ([[Hooks in LangChain]]);
   - **in a wrap-style hook**, e.g. `@dynamic_prompt`: the runtime comes with the request, so read `request.runtime.context`.

**Context, state or prompt?** Each value of a run belongs in one place:
- **the context**, if it's fixed for the whole run and the model must not change it;
- **the state**, if it changes during the run, or a later step needs it ([[State in LangChain]]);
- **the prompt**, if the model must read it. A value can be in the context *and* in the prompt: middleware can copy it there, so the model reads it but can't change the value your code uses.

## Example

Take agents that build a portfolio under a **mandate**: three funds (equity, bond and cash), and at most 40% in equity. The mandate must reach three places: the returns tool (which funds to measure), the prompt (so the model knows the rules), and the risk check (the 40% limit).

```python
from dataclasses import dataclass

@dataclass(frozen=True)          # frozen: nobody can change the mandate during the run
class Mandate:
    funds: tuple[str, ...]
    max_equity: float

researcher = create_agent(model, tools=[get_returns, get_volatility], context_schema=Mandate)
researcher.invoke({"messages": [...]}, context=Mandate(("equity", "bond", "cash"), 0.40))
```

```python
@tool
def get_returns(days: int, runtime: ToolRuntime[Mandate]) -> dict:   # a tool
    funds = runtime.context.funds                                       # hidden from the model

@dynamic_prompt
def mandate_prompt(request) -> str:                                    # a wrap-style hook
    m = request.runtime.context
    return f"{request.system_prompt}\n\nThe funds: {', '.join(m.funds)}. At most {m.max_equity:.0%} in equity."

class EquityCapGuardrail(AgentMiddleware):
    def after_agent(self, state, runtime):                             # a node-style hook
        cap = runtime.context.max_equity                               # 0.40
```

The model asks for `get_returns(days=90)` and gets the returns of all three funds without ever naming them ([[Tools in LangChain]]). The mandate is in the context *and* shown in the prompt: the model reads it, but can't change the value your code uses.

## Watch out
- **The context isn't checked by any schema**, and nothing stops your own code from changing it. Use `frozen=True` if it must never change.
- **Calling a tool by hand** (outside an agent) fails if it needs a runtime: LangChain fills it in only inside the agent ([[The agent loop in LangChain]]).

## References
- [LangChain docs: Runtime](https://docs.langchain.com/oss/python/langchain/runtime)
- [LangChain docs: Tools](https://docs.langchain.com/oss/python/langchain/tools)
- [LangChain docs: Context engineering](https://docs.langchain.com/oss/python/langchain/context-engineering)

## Related
[[A Brief Overview on LangChain]] · [[Tools in LangChain]] · [[Middleware in LangChain]] · [[Hooks in LangChain]] · [[State in LangChain]] · [[Agents in LangChain]] · [[Runtime in LangGraph]]
