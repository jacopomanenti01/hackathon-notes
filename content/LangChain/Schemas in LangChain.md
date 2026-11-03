---
tags: [note]
topic: langchain
sources:
  - https://docs.langchain.com/oss/python/langchain/structured-output
  - https://docs.langchain.com/oss/python/langchain/tools
status: current
created: 2026-10-30
updated: 2026-11-02
---
# Schemas in LangChain

> A schema describes the shape of some data: its fields, their types and their rules. The model reads it to know what to write, and your code uses it to check what the model wrote.

## Context

In [[The running example]], the strategist must return the portfolio's weights. A model writes text, and text can be anything:

```text
"I'd go mostly equity, maybe 60%, and put the rest in bonds."
```

That's hard for the next step to use: "maybe 60%" isn't a number, the Cash Fund is missing, and the risk check can't read a sentence. It needs something like:

```python
Allocation(weights={"equity": 0.55, "bond": 0.30, "cash": 0.15},
           rationale="Equity has the best return for its risk.")
```

A **schema** is how you get that.

## What a schema does

A schema is a Python class, usually a Pydantic `BaseModel`, that declares the fields, their types and their allowed values. It does **two jobs at once**:

1. **It instructs the model.** The class docstring and each field's `description` are sent to the model: they're part of the prompt.
2. **It checks the answer.** Pydantic rejects data of the wrong shape, and a **validator** (`@field_validator`) enforces any rule you write in Python. When a check fails, the error goes back to the model, which answers again.

## Example

```python
from typing import Literal
from pydantic import BaseModel, Field, field_validator

class Allocation(BaseModel):
    """How much of the portfolio goes into each fund, and why."""        # the model reads this
    weights: dict[Literal["equity", "bond", "cash"], float] = Field(
        description="A weight between 0 and 1 for every fund")
    rationale: str

    @field_validator("weights")
    @classmethod
    def _every_fund(cls, weights):
        missing = sorted({"equity", "bond", "cash"} - set(weights))
        if missing:
            raise ValueError(f"Every fund needs a weight; missing: {missing}")
        if any(w < 0 for w in weights.values()):
            raise ValueError("Weights can't be negative")
        return weights
```

```python
from langchain.agents.structured_output import ToolStrategy

strategist = create_agent(model, response_format=ToolStrategy(Allocation), ...)
result = strategist.invoke(...)
allocation = result["structured_response"]       # an Allocation, already checked
```

In the example, the strategist's first answer gives equity and bond only. The validator raises *"Every fund needs a weight; missing: ['cash']"*, LangChain sends that message back, and the second answer (55/30/15) passes.

## Where schemas are used

| Schema | Passed to | Who fills it in | Seen by the model? |
|---|---|---|---|
| `ReturnsInput` | `@tool(args_schema=...)` | the model, when calling a tool | yes |
| `Allocation` | `response_format=ToolStrategy(...)` | the model, as its final answer | yes |
| `Mandate` | `context_schema=` | your code ([[Runtime in LangChain]]) | **no** |

`ToolStrategy` gives the model the schema **as a tool to call** for its final answer: a tool's arguments are already structured data. Unlike a real tool, nothing runs afterwards: the arguments *are* the answer. Retrying on a failed check is on by default (`handle_errors=True`). Passing the schema directly, `response_format=Allocation`, lets LangChain choose between this and the model provider's own structured output.

## Watch out
- **Error messages are prompts.** Write them as instructions the model can act on ("missing: ['cash']"), not just "invalid".
- **Valid isn't good.** A schema checks shape and hard rules. It can't tell whether the weights are a sensible portfolio.
- **Keep schemas simple**: flat fields, dicts, fixed values (`Literal`). Deeply nested schemas are harder for models to fill in.

## Related
[[LangChain Overview]] · [[The running example]] · [[Agents in LangChain]] · [[Tools in LangChain]] · [[What is a tool]] · [[State in LangChain]]

## My notes & questions
