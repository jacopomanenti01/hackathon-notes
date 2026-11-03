---
tags: [note]
topic: langgraph
sources:
  - https://docs.langchain.com/oss/python/langchain/models
  - https://docs.langchain.com/oss/python/langgraph/workflows-agents
status: current
created: 2026-10-30
updated: 2026-11-03
---
# Schemas in LangGraph

> A schema is a description of the exact form an answer must take. Bound to the model as a tool it *must* call, it **forces** the form; checked by your node, it **rejects** a wrong answer and asks again.

## What it is

When the next step is code, not a person, a free-text answer won't do: code can't reliably read *"mostly equity, some bonds"*. A **schema** fixes the form of the answer. It does two jobs: it **tells the model** what to write, and it **checks** what the model wrote. LangChain does both with one parameter ([[Schemas in LangChain]]); in LangGraph you write the few lines yourself, inside a node.

## Its components

- **The schema**: the fields, their types, and their descriptions, which the model reads.
- **Validators**: plain Python rules that check the content and raise an error with a precise message.
- **Forcing the form**: the model is made to answer by filling in the schema, never in free text.
- **The correction loop**: a failed check goes back to the model, which answers again, up to a maximum number of attempts.

## How LangGraph implements it

- **The schema**: a Pydantic class. Its docstring and `Field(description=...)` tell the model what to write; its types and `@field_validator` methods check it.
- **Forcing the form**: `model.bind_tools([Allocation], tool_choice="any")`. The schema is offered as a tool with no function behind it, and the model **must** call a tool, so its reply can only be the schema's fields. The arguments of that call *are* the answer.
- **The check**: `Allocation.model_validate(reply.tool_calls[0]["args"])` runs every type check and validator, and raises a `ValidationError` if one fails.
- **The correction loop**: a `for` loop inside the node. On failure, add the model's reply and a `ToolMessage` with the error, and call the model again. After the last attempt, raise an error.

The loop runs **inside one node**, so its attempts are not graph steps and aren't stored in the state. The shortcut `model.with_structured_output(Allocation)` also returns an `Allocation`, but when the check fails it raises the error instead of sending it back to the model.

## Example

Take a strategist agent that turns research notes into the weights of three funds (equity, bond and cash). The next step is code that checks equity is at most 40%, so it needs a number for every fund:

```python
from typing import Literal
from pydantic import BaseModel, Field, field_validator

class Allocation(BaseModel):
    """How much of the portfolio goes into each fund, and why."""                   # the model reads this
    weights: dict[Literal["equity", "bond", "cash"], float] = Field(description="A weight between 0 and 1 for every fund")
    rationale: str

    @field_validator("weights")
    @classmethod
    def _every_fund(cls, weights):
        missing = sorted({"equity", "bond", "cash"} - set(weights))
        if missing:
            raise ValueError(f"Every fund needs a weight; missing: {missing}")
        return weights
```

The strategist's node forces the form, checks it, and asks again on an error:

```python
from langchain.messages import HumanMessage, SystemMessage, ToolMessage
from pydantic import ValidationError

def strategist(state: PortfolioState) -> dict:
    form = model.bind_tools([Allocation], tool_choice="any")              # it must fill in the form
    messages = [SystemMessage("You are a portfolio strategist. Turn the research into weights."),
                HumanMessage(state["notes"])]
    for _ in range(3):                                                    # at most 3 attempts
        reply = form.invoke(messages)
        call = reply.tool_calls[0]
        try:
            return {"proposed_weights": Allocation.model_validate(call["args"]).weights}
        except ValidationError as error:                                  # say what was wrong, ask again
            messages += [reply, ToolMessage(f"Error: {error}\nFix your answer and try again.",
                                            tool_call_id=call["id"])]
    raise ValueError("no valid Allocation after 3 attempts")
```

Its first answer gives weights for equity and bond only. The check fails with *"Every fund needs a weight; missing: ['cash']"*, the error goes back, and the second answer, equity 55%, bond 30%, cash 15%, passes. Both attempts happen in one `strategist` step.

## Watch out
- **Error messages are prompts.** The model fixes only what the message tells it: *"missing: ['cash']"* works, *"invalid"* doesn't.
- **Valid isn't good.** A schema checks the form and hard rules, not whether the answer makes sense: 55% in equity passes it, and breaks the mandate ([[Branches in LangGraph]]).
- **The state's types aren't a schema.** The `TypedDict` is never sent to the model and isn't checked at run time ([[State in LangGraph]]).

## Related
[[LangGraph Overview]] · [[Agents in LangGraph]] · [[Tools in LangGraph]] · [[Errors and retries in LangGraph]] · [[Schemas in LangChain]]

## My notes & questions
