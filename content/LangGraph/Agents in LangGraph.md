---
tags: [note]
topic: langgraph
sources:
  - https://docs.langchain.com/oss/python/langgraph/workflows-agents
  - https://docs.langchain.com/oss/python/langgraph/graph-api
  - https://docs.langchain.com/oss/python/langchain/models
  - https://docs.langchain.com/oss/python/langchain/messages
status: current
created: 2026-10-30
updated: 2026-11-03
---
# Agents in LangGraph

> In LangGraph, an agent is a node you write yourself: a function that builds the messages, calls the model, and turns the reply into state keys.

## What it is

A model on its own answers once, from what it already knows. An **agent** is a model that can use tools in a loop: ask for something, read the result, decide again, until it answers ([[Introduction#What is a loop|What is a loop]]). LangGraph has no agent class to configure. You write the agent as a node, and if it uses tools, you draw its loop in the graph ([[The agent loop in LangGraph]]). Nothing is hidden: every message the model receives is built in front of you.

## Its components

An agent node does four things, in plain Python:
1. **read** what it needs from the state ([[State in LangGraph]]), and the fixed values of the run from the context;
2. **build the messages**: a system message (its role and rules) and the input, or the conversation so far;
3. **call the model**, in one of three ways (below);
4. **return** the reply as state keys.

The way you call the model decides what can come back:
- **plain**: `model.invoke(messages)` returns text;
- **with tools**: `model.bind_tools(tools).invoke(messages)` returns text **or** tool calls, which a `ToolNode` then runs ([[Tools in LangGraph]]);
- **with a forced form**: bind a schema the model must fill in, and you get an answer in a fixed shape, which you check ([[Schemas in LangGraph]]).

## How LangGraph implements it

- **The node**: a function `def agent(state, runtime) -> dict`. LangGraph passes the state, and the `runtime`, if the function asks for it.
- **The context**: the fixed values of a run, which the model must not choose (the rules, the inputs it works on). Declare their shape with `StateGraph(..., context_schema=Mandate)`, pass them with `workflow.invoke(..., context=...)`, and read them in the node as `runtime.context`. Type the argument as `runtime: Runtime[Mandate]` (from `langgraph.runtime`) ([[Runtime in LangGraph]]).
- **The messages**: `SystemMessage` and `HumanMessage` from `langchain.messages`. The model reads only these: whatever isn't in a message, it doesn't know.
- **Adding it to the graph**: `graph.add_node("name", agent)`, with a `retry_policy` if the model's server may fail ([[Errors and retries in LangGraph]]).

## Example

Take a system that builds a portfolio from three funds (equity, bond and cash), under a **mandate**: those funds, and at most 40% in equity. The simplest agent is the writer, which explains the approved weights in a short market note:

```python
from langchain.messages import HumanMessage, SystemMessage

def writer(state: PortfolioState) -> dict:
    reply = model.invoke([
        SystemMessage("Write a short market note explaining the portfolio."),           # 2. messages
        HumanMessage(f"{state['approved_weights']} {state['risk_review']}"),            # 1. read
    ])                                                                                   # 3. call
    return {"market_note": reply.content}                                               # 4. state keys
```

The researcher, which measures the funds with tools, also reads the mandate from the context, and binds its tools:

```python
from langgraph.runtime import Runtime

def researcher(state: PortfolioState, runtime: Runtime[Mandate]) -> dict:
    funds = ", ".join(runtime.context.funds)                      # fixed for the run
    system = SystemMessage(f"You are a market researcher. The funds: {funds}.")
    reply = model.bind_tools([get_returns, get_volatility]).invoke([system, *state["messages"]])
    if reply.tool_calls:
        return {"messages": [reply]}                              # the tools node runs them next
    return {"messages": [reply], "notes": reply.content}          # done: the notes
```

The writer never sees the researcher's conversation: its prompt holds only the two keys it reads, the approved weights (equity 40%, bond 40%, cash 20%) and the risk review (equity was cut from 55%). Each agent starts from the messages its node builds, and nothing else.

## Watch out
- **Give the model what it needs, in a message.** If an agent needs a new piece of information, put it in the messages its node builds: the model knows nothing else.
- **Server errors stop the node.** Add a `retry_policy` to each node that calls a model.

## References
- [LangGraph docs: Workflows and agents](https://docs.langchain.com/oss/python/langgraph/workflows-agents)
- [LangGraph docs: Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api)
- [LangChain docs: Models](https://docs.langchain.com/oss/python/langchain/models)
- [LangChain docs: Messages](https://docs.langchain.com/oss/python/langchain/messages)

## Related
[[A Brief Overview on LangGraph]] · [[The agent loop in LangGraph]] · [[Tools in LangGraph]] · [[Runtime in LangGraph]] · [[Schemas in LangGraph]] · [[State in LangGraph]] · [[Agents in LangChain]]
