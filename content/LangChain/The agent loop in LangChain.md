---
tags: [note]
topic: langchain
sources:
  - https://docs.langchain.com/oss/python/langchain/agents
  - https://docs.langchain.com/oss/python/langchain/messages
status: current
created: 2026-10-30
updated: 2026-11-03
---
# The agent loop in LangChain

> Inside `create_agent` there are two steps, **model** and **tools**, repeated until the model stops asking for tools. One list, `messages`, carries the whole conversation from one turn to the next.

## What it is

The **agent loop** is the cycle that makes an agent work: the model decides, your program runs the tools it asked for, the model reads the results, and so on until it answers ([[Introduction#What is a loop|What is a loop]]). Each round is a **turn**: one model call plus the tools it requested ([[Introduction#What is a turn|What is a turn]]). This note is about how that loop looks *inside* LangChain's `create_agent`.

## Its components

| Component | What it does |
|---|---|
| **model step** | sends all the messages so far to the model, with the tool descriptions, and adds its reply |
| **tools step** | runs every tool call in the reply, and adds each result |
| **the `messages` list** | the conversation, resent in full on every turn: it's the model's only memory |
| **the stop condition** | a reply without tool calls (or a valid answer schema) ends the loop |

The messages in the list have types:

| Message | Who writes it | Holds |
|---|---|---|
| `SystemMessage` | your code | the instructions |
| `HumanMessage` | your code | the request |
| `AIMessage` | the model | its reply: text, or `tool_calls` (name, arguments, id) |
| `ToolMessage` | the tools step | a tool's result, tagged with the call's `tool_call_id` |

## How LangChain implements it

`create_agent` builds a small graph with a `model` node and a `tools` node, and an edge back from tools to model:

```mermaid
flowchart LR
    S([START]) --> M["model<br/>reads all messages"]
    M -->|reply has tool calls| T["tools<br/>runs them, adds results"]
    T --> M
    M -->|no tool calls| E([END])
```

Written by hand, it's roughly this *(simplified)*:

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

This version runs as is only for tools **without** a hidden `runtime` argument (calling one by hand fails with *"runtime: Field required"*). On top of it, `create_agent` fills in the runtime ([[Runtime in LangChain]]), sends argument errors back to the model ([[Tools in LangChain]]), checks the final answer against a schema ([[Schemas in LangChain]]) and runs your middleware at each [[Hooks in LangChain|hook]].

**How the loop ends:**

| What happens | Result |
|---|---|
| the model replies **without tool calls** | the text is the answer |
| the model fills in the **answer schema** and it passes the checks | `structured_response` is set |
| a **middleware** stops it, e.g. a model call limit | the agent ends, possibly without an answer |
| the **recursion limit** is hit | an error: a backstop for runaway loops ([[Limits in LangChain]]) |

An invalid answer does **not** end the loop: the error goes back as a message and the model answers again.

## Example

Take a researcher agent asked to *"Research the funds"* (equity, bond and cash), with two tools: `get_returns(days)`, for at most 365 days, and `get_volatility(fund)`. Its run takes four [[Introduction#What is a turn|turns]], and the `messages` list grows every time:

| Turn | model step: the model reads all `messages` and replies | tools step: run the calls, add the results |
|---|---|---|
| 1 | "call `get_returns(days=500)`" | rejected: the error message is added |
| 2 | "call `get_returns(days=90)`" | the returns are added |
| 3 | "call `get_volatility("equity")` and `get_volatility("bond")`" | both results are added |
| 4 | the notes, with no tool call | nothing to run: the loop ends |

At turn 4 the model can compare the returns from turn 2 with the volatility from turn 3, because it receives the whole list on every call.

## Watch out
- **The list grows every turn**, and every model call resends all of it. Many turns with big tool results get slow and costly.
- **From outside, you only see the final state.** To see each turn, look at the full `messages`, or a trace of the run.

## References
- [LangChain docs: Agents](https://docs.langchain.com/oss/python/langchain/agents)
- [LangChain docs: Messages](https://docs.langchain.com/oss/python/langchain/messages)

## Related
[[A Brief Overview on LangChain]] · [[Introduction]] · [[Agents in LangChain]] · [[Tools in LangChain]] · [[Hooks in LangChain]] · [[Middleware in LangChain]]
