---
tags: [note]
topic: langgraph
sources:
  - https://docs.langchain.com/oss/python/langgraph/graph-api
  - https://docs.langchain.com/oss/python/langgraph/use-graph-api
status: current
created: 2026-10-30
updated: 2026-11-03
---
# State in LangGraph

> The state is one shared dictionary for the whole run. Each node reads it and returns only the keys it changed, and LangGraph merges them in.

## What it is

The nodes of a graph never call each other or pass arguments to each other ([[Workflow in LangGraph]]). So how does one node get what another produced? They all share one dictionary, the **state**. A node reads the keys it needs and returns only the keys it produced; LangGraph merges them in, and the rest of the state stays as it was.

## Its components

- **The keys**: declared once, with the type of each value. Only declared keys are kept.
- **The updates**: what each node returns, a dict of only the keys it changed.
- **The reducers**: for each key, how an update is combined with the old value. Without one, the new value **replaces** the old; with one, the two are **combined**, e.g. new messages are appended to the conversation.

## How LangGraph implements it

- **Declaring the keys**: a `TypedDict`, a dict whose keys and types are written down, given to `StateGraph(MyState)`. With `total=False`, a key may be missing until a node fills it in.
- **Updating**: a node returns a dict, e.g. `{"notes": "..."}`. LangGraph merges it into the state, and the next node receives the merged state.
- **Adding a reducer**: write the key's type as `Annotated[type, reducer]`. The reducer is a function of the old and the new value:
  - `add_messages` appends new messages to a conversation (and replaces a message that has the same id);
  - `operator.add` concatenates two lists, which is what you need when two nodes running *in parallel* write the same key: without a reducer, LangGraph raises an error.
- **Reading the result**: `workflow.invoke(...)` returns the final state.

```python
from typing import Annotated, TypedDict
from langchain.messages import AnyMessage
from langgraph.graph.message import add_messages

class MyState(TypedDict, total=False):
    question: str                                          # replaced when written again
    messages: Annotated[list[AnyMessage], add_messages]    # appended to
```


## Example

Take a system that builds a portfolio from three funds (equity, bond and cash), with at most 40% in equity: a researcher with tools, a strategist, a risk check that may rebalance the weights, and a writer. Its state:

```python
class PortfolioState(TypedDict, total=False):
    messages: Annotated[list[AnyMessage], add_messages]   # researcher <-> tools
    notes: str                    # researcher -> strategist
    proposed_weights: dict        # strategist -> risk_check
    approved_weights: dict        # risk_check, then rebalance -> writer
    risk_review: dict             # risk_check -> writer: was equity within the limit?
    market_note: str              # writer -> the end
```

The comment on each key says who writes it and who reads it, so the class doubles as a map of the system. The state fills up as the run goes:
1. the researcher's four [[Introduction#What is a turn|turns]] add their requests and the tools' results to `messages`, one turn at a time: the reducer appends them. Its last reply also goes into `notes`;
2. the strategist reads `notes` and writes `proposed_weights`: equity 55%, bond 30%, cash 15%;
3. the risk check writes `approved_weights` (the same weights) and `risk_review` (equity over the limit);
4. the rebalance step returns a new `approved_weights`, equity 40%, bond 40%, cash 20%, which **replaces** the old one;
5. the writer reads both and writes `market_note`.

## Watch out
- **Declare a key before you write it.** A key a node returns that isn't in the state is silently dropped: no error, and the next node just doesn't find it.
- **A conversation needs its reducer.** Without `add_messages`, each turn would replace the whole conversation with its own new messages.
- **Keep big data out of `messages`.** Every model call that reads it resends it all.

## References
- [LangGraph docs: Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api)
- [LangGraph docs: Use the Graph API](https://docs.langchain.com/oss/python/langgraph/use-graph-api)

## Related
[[A Brief Overview on LangGraph]] · [[Workflow in LangGraph]] · [[The agent loop in LangGraph]] · [[Agents in LangGraph]] · [[State in LangChain]]
