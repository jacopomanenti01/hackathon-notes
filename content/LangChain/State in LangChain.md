---
tags: [note]
topic: langchain
sources:
  - https://docs.langchain.com/oss/python/langchain/short-term-memory
  - https://docs.langchain.com/oss/python/langchain/agents
  - https://docs.langchain.com/oss/python/langchain/multi-agent/custom-workflow
status: current
created: 2026-10-30
updated: 2026-11-02
---
# State in LangChain

> The state is a dictionary that holds everything a run has produced so far. Each step reads what it needs from it and writes back only what it produced.

## Context

In [[The running example]], three agents work one after the other, and each needs something from the one before:
- the **researcher** needs the mandate, and produces notes;
- the **strategist** needs the notes, and produces weights, which the risk check turns into approved weights;
- the **writer** needs the approved weights, and produces the market note.

The agents don't call each other. They all read from, and write to, one dictionary: the **state**. There are actually **two kinds** of state, one for the workflow and one inside each agent.

## The workflow's state

The workflow (researcher → strategist → writer, [[Workflow in LangChain]]) shares one dictionary. Its keys are declared as a `TypedDict`, a normal dict whose keys and types are written down; `total=False` means a key may be missing until a step fills it in:

```python
from typing import TypedDict

class PortfolioState(TypedDict, total=False):
    mandate: Mandate            # set at the start
    notes: str                  # researcher -> strategist
    approved_weights: dict      # strategist (after the risk check) -> writer
    risk_review: dict           # strategist -> writer: what the risk check changed
    market_note: str            # writer -> the end
```

Each step returns **only the keys it changes**, e.g. the researcher returns `{"notes": "Equity: +4% ..."}`. They are **merged** into the state; a key written again replaces its old value.

## Each agent's own state

Inside each agent there's a **second, separate state**, the one its loop uses ([[The agent loop in LangChain]]):
- `messages`: the conversation. New messages are *added* to the list, never replaced;
- `structured_response`: the checked answer, if the agent has a `response_format`.

The two are kept apart on purpose. The researcher's conversation holds every return and volatility it fetched, and the strategist doesn't need any of it. So the step copies out only the result:

```python
def research_step(state: PortfolioState) -> dict:
    result = researcher.invoke({"messages": [{"role": "user", "content": "Research the funds."}]},
                               context=state["mandate"])       # the agent's own state, at the end
    return {"notes": result["messages"][-1].content}            # into the workflow's state
```

## Adding keys to an agent's state

The risk check runs inside the strategist, as middleware ([[Middleware in LangChain]]), and must store the approved weights. But an agent's state only has `messages` and `structured_response`. So you declare new keys with a subclass of `AgentState`, and attach it with `state_schema`:

```python
from typing import NotRequired
from langchain.agents import AgentState

class RiskState(AgentState):
    approved_weights: NotRequired[dict]     # NotRequired: may be missing
    risk_review: NotRequired[dict]

class EquityCapGuardrail(AgentMiddleware[RiskState]):
    state_schema = RiskState                # create_agent adds these keys to the strategist's state
```

The strategist's step then reads `result["approved_weights"]` and copies it into the workflow's state. (`create_agent(state_schema=...)` adds keys the same way, without middleware.)

## Watch out
- **Declare a key before you write it.** A step or a middleware that returns an undeclared key gets no error: the key is **silently dropped**, and the next step just doesn't find it.
- **A state lasts one run.** To keep a conversation across several `invoke` calls (a chat), give the agent a **checkpointer** and the same `thread_id` each time; that's LangChain's short-term memory.

## Related
[[LangChain Overview]] · [[The running example]] · [[Workflow in LangChain]] · [[Agents in LangChain]] · [[Middleware in LangChain]] · [[Hooks in LangChain]] · [[The agent loop in LangChain]]

## My notes & questions
