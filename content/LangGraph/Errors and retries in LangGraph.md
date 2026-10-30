---
tags: [note]
topic: langgraph
template_files: [src/systems/langgraph/workflow.py, src/systems/langgraph/agents.py, src/systems/langgraph/tools.py, src/config.py]
status: current
created: 2026-10-30
updated: 2026-10-30
---
# Errors and retries in LangGraph

> Four different things can go wrong in a run, and the template has a different guard for each. Knowing which is which tells you where to look when a date fails.

## Context

On 31 January 2025, imagine four bad moments in the LangGraph system's run:
1. The model asks for statistics with `lookback_days=1`, below the minimum of 5.
2. `analyst_signals` answers with a missing asset.
3. The model server times out while `portfolio_manager` is waiting.
4. The analyst keeps asking for new windows and never stops.

None of them should crash the date immediately. Each has its own fix:

| Problem | Guard | Where (in `src/`) | What happens | When it gives up |
|---|---|---|---|---|
| 1. a **tool call fails** (bad arguments, or the tool raises `ValueError`) | `tool_error` | `systems/langgraph/workflow.py:41`, `systems/langgraph/tools.py:63` | the error becomes the tool's result, and the model calls again ([[Tools in LangGraph]]) | the analyst's round limit (row 4) |
| 2. an **answer is invalid** | corrections in `ask_for_answer` | `systems/langgraph/agents.py:84–98`; `AGENT_MAX_CORRECTIONS` = 4 in `config.py:11` | the error goes back, and the model answers again ([[Schemas in LangGraph]]) | after 4 corrections: the date fails |
| 3. a **node raises** (timeout, server error) | `RetryPolicy` | `systems/langgraph/workflow.py:30`, used at lines 37, 44, 47, 52 | LangGraph waits and runs **the whole node** again | after 3 attempts: the date fails |
| 4. the model **keeps calling tools** | round limit | `systems/langgraph/agents.py:113`, `:132` | the tool is no longer offered, so the model must answer ([[The agent loop in LangGraph]]) | — |
| any loop with no other limit | `recursion_limit` | `systems/langgraph/workflow.py:73`; `RECURSION_LIMIT` = 40 in `config.py:14` | `GraphRecursionError` | immediately: the date fails |

## Each guard in the code

### 1. A tool call fails: `tool_error`
```python
# systems/langgraph/workflow.py:41: the ToolNode catches the error...
graph.add_node("tools", ToolNode([get_asset_statistics], handle_tool_errors=tool_error))

# systems/langgraph/tools.py:63: ...and this text becomes the tool's result
def tool_error(error: ValueError | ToolInvocationError) -> str:
    return f"Error: {error}\nFix the arguments and call the tool again."
```
It catches bad arguments (`ToolInvocationError`), and the `ValueError` the tool raises itself (`tools.py:40`: "No prices in the last N days; use a longer window"). The graph then goes back to `analyst`, which can call again.

### 2. An answer is invalid: the correction loop
```python
# systems/langgraph/agents.py:84–98 (ask_for_answer)
model = llm.bind_tools([schema], tool_choice="required")
messages = list(prompt)
for attempt in range(AGENT_MAX_CORRECTIONS + 1):          # 4 corrections = 5 tries
    reply = model.invoke(messages, config={"run_name": f"{agent}_llm"})
    try:
        if not reply.tool_calls:                            # no call, or broken JSON
            raise ValueError(f"call {schema.__name__} with your answer")
        return schema.model_validate(reply.tool_calls[0]["args"])   # validators run here
    except ValueError as error:
        messages.append(correction(error, attempt))         # the error goes back to the model
raise ValueError(f"{agent} gave no valid {schema.__name__}")
```
`correction` (`agents.py:101`) writes the message: "Your last answer was invalid: … Answer again. Attempts left: N." Used by `analyst_signals` and `portfolio_manager`.

### 3. A node raises: `RetryPolicy`
```python
# systems/langgraph/workflow.py:30, then on every LLM node (lines 37, 44, 47, 52)
LLM_RETRY = RetryPolicy(max_attempts=3)
graph.add_node("analyst", partial(analyst, llm=llm), retry_policy=LLM_RETRY)
graph.add_node("analyst_signals", partial(analyst_signals, llm=llm), retry_policy=LLM_RETRY)
graph.add_node("portfolio_manager", partial(portfolio_manager, llm=llm), retry_policy=LLM_RETRY)
graph.add_node("risk_manager", partial(risk_manager, llm=llm), retry_policy=LLM_RETRY)
```
If the node raises, LangGraph waits a little (longer each time) and calls the node function again from the start, up to 3 attempts. It's for problems that a second try can fix: the network, an overloaded server. The plain-Python nodes (`risk_review`, `rebalance`) and `tools` have no retry policy.

### 4. The model keeps calling tools: the round limit
```python
# systems/langgraph/agents.py:113 and :132 (analyst)
MAX_TOOL_ROUNDS = 4
tools = [get_asset_statistics] if rounds < MAX_TOOL_ROUNDS else []   # no tools: it must answer
```

### Backstop: the recursion limit
```python
# systems/langgraph/workflow.py:73
return graph.compile().with_config(recursion_limit=RECURSION_LIMIT)    # 40 node runs per date
```

## Two kinds of error, two reactions

- **The model can fix it** (bad arguments, invalid answer): send the error *to the model* as a message. That's rows 1 and 2.
- **The model can't fix it** (network, server): *run the step again*. That's row 3.

In a tool or a validator, raise `ValueError` with a clear message for anything the model can correct. Any other exception stops the date.

## Watch out
- **Retries multiply.** One node with 3 attempts × 5 answers each can mean 15 model calls. Keep the limits small.
- **A reply cut off at `max_tokens`.** Reasoning models (like `gpt-oss`) count their hidden reasoning towards `max_tokens`, so a reply can be cut off before it calls anything. `core/llm.py` then leaves `reply.tool_calls` empty. In `ask_for_answer`, that's an ordinary correction ("call Analysis with your answer"). In `analyst` it isn't caught: the cut-off reply counts as the final notes, possibly empty, and the graph moves on. If that happens, raise `max_tokens` in the notebook's `get_llm(...)`.
- **Line numbers drift.** They're correct for the template as of 30 Oct 2026. Search for the names if they've moved.
- **A failed date shows in Langfuse** as an error on that date's trace: open it to see which guard gave up.

## Related
[[Tools in LangGraph]] · [[Schemas in LangGraph]] · [[The agent loop in LangGraph]] · [[Workflow in LangGraph]] · [[Middleware in LangChain]]

