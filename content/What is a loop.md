---
tags: [note]
topic: general
sources:
  - https://huggingface.co/learn/agents-course/unit1/tools
  - https://docs.langchain.com/oss/python/langchain/agents
status: current
created: 2026-11-02
updated: 2026-11-02
---
# What is a loop

> The agent loop is the cycle *model decides → program runs the tools → model reads the results*, repeated until the model answers without asking for a tool. It's what turns a single model call into an agent.

## Context

In [[The running example]], the researcher can't decide which funds' risk to check before it has seen their returns, and it can't fix its bad `days=500` call before it sees the error. One model call can't do that: the model would have to guess everything at once. The researcher needs to **ask, read the answer, and decide again**, four times. That repetition is the **loop**.

## How it works

Each round of the loop is a [[What is a turn|turn]]:

1. **The model decides.** It reads everything so far (the instructions, the request, earlier calls and their results) and replies with either tool calls or a final answer.
2. **If it asked for tools,** the program runs them ([[What is a tool]]) and adds each result to the conversation.
3. **Back to step 1**, with the longer conversation.
4. **If it answered without tool calls,** the loop ends: that reply is the answer.

```mermaid
flowchart LR
    S([request]) --> M["model decides"]
    M -->|tool calls| T["program runs the tools"]
    T -->|results added| M
    M -->|no tool calls| E([answer])
```

In the example: turn 1 asks for `days=500` and reads the error; turn 2 asks for `days=90`; turn 3 asks for the volatility of equity and bond; turn 4 writes the notes, with no tool call, so the loop ends.

## What every loop needs

| Need | Why | In the example |
|---|---|---|
| **memory** | the model remembers nothing between calls: the conversation is resent every turn | the returns from turn 2 are still there at turn 4 |
| **a way to recover** | a bad call shouldn't end the run: the error goes back as a result | turn 1's error, fixed at turn 2 |
| **a stop condition** | the model decides when it's done | turn 4: no tool calls |
| **a limit** | a model may never decide it's done | at most 8 model calls, 6 tool calls |

## Who writes the loop

The idea is the same everywhere; only who writes it changes ([[LangChain or LangGraph]]):
- in **LangChain**, `create_agent` runs it for you, and you add code at fixed moments ([[The agent loop in LangChain]]);
- in **LangGraph**, you draw it yourself: a model node, a tools node, and an edge back ([[The agent loop in LangGraph]]).

## Watch out
- **Every turn costs more than the last.** The whole conversation is resent each time, so long loops with big tool results get slow and expensive.
- **A loop without a limit can run forever**, or until something crashes.

## Related
[[LangChain Overview]] · [[The running example]] · [[What is a turn]] · [[What is a tool]] · [[The agent loop in LangChain]] · [[LangChain or LangGraph]]

## My notes & questions
