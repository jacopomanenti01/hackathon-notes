---
tags: [note]
topic: langchain
template_files: [src/systems/langchain/schema.py]
status: current
created: 2026-10-30
updated: 2026-10-30
---
# Schemas in LangChain

> A schema describes the shape of some data. The model reads it to know what to write, and your code uses it to check what the model wrote.

## Context

At step 6 of the [[Agents in LangChain#Context|running example]], the model "writes its answer: a signal per asset and a short reason". But a model writes text, and text can be anything:

```text
"SPX looks very strong, NDX is fine too, I'd avoid rates."
```

That answer has three problems. "Very strong" isn't one of high, neutral or low. It doesn't give a signal for every asset. And the next agent can't read it reliably. What we need is something like:

```python
Analysis(signals={"SPX": "high", "NDX": "neutral", "EUROSTR=": "neutral", ...},
         rationale="SPX ranks top third in both the 90- and 365-day windows ...")
```

A **schema** is how you get that.

## What a schema does

A schema is a Python class (Pydantic `BaseModel`, or a `dataclass`) that declares the fields, their types and their allowed values. It does **two jobs at once**:

1. **It instructs the model.** The class docstring and each field's `description` are sent to the model, so they're part of the prompt.
2. **It checks the answer.** Pydantic rejects data of the wrong shape, and a **validator** (`@field_validator`) can enforce any rule you write in Python. When a check fails, the error message goes back to the model, which tries again.

```python
class Analysis(BaseModel):
    """One high, neutral, or low signal per asset and a concise rationale."""   # the model reads this
    signals: dict[str, Literal["high", "neutral", "low"]] = Field(description="One signal for every asset")
    rationale: str = Field(description="Concise rationale")

    @field_validator("signals")
    @classmethod
    def _every_asset(cls, signals):
        validate_asset_names(signals)   # raises "Missing: [...]; unexpected: [...]" -> back to the model
        return signals
```

With this schema, the answer above fails twice: "very strong" isn't allowed by `Literal`, and the missing assets fail `_every_asset`. The model gets both errors and answers again.

## The three schemas in the example

| Schema | In the example | Who fills it in |
|---|---|---|
| `StatisticsInput` | `lookback_days=90` in the tool call (5 to 3650) | the model, when calling a tool |
| `Analysis` | the final answer, step 6 | the model, as its answer (`ToolStrategy`) |
| `AgentContext` | the date 2025-01-31 | your code; the model never sees it |

`response_format=ToolStrategy(Analysis)` **forces the format** of the agent's final answer: the schema is offered to the model as a tool it must call, so it can only answer by filling in the fields. The LangGraph system does the same by hand with `ask_for_answer` ([[Schemas in LangGraph#Forcing the format, like in LangChain|forcing the format, like in LangChain]]).

## Watch out
- **Error messages are prompts.** Write them as instructions the model can act on ("Missing: ['NDX']"), not just "invalid".
- **A valid answer isn't a good answer.** A schema checks shape and hard rules. It can't tell whether the signals are sensible.

## Related
[[Agents in LangChain]] · [[What is a tool]] · [[Tools in LangChain]] · [[State in LangChain]]


