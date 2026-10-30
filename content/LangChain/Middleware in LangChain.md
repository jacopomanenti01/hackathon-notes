---
tags: [note]
topic: langchain
template_files: [src/systems/langchain/middleware.py, src/systems/langchain/agents.py]
status: current
created: 2026-10-30
updated: 2026-10-30
---
# Middleware in LangChain

> Middleware is your own code, plugged into the agent's loop: it runs before, around or after the model and tool calls, and at the start or end of the agent.

## Context

Look again at the [[Agents in LangChain#Context|running example]] and ask what could go wrong, or what's missing:
- **The date.** How does the model know the date is 31 January 2025? It isn't in the analyst's prompt (`ANALYST` in `prompts.py`).
- **A broken call.** At step 3 the model's reply might be cut off half-way through `{"lookback_days": 9`. Then the agent would stop without an answer.
- **A model that never stops** asking for statistics: 90 days, 120, 180, 200, ...
- **A server error** from the LLM provider in the middle of the run.

None of these is the model's job, and `create_agent` runs the loop for you, so there's nowhere to put your own `if`. **Middleware** is where that code goes. Here's the analyst's list, one item per problem:

```python
# agents.py (analyst)
middleware = [
    reporting_date_prompt,               # adds "The reporting date is 2025-01-31." to the prompt
    retry_invalid_tool_calls,            # broken call: show the error, ask again
    ModelRetryMiddleware(max_retries=2), # server error: retry the call
    tool_call_limit(),                   # at most MAX_TOOL_CALLS tool calls
    call_limit(),                        # at most MAX_MODEL_CALLS model calls
]
```

## How it plugs in

The loop has fixed moments where middleware can run, called **hooks**: before the agent starts, before, around and after each model call, around each tool call, and after the agent ends ([[Hooks in LangChain]]). A middleware is one or more functions, each attached to a hook. The agent runs them in list order.

## Three ways to create one

| Way | Use it when | Template example |
|---|---|---|
| **Prebuilt** class | a common need someone already solved | `ModelRetryMiddleware`, `ModelCallLimitMiddleware`, `ToolCallLimitMiddleware` |
| **Decorator** on a function | one hook, simple logic | `@dynamic_prompt` (`reporting_date_prompt`), `@wrap_model_call` (`retry_invalid_tool_calls`) |
| **Subclass** of `AgentMiddleware` | several hooks, or it must store new keys | `UsExposureGuardrail` |

```python
# A decorator: runs before every model call, returns the system prompt to use
@dynamic_prompt
def reporting_date_prompt(request):
    return f"{request.system_prompt}\n\nThe reporting date is {request.runtime.context.reporting_date}."

# A subclass: runs once, after the portfolio manager has answered, and stores new keys
class UsExposureGuardrail(AgentMiddleware[RiskState]):
    state_schema = RiskState            # adds approved_weights, risk_review to the agent's state
    def after_agent(self, state, runtime):
        proposed = state["structured_response"].normalized()
        approved = rebalance(proposed) if us_exposure(proposed) > MAX_US_EXPOSURE else proposed
        return {"approved_weights": approved, ...}
```

This is step 3 of [[The hackathon task]]: the guardrail is plain Python, so the 20% US limit holds whatever the model said. In the LangGraph system, the same check is a separate node.

## Watch out
- **Every agent needs a limit.** Without `call_limit()` and `tool_call_limit()`, nothing stops a model that keeps calling tools.
- **New keys need `state_schema`.** Middleware can only write keys declared in the agent's state ([[State in LangChain]]).
- **Order matters.** Middleware listed first wraps the ones after it.

## Related
[[Hooks in LangChain]] · [[Agents in LangChain]] · [[State in LangChain]] · [[The agent loop in LangChain]]


