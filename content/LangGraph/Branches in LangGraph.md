---
tags: [note]
topic: langgraph
template_files: [src/systems/langgraph/steps.py, src/systems/langgraph/workflow.py]
status: current
created: 2026-10-30
updated: 2026-10-30
---
# Branches in LangGraph

> A branch is a fork in the graph: a small function, the **router**, reads the state and returns the name of the node to run next.

## Context

On 31 January 2025, `portfolio_manager` proposes weights with 25% in US assets. `risk_review` checks them against the 20% limit and records `approved: False`. What runs next depends on that result:
- **approved**: go straight to `risk_manager`;
- **not approved** (our date): go to `rebalance` first, which scales the US assets down to 20%, then to `risk_manager`.

On another date, with US at 15%, the same graph skips `rebalance`. One graph, two possible routes, decided at run time.

```mermaid
flowchart LR
    PM["portfolio_manager"] --> RR["risk_review"]
    RR -->|"after_review: approved"| RM["risk_manager"]
    RR -->|"after_review: not approved"| RB["rebalance"]
    RB --> RM
```

## Router + conditional edge

A **router** is a plain function: state in, name of the next node out.

```python
# steps.py
def after_review(state) -> Literal["risk_manager", "rebalance"]:
    return "risk_manager" if state["risk_review"]["approved"] else "rebalance"
```

A **conditional edge** attaches the router to a node: after `risk_review`, call `after_review` and go where it says.

```python
# workflow.py
graph.add_conditional_edges("risk_review", after_review, ["risk_manager", "rebalance"])
graph.add_edge("rebalance", "risk_manager")
```

The list `["risk_manager", "rebalance"]` names the possible destinations, so LangGraph can check the graph and draw it.

## The router decides, the nodes work

The router only *chooses*. The checking and fixing happen in plain-Python nodes, without any model:
- **`risk_review`** runs every rule in `RULES` and writes the result to `risk_review` in the state. It writes `approved_weights` only if the weights pass.
- **`rebalance`** scales US assets down and the others up, checks again, and writes `approved_weights`.

Because these are plain Python, the limit always holds, whatever the model proposed. To add a risk rule, write a function `weights -> violation message or None` and add it to `RULES` in `steps.py`.

## The other router in the graph

The analyst loop uses the same mechanism. `tools_condition` is a router LangGraph provides: "the last message has tool calls → `tools`, otherwise → `END`" ([[The agent loop in LangGraph]]). A conditional edge that points *back* to an earlier node makes a **loop**.

## Compared with LangChain

The LangChain system has no branch: the same check runs as middleware after the portfolio manager answers ([[Middleware in LangChain]]). Here the decision is visible in the graph, and in Langfuse you see `rebalance` only on the dates where it ran.

## Watch out
- **Return a name the edge knows.** If the router returns a node name that isn't among the destinations, the run fails.
- **Routers shouldn't change the state.** They only read it. Put the work in a node before the router.

## Related
[[Workflow in LangGraph]] · [[The agent loop in LangGraph]] · [[State in LangGraph]] · [[Middleware in LangChain]]

## My notes & questions
