---
tags: [note]
topic: langgraph
template_files: [src/systems/langgraph/state.py]
status: current
created: 2026-10-30
updated: 2026-10-30
---
# State in LangGraph

> The state is one shared dictionary for the whole run. Each node reads it and returns only the keys it changes, and LangGraph merges them in.

## Context

On 31 January 2025 ([[Workflow in LangGraph]]), seven nodes run one after the other, and none of them calls another. So how does `portfolio_manager` get the signals that `analyst_signals` produced?

They all share one dictionary, the **state**. Here it is, filling up as the date runs *(values shortened)*:

| After node | Keys it returned | The state now has |
|---|---|---|
| start (`core/run.py`) | `reporting_date: "2025-01-31"` | 1 key |
| `analyst` and `tools` (several rounds) | `messages`: the research conversation | + `messages` |
| `analyst_signals` | `signals`, `analyst_report` | + 2 |
| `portfolio_manager` | `proposed_weights` (US at 25%), `portfolio_rationale` | + 2 |
| `risk_review` | `risk_review: {approved: False, ...}` | + 1 |
| `rebalance` | `approved_weights` (US at 20%), `risk_review` (updated) | + 1, 1 replaced |
| `risk_manager` | `risk_summary` | + 1 |

At the end, `core/` writes `approved_weights` to `weights.csv`, and `report.py` turns the rest into the PDF.

## Declaring the keys

The keys are declared once, in `state.py`, as a `TypedDict`: a dictionary type that lists its keys and their value types. `total=False` means a key may be missing, because each key only appears once its node has run.

```python
# state.py (shortened)
class PortfolioState(TypedDict, total=False):
    reporting_date: str                                     # core -> every node, and the tool
    messages: Annotated[list[AnyMessage], add_messages]     # analyst <-> tools -> analyst_signals
    signals: dict[str, str]                                 # analyst_signals -> portfolio_manager, PDF
    approved_weights: dict[str, float]                      # risk_review or rebalance -> CSV (required)
    ...
```

The comment on each key says who writes it and who reads it, so the file doubles as a map of the system.

## Replace or combine: reducers

When a node returns a key that already exists, what happens to the old value?

- **By default, it's replaced.** `rebalance` returns a new `risk_review`, and the old one is gone.
- **With a reducer, the two are combined.** A **reducer** is a function, attached with `Annotated[type, function]`, that merges the old and new values. `messages` uses `add_messages`, so each round of the analyst loop *appends* its new messages instead of wiping the conversation:

```python
# analyst node: returns only the new reply...
return {"messages": [reply]}
# ...and add_messages adds it to the end of the existing list
```

Without the reducer, each round of the loop would erase the previous ones ([[The agent loop in LangGraph]]).

You also need a reducer when two nodes running *in parallel* write the same key (e.g. `Annotated[list[str], operator.add]`). Otherwise LangGraph raises an error.

## Compared with LangChain

The LangChain system has two levels of state: the workflow's, and each agent's own, with the agent's conversation kept inside the agent ([[State in LangChain]]). Here there's **one** state: the analyst's conversation (`messages`) lives in it, next to the signals and weights.

## Watch out
- **Declare a key before using it.** A key a node returns that isn't in `PortfolioState` is silently dropped: no error, and the next node just doesn't find it.
- **`approved_weights` is required.** `core/` writes it to `weights.csv`. Keep the keys `report.py` reads.

## Related
[[Workflow in LangGraph]] · [[The agent loop in LangGraph]] · [[Agents in LangGraph]] · [[State in LangChain]]


