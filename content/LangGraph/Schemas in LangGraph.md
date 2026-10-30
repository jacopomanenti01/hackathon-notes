---
tags: [note]
topic: langgraph
template_files: [src/systems/langgraph/schema.py, src/systems/langgraph/agents.py]
status: current
created: 2026-10-30
updated: 2026-10-30
---
# Schemas in LangGraph

> A schema is a form the model must fill in. `ask_for_answer` hands the model the form, checks what it wrote, and sends back the errors until the answer is valid.

## Context

On 31 January 2025, `analyst_signals` has the analyst's research notes and must give one signal per asset. The next node, `portfolio_manager`, relies on getting exactly that: every asset, and only `high`, `neutral` or `low`. Free text won't do:

```text
"SPX looks very strong, NDX is fine, I'd avoid rates."
```

That answer has three problems: "very strong" isn't an allowed value, assets are missing, and code can't read it reliably. So the node uses the `Analysis` schema, and the run looks like this:

| Attempt | The model answers | The check says | Then |
|---|---|---|---|
| 1 | `{"SPX": "high", "NDX": "neutral"}` | `Missing: ['EUROSTR=', 'USDSOFR=', ...]` | the error goes back: "Your last answer was invalid: … Answer again. Attempts left: 4." |
| 2 | every asset, valid values | passes | the node returns `signals` and `analyst_report` |

## The schema: an instruction and a check

```python
# schema.py (shortened)
class Analysis(BaseModel):
    """One high, neutral, or low signal per asset and a concise rationale."""      # the model reads this
    signals: dict[str, Literal["high", "neutral", "low"]] = Field(description="One signal for every asset")
    rationale: str = Field(description="Concise rationale")

    @field_validator("signals")
    @classmethod
    def _every_asset(cls, signals):
        validate_asset_names(signals)     # raises "Missing: [...]; unexpected: [...]"
        return signals
```

A schema does two jobs. Its docstring and field descriptions **tell the model** what to write. Its types and **validators** (`@field_validator`, plain Python) **check** what it wrote and raise a `ValueError` with a precise message. The same idea works in LangChain ([[Schemas in LangChain]]).

## `ask_for_answer`: the form-filling loop

```python
# agents.py (shortened)
model = llm.bind_tools([schema], tool_choice="required")       # 1. the form, as a tool it must call
for attempt in range(AGENT_MAX_CORRECTIONS + 1):
    reply = model.invoke(messages)                              # 2. the model fills it in
    try:
        if not reply.tool_calls:                                #    no call, or broken JSON
            raise ValueError(f"call {schema.__name__} with your answer")
        return schema.model_validate(reply.tool_calls[0]["args"])   # 3. check it
    except ValueError as error:
        messages.append(correction(error, attempt))             # 4. say what was wrong, ask again
raise ValueError(f"{agent} gave no valid {schema.__name__}")    # out of attempts: the date fails
```

1. The schema is given to the model **as a tool** ([[Tools in LangGraph#A tool that runs nothing: a form|a form]]), and `tool_choice="required"` forces it to call it.
2. The model "calls" the form. Its arguments are the answer.
3. `model_validate` runs every type check and validator.
4. On failure, a message with the error is appended, and the model answers again, up to `AGENT_MAX_CORRECTIONS` (4) times.

The retries are a `for` loop **inside one node**. They don't add graph steps and aren't stored in the state.

## Watch out
- **Error messages are prompts.** The model fixes only what the message tells it: "Missing: ['NDX']" works, "invalid" doesn't.
- **Keep schemas simple** (flat dicts, enums). The model servers are strict about tool-call formats.
- **Valid isn't good.** A schema checks shape and hard rules, not whether the signals make sense.

## Related
[[Agents in LangGraph]] · [[Tools in LangGraph]] · [[Errors and retries in LangGraph]] · [[Schemas in LangChain]]

## My notes & questions
