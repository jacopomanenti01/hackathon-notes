---
tags: [note]
topic: langgraph
template_files: [src/systems/langgraph/agents.py, src/systems/langgraph/prompts.py]
status: current
created: 2026-10-30
updated: 2026-10-30
---
# Agents in LangGraph

> In the LangGraph system, an agent is a node you write yourself: a function that builds the messages, calls the model, and turns the reply into state keys.

## Context

On 31 January 2025, the portfolio manager's job is: *"here are the signals for every asset; give a weight to each."* In the LangChain system, `create_agent` would handle this ([[Agents in LangChain]]). Here, the `portfolio_manager` node does every step itself, in plain Python:

1. **Read** what it needs from the state: the date, the signals, the analyst's rationale.
2. **Build the messages**: a `SystemMessage` (the role, from `prompts.py`) and a `HumanMessage` (the data, as JSON).
3. **Call the model**, and check the answer against the `Allocation` form.
4. **Return** the result as state keys: `proposed_weights` and `portfolio_rationale`.

```python
# agents.py (shortened)
def portfolio_manager(state, llm):
    data = {"reporting_date": state["reporting_date"], "signals": state["signals"],
            "analyst_rationale": state["analyst_report"]}                       # 1. read
    prompt = [SystemMessage(content=PORTFOLIO_MANAGER),
              HumanMessage(content=json.dumps(data))]                           # 2. messages
    allocation = ask_for_answer(llm, "portfolio_manager", prompt, Allocation)   # 3. call + check
    return {"proposed_weights": allocation.normalized(),
            "portfolio_rationale": allocation.rationale}                         # 4. state keys
```

Nothing is hidden: every message the model receives is built in front of you.

## Three ways to call the model

The template has three small helpers in `agents.py`, and every LLM node uses one of them. They differ in **what is bound to the model**, and so in what comes back:

| Helper | What is bound | The reply | Then | Who uses it |
|---|---|---|---|---|
| `ask_for_text` | nothing | text | the node uses the text | `risk_manager`: the explanation for the PDF |
| `ask_with_tools` | real tools (functions) | text **or** tool calls | the graph's `tools` node runs the calls, then comes back ([[The agent loop in LangGraph]]) | `analyst` |
| `ask_for_answer` | a Pydantic schema, as a tool the model **must** call | an object of that schema (`Analysis`, `Allocation`) | checked; a wrong answer is sent back with the error ([[Schemas in LangGraph]]) | `analyst_signals`, `portfolio_manager` |

The last row is a **forced output format**. The model can't reply in free text: it must fill in the schema's fields. This is the same thing `response_format=ToolStrategy(Analysis)` does in the LangChain system ([[Agents in LangChain]]), written out by hand.

```python
# agents.py (shortened)
summary  = ask_for_text(llm, "risk_manager", prompt)                         # free text
reply    = ask_with_tools(llm, "analyst", messages, [get_asset_statistics])  # text or tool calls
analysis = ask_for_answer(llm, "analyst_signals", prompt, Analysis)          # forced format
```

## Why the analyst is split in two

The analyst's work runs in two nodes:
- **`analyst`** researches: it calls `get_asset_statistics` as often as it needs, then writes its notes in plain text;
- **`analyst_signals`** takes those notes and the statistics, and fills in the `Analysis` form: one signal per asset.

Separating them keeps each step simple. Why the form is a schema bound to the model, and not the state's types, and how the two could be one node: [[Schemas in LangGraph#Who uses ask_for_answer, and why the analyst is split|Schemas in LangGraph]].

## The agents' memory

A hand-written agent sees **only the messages its node builds**. `portfolio_manager` never sees the analyst's conversation, only the keys `signals` and `analyst_report` from the state ([[State in LangGraph]]). Each LLM node starts from a fresh prompt.

## Watch out
- **Network errors are handled by the graph, not the node.** Each LLM node is added with `retry_policy=LLM_RETRY`, which reruns it on a timeout or server error ([[Errors and retries in LangGraph]]).
- **Give the model what it needs, in the message.** If a node needs a new piece of information, add it to the `data` dictionary. The model knows nothing else.

## Related
[[Workflow in LangGraph]] · [[The agent loop in LangGraph]] · [[Schemas in LangGraph]] · [[State in LangGraph]] · [[Agents in LangChain]]


