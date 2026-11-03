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
updated: 2026-11-03
---
# Middleware in LangChain

> Middleware is your own code, plugged into the agent's loop: it runs before, around or after the model and tool calls, and at the start or end of the agent.

## What it is

Some things are not the model's job: enforcing a rule whatever the model wrote, stopping a loop that never ends, retrying when the provider fails, adding information to the prompt. Because `create_agent` runs the loop for you, that code needs a place to go. **Middleware** is that place: code attached to the loop's hooks, the fixed moments where LangChain calls you ([[Hooks in LangChain]]).

## Its components

Middleware comes in three kinds:
- **built-in**: a ready-made class from LangChain, for a common need someone already solved;
- **decorator**: a function with a hook decorator, for one hook and simple logic;
- **class**: a subclass of `AgentMiddleware`, for several hooks, settings, or new keys in the state.

## How LangChain implements it

Whatever the kind, you give an agent a list, `create_agent(middleware=[...])`, and the order matters ([[Hooks in LangChain]]). Everything is imported from `langchain.agents.middleware`.

### Built-in middleware

A class you create with a few settings and add to the list. The most useful ones:
- `ModelRetryMiddleware`: retries a model call that failed (timeout, server error);
- `ModelCallLimitMiddleware` and `ToolCallLimitMiddleware`: stop the loop after too many model or tool calls ([[Limits in LangChain]]);
- `ToolErrorMiddleware` and `ToolRetryMiddleware`: turn a tool's error into a message for the model, or retry the tool;
- `ModelFallbackMiddleware`: tries another model when the first one fails;
- `SummarizationMiddleware`: summarizes old messages when the conversation gets long;
- `HumanInTheLoopMiddleware`: pauses for a human's approval before some tool calls;
- `PIIMiddleware`: detects and hides personal data.

Their parameters, and the full list, are in the [official docs](https://docs.langchain.com/oss/python/langchain/middleware/built-in).

```python
from langchain.agents.middleware import ModelRetryMiddleware, ModelCallLimitMiddleware

agent = create_agent(model, tools=[...], middleware=[
    ModelRetryMiddleware(max_retries=2),        # server error: retry the call
    ModelCallLimitMiddleware(run_limit=8),      # at most 8 model calls
])
```

### Decorator

A plain function, attached to one hook by its decorator: `@before_agent`, `@before_model`, `@after_model`, `@after_agent`, `@wrap_model_call`, `@wrap_tool_call`, or `@dynamic_prompt`, a shortcut that only changes the system prompt. The function receives what its hook gives it ([[Hooks in LangChain]]), and you add it to the list as it is.

```python
from langchain.agents.middleware import before_model

@before_model                                   # runs before each model call
def log_turn(state, runtime):
    print(f"model call with {len(state['messages'])} messages")
    return None                                 # nothing to change in the state

agent = create_agent(model, tools=[...], middleware=[log_turn])
```

### Class

A subclass of `AgentMiddleware`, with one method per hook it uses, e.g. `before_model(self, state, runtime)` or `after_agent(self, state, runtime)`. You add an instance to the list. Choose a class over a decorator when the middleware needs at least one of:
- **settings**: values you choose when you create it, passed to `__init__`;
- **several hooks** that work together, sharing the same settings and data;
- **new keys in the state** used by those hooks: the class declares them once, with `state_schema`, or they're silently dropped ([[State in LangChain]]).

Say you want to count an agent's [[What is a turn|turns]]: warn when a run takes too many, and keep the count in the result so your code can check it. That needs all three:

```python
from typing import NotRequired
from langchain.agents import AgentState
from langchain.agents.middleware import AgentMiddleware

class TurnState(AgentState):                    # the agent's state, plus one key
    turns: NotRequired[int]

class TurnTracker(AgentMiddleware[TurnState]):
    state_schema = TurnState                    # new key: declare it, or it's dropped

    def __init__(self, warn_after: int):        # a setting, chosen per agent
        super().__init__()
        self.warn_after = warn_after

    def before_agent(self, state, runtime):     # hook 1: once, at the start
        return {"turns": 0}

    def after_model(self, state, runtime):      # hook 2: after each model reply
        turns = state["turns"] + 1
        if turns == self.warn_after:
            print(f"turn {turns}: the agent is taking long")
        return {"turns": turns}                 # update the state

    def after_agent(self, state, runtime):      # hook 3: once, at the end
        print(f"done in {state['turns']} turns")

agent = create_agent(model, tools=[...], middleware=[TurnTracker(warn_after=3)])
result = agent.invoke({"messages": [...]})
```

For a run of four turns, it prints `turn 3: the agent is taking long`, then `done in 4 turns`, and `result["turns"]` is `4`. With decorators you'd need three separate functions, each declaring the same `state_schema`, and the setting hard-coded in them: the class keeps it all in one place, and you can reuse it with another value, e.g. `TurnTracker(warn_after=5)`.

## Example

Take two agents that build a portfolio from three funds (equity, bond and cash), under a **mandate** of at most 40% in equity: a researcher that measures returns and risk with tools, and a strategist that proposes the weights. Their lists mix all three kinds:

```python
researcher = create_agent(model, tools=[get_returns, get_volatility], context_schema=Mandate,
    middleware=[
        mandate_prompt,                              # decorator: adds the mandate to the prompt
        ModelRetryMiddleware(max_retries=2),         # built-in
        ToolCallLimitMiddleware(run_limit=6),        # built-in
        ModelCallLimitMiddleware(run_limit=8),       # built-in
    ])
strategist = create_agent(model, response_format=ToolStrategy(Allocation), context_schema=Mandate,
    middleware=[mandate_prompt, ModelCallLimitMiddleware(run_limit=8), EquityCapGuardrail()])  # + a class
```

The decorator and the class are your own:

```python
from langchain.agents.middleware import AgentMiddleware, dynamic_prompt

@dynamic_prompt                                      # builds the prompt before each model call
def mandate_prompt(request) -> str:
    m = request.runtime.context
    return f"{request.system_prompt}\n\nThe funds: {', '.join(m.funds)}. At most {m.max_equity:.0%} in equity."

class EquityCapGuardrail(AgentMiddleware[RiskState]): # runs once, after the strategist answers
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

The strategist answers equity 55%, bond 30%, cash 15%. The answer has the right form, so its schema accepts it ([[Schemas in LangChain]]), but it breaks the mandate. The guardrail caps equity at 40% and approves **equity 40%, bond 40%, cash 20%**. It's plain Python, so the rule holds whatever the model said: a **deterministic guardrail**. A **model-based** guardrail would ask a second model to judge the answer.

## Watch out
- **Every agent needs a limit.** Without `ModelCallLimitMiddleware`, nothing stops a model that keeps calling tools. When it stops the agent, there is no answer: `structured_response` is `None` ([[Limits in LangChain]]).
- **New keys need `state_schema`.** A key a middleware returns but never declared is silently dropped ([[State in LangChain]]).
- **Order matters.** The first middleware in the list wraps the others ([[Hooks in LangChain]]).

## Related
[[LangChain Overview]] · [[Hooks in LangChain]] · [[Runtime in LangChain]] · [[Agents in LangChain]] · [[State in LangChain]] · [[The agent loop in LangChain]] · [[Limits in LangChain]]

## My notes & questions
