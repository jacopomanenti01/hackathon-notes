---
tags: [note]
topic: langchain
sources:
  - https://docs.langchain.com/oss/python/langchain/short-term-memory
  - https://docs.langchain.com/oss/python/langchain/agents
  - https://docs.langchain.com/oss/python/langchain/multi-agent/custom-workflow
status: current
created: 2026-10-30
updated: 2026-11-03
---
# State in LangChain

> The state is a dictionary that holds everything a run has produced so far. Each step reads what it needs from it and writes back only what it produced.

## What it is

When several steps work together, each one needs results from the others. They don't call each other or pass arguments to each other: they all read from, and write to, one dictionary, the **state**. A step reads the keys it needs and returns only the keys it produced; the rest of the state stays as it was.

## Its components

There are two levels of state, kept apart on purpose:

| Level | Holds | Lives |
|---|---|---|
| **each agent's own state** | its conversation (`messages`), its checked answer (`structured_response`), and any keys you added | inside one agent, for one run |
| **the workflow's shared state** | only the results the agents pass to each other | across the whole workflow ([[Workflow in LangChain]]) |

An agent's conversation holds every tool call and result it made, and the next agent doesn't need any of it: it only needs the result.

## How LangChain implements it

### An agent's own state

- **What's in it:** `create_agent` gives every agent an `AgentState` with two keys: `messages`, the conversation, and `structured_response`, the checked answer when the agent has a schema ([[Schemas in LangChain]]).
- **Adding keys:** subclass `AgentState` and pass it as `state_schema`, either to `create_agent` or to a middleware that needs the keys ([[Middleware in LangChain]]).
- **How it's updated:** a new message is *added* to `messages`, so the conversation grows turn by turn; any other key is *replaced* by the new value.

```python
from typing import NotRequired, TypedDict
from langchain.agents import AgentState, create_agent

class MyAgentState(AgentState):
    attempts: NotRequired[int]          # a new key, filled in later by a hook

agent = create_agent(model, tools=[...], state_schema=MyAgentState)
```

### A workflow's shared state

- **What's in it:** only the keys you declare, in a `TypedDict`, a dict whose keys and types are written down ([[Workflow in LangChain]]).
- **Adding keys:** add them to the `TypedDict`.
- **How it's updated:** each step returns a dict with only the keys it changed. LangGraph merges it into the state: the other keys stay as they were, and a key written again replaces its old value.

```python
class MyWorkflowState(TypedDict, total=False):
    question: str
    answer: str

def answer_step(state: MyWorkflowState) -> dict:
    return {"answer": "..."}            # only what this step produced; `question` is kept
```

### Rules for both

- **A key can be optional.** `total=False` (for the whole `TypedDict`) or `NotRequired[...]` (for one key) means the key may be missing until a step fills it in.
- **An undeclared key is silently dropped.** If a step returns a key the state doesn't declare, there's no error: the next step just doesn't find it.
- **A state lasts one run.** To continue a conversation over several `invoke` calls, give the agent a **checkpointer**, which saves the state, and pass the same `thread_id` each time. That's LangChain's short-term memory:

```python
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(model, tools=[...], checkpointer=InMemorySaver())
config = {"configurable": {"thread_id": "1"}}
agent.invoke({"messages": [...]}, config=config)
agent.invoke({"messages": [...]}, config=config)   # the agent remembers the first call
```

## Example

Take three agents that build a portfolio from three funds (equity, bond and cash), with at most 40% in equity: a researcher, a strategist with a risk check, and a writer. The shared state:

```python
from typing import NotRequired, TypedDict
from langchain.agents import AgentState

class PortfolioState(TypedDict, total=False):
    mandate: Mandate            # set at the start
    notes: str                  # researcher -> strategist
    approved_weights: dict      # strategist (after the risk check) -> writer
    risk_review: dict           # strategist -> writer: what the risk check changed
    market_note: str            # writer -> the end

class RiskState(AgentState):    # keys the risk check adds to the strategist's own state
    approved_weights: NotRequired[dict]
    risk_review: NotRequired[dict]
```

Each step runs its agent and copies only the result into the shared state:

```python
def research_step(state: PortfolioState) -> dict:
    result = researcher.invoke({"messages": [{"role": "user", "content": "Research the funds."}]},
                               context=state["mandate"])
    return {"notes": result["messages"][-1].content}
```

The shared state fills up step by step:
1. the researcher's own state holds its four [[What is a turn|turns]], with every return and volatility it asked for; only its notes go into the shared state, under `notes`;
2. the strategist reads `notes`; its risk check writes `approved_weights` (equity 40%, bond 40%, cash 20%) and `risk_review` (equity was cut from 55%) into its own state, and its step copies both into the shared state;
3. the writer reads both and writes `market_note`, which can explain why equity is at the limit.

## Watch out
- **Declare a key before you write it.** An undeclared key is dropped without any error.
- **Copy out only what the next step needs.** Passing a whole conversation along makes every later prompt longer and slower.

## Related
[[LangChain Overview]] · [[Workflow in LangChain]] · [[Agents in LangChain]] · [[Middleware in LangChain]] · [[Hooks in LangChain]] · [[The agent loop in LangChain]]

## My notes & questions
