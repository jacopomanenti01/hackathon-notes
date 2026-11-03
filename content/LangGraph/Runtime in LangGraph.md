---
tags: [note]
topic: langgraph
sources:
  - https://docs.langchain.com/oss/python/langgraph/graph-api
  - https://docs.langchain.com/oss/python/langgraph/use-graph-api
  - https://docs.langchain.com/oss/python/langchain/tools
status: current
created: 2026-11-03
updated: 2026-11-03
---
# Runtime in LangGraph

> The runtime is what your nodes and tools can see during a run but the model can't: above all the **context**, the fixed values you passed in when you started the run. They receive it as an extra, hidden argument.

## What it is

Some values a run needs should never be chosen by the model: the rules it must follow, the inputs it works on. If they were tool arguments, the model would fill them in, and it could get them wrong or change them. If they were in the state, any node could overwrite them. The **runtime** carries them from your code to the nodes and tools directly, fixed for the whole run, without passing through the model.

## Its components

- **The context**: fixed data for the run, set by your code when it starts the graph. It lasts one run, and nothing in the graph is meant to change it.
- **The store**: long-term memory, across runs (only if you give the graph one).
- **For tools only**: the graph's **state** ([[State in LangGraph]]), and the `tool_call_id` of the call they're answering.

## How LangGraph implements it

Using the context takes three steps:
1. **Declare its shape** with a dataclass, and give it to the graph: `StateGraph(MyState, context_schema=MyContext)`.
2. **Pass a value** each time you run the graph: `workflow.invoke(first_state, context=MyContext(...))`.
3. **Read it** where you need it. How you reach it depends on where your code runs:
   - **in a node**: add a `runtime: Runtime[MyContext]` argument (from `langgraph.runtime`) and read `runtime.context`. LangGraph passes it to any node that asks for it ([[Agents in LangGraph]]);
   - **in a tool**: add a `runtime: ToolRuntime[MyContext]` argument (from `langchain.tools`) and read `runtime.context`, or `runtime.state` for the graph's state. The `ToolNode` fills it in and leaves it out of the description the model sees ([[Tools in LangGraph]]).

**Context, state or prompt?** Each value of a run belongs in one place:
- **the context**, if it's fixed for the whole run and the model must not change it;
- **the state**, if it changes during the run, or a later step needs it;
- **the prompt**, if the model must read it. A value can be in the context *and* in the prompt: a node can copy it into the messages it builds, so the model reads it but can't change the value your code uses.

LangChain's runtime is the same idea, reached from tools and middleware instead of nodes ([[Runtime in LangChain]]).

## Example

Take a system that builds a portfolio under a **mandate**: three funds (equity, bond and cash), and at most 40% in equity. The mandate must reach three places: the returns tool (which funds to measure), the researcher's prompt (so the model knows the funds), and the risk check (the 40% limit).

```python
from dataclasses import dataclass

@dataclass(frozen=True)          # frozen: nobody can change the mandate during the run
class Mandate:
    funds: tuple[str, ...]
    max_equity: float

graph = StateGraph(PortfolioState, context_schema=Mandate)
...
workflow.invoke({"messages": [HumanMessage("Research the funds.")]},
                context=Mandate(("equity", "bond", "cash"), 0.40))
```

```python
from langchain.tools import ToolRuntime, tool
from langgraph.runtime import Runtime

@tool(args_schema=ReturnsInput)
def get_returns(days: int, runtime: ToolRuntime[Mandate]) -> dict:   # a tool
    funds = runtime.context.funds                                       # hidden from the model

def researcher(state: PortfolioState, runtime: Runtime[Mandate]) -> dict:   # a node
    system = SystemMessage(f"You are a market researcher. The funds: {', '.join(runtime.context.funds)}.")

def risk_check(state: PortfolioState, runtime: Runtime[Mandate]) -> dict:   # a node, no model
    within = weights["equity"] <= runtime.context.max_equity                # 0.40
```

The model asks for `get_returns(days=90)` and gets the returns of all three funds without ever naming them. The funds are also in the researcher's prompt, so the model reads them, but the value the tool and the risk check use comes from the context, and the model can't change it.

## Watch out
- **The context isn't checked by any schema**, and nothing stops your own code from changing it. Use `frozen=True` if it must never change.
- **Don't put fixed values in the state** just because every node can read it there: any node could overwrite them.
- **Calling a tool by hand** (outside the graph) fails if it needs a runtime: the `ToolNode` fills it in only inside the graph.

## References
- [LangGraph docs: Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api)
- [LangGraph docs: Use the Graph API](https://docs.langchain.com/oss/python/langgraph/use-graph-api)
- [LangChain docs: Tools](https://docs.langchain.com/oss/python/langchain/tools)

## Related
[[A Brief Overview on LangGraph]] · [[Tools in LangGraph]] · [[Agents in LangGraph]] · [[State in LangGraph]] · [[Runtime in LangChain]]
