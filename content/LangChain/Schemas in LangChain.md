---
tags: [note]
topic: langchain
sources:
  - https://docs.langchain.com/oss/python/langchain/structured-output
  - https://docs.langchain.com/oss/python/langchain/tools
status: current
created: 2026-10-30
updated: 2026-11-03
---
# Schemas in LangChain

> A schema describes the shape of some data: its fields, their types and their rules. The model reads it to know what to write, and your code uses it to check what the model wrote.

## What it is

A model writes free text, and code can't rely on free text. A **schema** fixes the shape of the data instead. It does **two jobs at once**:
1. **It instructs the model**: its descriptions are sent to the model, as part of the prompt.
2. **It checks the result**: data of the wrong shape, or that breaks a rule, is rejected, and the error goes back to the model so it can try again.

## Its components

| Component | What it is |
|---|---|
| **fields** | the pieces of data, e.g. a weight per fund and a rationale |
| **types** | what each field holds: a number, text, a list, a fixed set of values |
| **descriptions** | text the model reads: the class docstring, and each field's description |
| **validators** | your own rules, written in Python, e.g. "every fund needs a weight" |

## How LangChain implements it

A schema is usually a Pydantic `BaseModel`: fields with type hints, `Field(description=..., ge=..., le=...)` for descriptions and limits, and `@field_validator` for your own rules. It's used in three places:

| Passed to | Describes | Filled in by | Seen by the model? |
|---|---|---|---|
| `@tool(args_schema=...)` | a tool's arguments ([[Tools in LangChain]]) | the model, when calling the tool | yes |
| `create_agent(response_format=...)` | the agent's final answer | the model, as its answer | yes |
| `create_agent(context_schema=...)` | the fixed context ([[Runtime in LangChain]]) | your code | no |

For the final answer, `response_format` takes:
- `ToolStrategy(Schema)`: the schema is given to the model **as a tool to call** for its answer. Nothing runs afterwards: the arguments *are* the answer. A failed check goes back to the model (`handle_errors=True`, the default);
- `ProviderStrategy(Schema)`: the model provider's own structured output;
- the schema itself: LangChain picks one of the two.

The checked answer is in the result under `structured_response`.

## Example

Take a strategist agent that must return a portfolio's **weights**, the share of the money in each of three funds (equity, bond and cash). Left to itself, a model might write *"mostly equity, maybe 60%, the rest in bonds"*: no number for cash, and nothing code can check. With a schema:

```python
from typing import Literal
from pydantic import BaseModel, Field, field_validator
from langchain.agents.structured_output import ToolStrategy

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

strategist = create_agent(model, response_format=ToolStrategy(Allocation), ...)
allocation = strategist.invoke(...)["structured_response"]   # an Allocation, already checked
```

The strategist's first answer gives weights for equity and bond only. The validator raises *"Every fund needs a weight; missing: ['cash']"*, LangChain sends it back, and on its second [[Introduction#What is a turn|turn]] the strategist answers equity 55%, bond 30%, cash 15%, which passes.

## Watch out
- **Error messages are prompts.** Write them as instructions the model can act on ("missing: ['cash']"), not just "invalid".
- **Valid isn't good.** A schema checks shape and hard rules. It can't tell whether the weights are a sensible portfolio, and 55% in equity passes even if the mandate allows 40% ([[Middleware in LangChain]]).
- **Keep schemas simple**: flat fields, dicts, fixed values (`Literal`). Deeply nested schemas are harder for models to fill in.

## References
- [LangChain docs: Structured output](https://docs.langchain.com/oss/python/langchain/structured-output)
- [LangChain docs: Tools](https://docs.langchain.com/oss/python/langchain/tools)

## Related
[[A Brief Overview on LangChain]] · [[Agents in LangChain]] · [[Tools in LangChain]] · [[Introduction]] · [[State in LangChain]]
