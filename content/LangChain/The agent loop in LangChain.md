---
tags: [note]
topic: langchain
sources:
  - https://docs.langchain.com/oss/python/langchain/agents
  - https://docs.langchain.com/oss/python/langchain/messages
status: current
created: 2026-10-30
updated: 2026-11-02
---
# The agent loop in LangChain

> Inside `create_agent` there are two steps, **model** and **tools**, repeated until the model stops asking for tools. One list, `messages`, carries the whole conversation from one turn to the next. (The idea of the loop, independent of any library: [[What is a loop]].)

## Context

In [[The running example]], the researcher runs for 4 [[What is a turn|turns]]. Seen from inside the agent, each turn is the same two steps, and the `messages` list grows every time:

| Turn | model step: the model reads all `messages` and replies | tools step: run the calls, add the results |
|---|---|---|
| 1 | "call `get_returns(days=500)`" | rejected: the error message is added |
| 2 | "call `get_returns(days=90)`" | the returns are added |
| 3 | "call `get_volatility("equity")` and `get_volatility("bond")`" | both results are added |
| 4 | the notes, with no tool call | nothing to run: the loop ends |

At turn 4 the model can compare the returns (turn 2) with the volatility (turn 3) because it receives **the whole list** on every call. The model remembers nothing between calls: the list is its memory.

## The two steps

1. **model**: send all of `messages` to the model, with the tool descriptions. Add its reply to the list.
2. **tools**: if the reply contains tool calls, run each one, and add its result as a `ToolMessage` (the message type for a tool's result). Go back to step 1.

```mermaid
flowchart LR
    S([START]) --> M["model<br/>reads all messages"]
    M -->|reply has tool calls| T["tools<br/>runs them, adds results"]
    T --> M
    M -->|no tool calls| E([END])
```

## Example

The same loop written by hand *(simplified: roughly what `create_agent` does for you)*:

```python
from langchain.messages import HumanMessage, SystemMessage, ToolMessage

tools = {t.name: t for t in [get_returns, get_volatility]}
model_with_tools = model.bind_tools(list(tools.values()))     # the model gets the tool descriptions

messages = [SystemMessage("You are a market researcher. ..."), HumanMessage("Research the funds.")]
for turn in range(8):                                # a limit, so the loop always ends
    reply = model_with_tools.invoke(messages)        # 1. model step: send ALL the messages
    messages.append(reply)
    if not reply.tool_calls:                         # no tool calls: this reply is the answer
        break
    for call in reply.tool_calls:                    # 2. tools step: e.g. {"name": "get_volatility",
        result = tools[call["name"]].invoke(call["args"])   #     "args": {"fund": "bond"}, "id": "call_4"}
        messages.append(ToolMessage(content=str(result), tool_call_id=call["id"]))
```

This hand-written version runs as is only for tools **without** a `runtime` argument: calling `get_returns` by hand fails with *"runtime: Field required"*.

What `create_agent` adds on top: it fills in the hidden `runtime` (the mandate: [[Runtime in LangChain]]), checks arguments and sends errors back (turn 1), checks the final answer against a schema ([[Schemas in LangChain]]), and runs your middleware at each [[Hooks in LangChain|hook]].

## How the loop ends

| What happens | In the example |
|---|---|
| the model replies **without tool calls**: the text is the answer | the researcher's turn 4, the writer's only turn |
| the model fills in the **answer schema** (`response_format`), and it passes the checks | the strategist's turn 2 |
| a **middleware** stops it, e.g. a call limit | after too many model calls |
| the **recursion limit** is hit: an error | a backstop for runaway loops |

An **invalid answer doesn't end the loop**: the strategist's first answer is rejected, the error goes back as a message, and the strategist simply answers again.

## Watch out
- **The list grows every turn**, and every model call resends all of it. Many turns with big tool results get slow and costly.
- **From outside, you only see the final state.** To see each turn, look at the full `messages`, or a trace of the run.

## Related
[[LangChain Overview]] · [[The running example]] · [[What is a loop]] · [[What is a turn]] · [[What is a tool]] · [[Agents in LangChain]] · [[Tools in LangChain]] · [[Hooks in LangChain]] · [[Middleware in LangChain]]

## My notes & questions
