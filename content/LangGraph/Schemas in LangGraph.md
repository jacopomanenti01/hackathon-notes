---
tags: [note]
topic: langgraph
template_files: [src/systems/langgraph/schema.py, src/systems/langgraph/agents.py]
status: current
created: 2026-10-30
updated: 2026-10-30
---
# Schemas in LangGraph

> Binding a Pydantic schema as a tool the model *must* call **forces the format of its answer**: no free text, only the schema's fields. `ask_for_answer` does this, checks the answer, and sends back the errors until it's valid. It's the hand-written version of LangChain's `ToolStrategy`.

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

## Forcing the format, like in LangChain

Steps 1–2 are what force the output format. The model gets one "tool", `Analysis`, and `tool_choice="required"` means it must call it. So its reply can only be `Analysis(signals=..., rationale=...)`, never a paragraph of text.

The LangChain system does exactly the same thing, with one line instead of a loop ([[Schemas in LangChain]]):

| | LangGraph: `ask_for_answer` | LangChain: `response_format` |
|---|---|---|
| how you ask for the format | `ask_for_answer(llm, "analyst_signals", prompt, Analysis)` | `create_agent(..., response_format=ToolStrategy(Analysis, handle_errors=True))` |
| how the format is forced | the schema is bound as a tool the model must call | the same: `ToolStrategy` offers the schema as a tool |
| invalid answer | the error goes back, up to `AGENT_MAX_CORRECTIONS` times | the error goes back (`handle_errors=True`) |
| where you get the result | the return value: an `Analysis` | `result["structured_response"]`: an `Analysis` |
| who writes the loop | you (`agents.py`, ~15 lines) | LangChain |

Same schema, same mechanism, same result. In LangGraph you can read every step.

## Same bind_tools, two uses

A real tool and a structured answer go through the same mechanism. What you bind is the difference:

| | A real tool | A structured answer |
|---|---|---|
| helper | `ask_with_tools` | `ask_for_answer` |
| what you bind | a function: `bind_tools([get_asset_statistics])` | a schema with no function behind it: `bind_tools([Analysis], tool_choice="required")` |
| what the model does | *asks* for the tool, when it decides to | *must* call it, and fills in the fields |
| what runs it | a `ToolNode` in the graph | nothing: you read `reply.tool_calls[0]["args"]` |
| what it fixes | nothing: the model may reply in text or call the tool | the **format** of the answer |
| who uses it | `analyst` | `analyst_signals`, `portfolio_manager` |

So a real tool needs **both** the schema and the node: the schema lets the model ask, and the node runs it ([[Tools in LangGraph]]). A structured answer needs only the schema, because the model's arguments *are* the answer.

## Why not use the state's types?

`PortfolioState` already declares `signals: dict[str, str]` ([[State in LangGraph]]). Couldn't that shape the model's answer? No, for two reasons:
- **The model never sees it.** The state describes what *nodes* pass to each other. It's never sent to the LLM, so it can't tell the model what to write.
- **Nothing checks it at run time.** A `TypedDict` is only a type hint. If a node returns `signals = "SPX looks strong"`, LangGraph stores it without complaint.

| | `PortfolioState` (the state's types) | `Analysis` (the schema bound to the model) |
|---|---|---|
| who reads it | your nodes | the model |
| checked at run time? | no | yes: types and validators |
| what it's for | the hand-off between nodes | the model's answer |

So the node still has to get a well-formed answer *from the model*, then copy it into the state. The schema-as-tool does the first part.

## Who uses ask_for_answer, and why the analyst is split

The analyst's work is two nodes:
- **`analyst`** researches. It calls `ask_with_tools` with the *real* tool, loops with the `ToolNode`, and writes its notes in plain text, with no forced format ([[The agent loop in LangGraph]]).
- **`analyst_signals`** fills in the form. It calls `ask_for_answer(..., Analysis)` on those notes and returns checked `signals`.

**Could it be one node?** Yes: bind both, `llm.bind_tools([get_asset_statistics, Analysis])`, and treat an `Analysis` call as "done". That's essentially what `ToolStrategy` does inside LangChain's `create_agent` ([[Schemas in LangChain]]). The template keeps the two separate so each step stays simple: free research first, then one strictly checked answer.

**And `with_structured_output`?** LangChain's `llm.with_structured_output(Analysis)` also returns an `Analysis` object. But when the answer fails validation it raises an error, instead of sending the error back to the model. That's why the template writes its own small loop.

## Watch out
- **Error messages are prompts.** The model fixes only what the message tells it: "Missing: ['NDX']" works, "invalid" doesn't.
- **Keep schemas simple** (flat dicts, enums). The model servers are strict about tool-call formats.
- **Valid isn't good.** A schema checks shape and hard rules, not whether the signals make sense.

## Related
[[Agents in LangGraph]] · [[Tools in LangGraph]] · [[Errors and retries in LangGraph]] · [[Schemas in LangChain]]


