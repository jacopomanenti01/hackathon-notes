---
tags: [note]
topic: langchain
sources:
  - https://docs.langchain.com/oss/python/langchain/middleware/overview
  - https://docs.langchain.com/oss/python/langchain/middleware/built-in
  - https://docs.langchain.com/oss/python/langchain/middleware/custom
  - https://docs.langchain.com/oss/python/langchain/guardrails
  - https://medium.com/@ale.garavaglia/langchain-middlewares-lightweight-hooks-for-more-structured-agents-f0abba828934
status: current
created: 2026-10-30
updated: 2026-11-02
---
# Middleware in LangChain

> Middleware is your own code, plugged into the agent's loop: it runs before, around or after the model and tool calls, and at the start or end of the agent.

## Context

Look at [[The running example]] and ask what isn't the model's job:
- **The mandate.** The model must know which funds it may use and that equity is capped at 40%, but those values live in the hidden context, not in the prompt ([[Runtime in LangChain]]).
- **The limit.** The strategist proposes 55% in equity. The 40% cap must hold whatever the model wrote.
- **A model that never stops**, asking for returns and volatility over and over.
- **A server error** from the model provider in the middle of the run.

`create_agent` runs the loop for you, so there's no place for your own `if`. **Middleware** is where that code goes, attached to the loop's [[Hooks in LangChain|hooks]]:

```python
researcher = create_agent(model, tools=[get_returns, get_volatility], context_schema=Mandate,
    middleware=[
        mandate_prompt,                              # adds the mandate to the system prompt
        ModelRetryMiddleware(max_retries=2),         # server error: retry the call
        ToolCallLimitMiddleware(run_limit=6),        # at most 6 tool calls
        ModelCallLimitMiddleware(run_limit=8),       # at most 8 model calls
    ])
strategist = create_agent(model, response_format=ToolStrategy(Allocation), context_schema=Mandate,
    middleware=[mandate_prompt, ModelCallLimitMiddleware(run_limit=8), EquityCapGuardrail()])
```

## Three ways to create one

| Way | Use it when | Example |
|---|---|---|
| **Built-in** class | a common need someone already solved | `ModelRetryMiddleware`, `ModelCallLimitMiddleware`, `ToolCallLimitMiddleware`; also summarizing long conversations, asking a human for approval, hiding personal data |
| **Decorator** on a function | one hook, simple logic | `@dynamic_prompt` (`mandate_prompt`), `@before_model`, `@wrap_model_call` |
| **Subclass** of `AgentMiddleware` | several hooks, settings, or new state keys | `EquityCapGuardrail` |

## Example

```python
from langchain.agents.middleware import AgentMiddleware, dynamic_prompt

@dynamic_prompt                                      # a decorator: builds the prompt before each call
def mandate_prompt(request) -> str:
    m = request.runtime.context
    return f"{request.system_prompt}\n\nThe funds: {', '.join(m.funds)}. At most {m.max_equity:.0%} in equity."

class EquityCapGuardrail(AgentMiddleware[RiskState]): # a class: runs once, after the answer
    state_schema = RiskState                          # adds approved_weights, risk_review to the state

    def after_agent(self, state, runtime):
        weights = state["structured_response"].weights
        total = sum(weights.values())
        weights = {f: w / total for f, w in weights.items()}          # make them sum to 1
        cap = runtime.context.max_equity
        if weights["equity"] <= cap:
            return {"approved_weights": weights, "risk_review": {"within_limit": True}}
        rest = 1 - weights["equity"]                                   # cap equity, scale the others up
        approved = {f: cap if f == "equity" else w * (1 - cap) / rest for f, w in weights.items()}
        return {"approved_weights": approved,
                "risk_review": {"within_limit": False, "equity_proposed": weights["equity"]}}
```

In the example, the strategist proposes equity 55%, bond 30%, cash 15%: the guardrail caps equity at 40% and approves **equity 40%, bond 40%, cash 20%**. It's plain Python, so the rule holds whatever the model said. This is a **deterministic guardrail** (a rule in code). A **model-based** guardrail would ask a second model to judge the answer.

## Watch out
- **Every agent needs a limit.** Without `ModelCallLimitMiddleware`, nothing stops a model that keeps calling tools. When it stops the agent, there is no answer: `structured_response` is `None`.
- **New keys need `state_schema`.** A key a middleware returns but never declared is silently dropped ([[State in LangChain]]).
- **Order matters.** The first middleware in the list wraps the others ([[Hooks in LangChain]]).

## Related
[[LangChain Overview]] · [[The running example]] · [[Hooks in LangChain]] · [[Runtime in LangChain]] · [[Agents in LangChain]] · [[State in LangChain]] · [[The agent loop in LangChain]]

## My notes & questions
