---
tags: [note]
topic: langchain
sources:
  - https://docs.langchain.com/oss/python/langchain/quickstart
  - https://docs.langchain.com/oss/python/langchain/agents
  - https://docs.langchain.com/oss/python/langchain/runtime
status: current
created: 2026-10-30
updated: 2026-11-02
---
# Agents in LangChain

> An agent is a model that can use tools in a loop: it asks for data, reads it, and decides again, until it has an answer. `create_agent` builds one for you.

## Context

In [[The running example]], the researcher's job is *"Research the funds."*

**With a plain model call**, you send that sentence to the model and get one reply. The model has no access to the funds' prices, so it can only guess. One call, one answer.

**With an agent**, the same request becomes the researcher's 4 turns: the model asks for the returns, our program runs `get_returns` and sends back the result, the model asks for the volatility, and so on, until it writes its notes. The model never touches the data: it only *asks* for a function, and the program runs it ([[What is a tool]]).

That cycle, *model decides → program runs the tool → model reads the result*, is the **agent loop** ([[What is a loop]]; inside `create_agent`: [[The agent loop in LangChain]]). An **agent** is a model, plus tools, plus that loop.

## The parts of `create_agent`

You don't write the loop. You describe the agent, and LangChain builds it:

| Part | What it means | The researcher's |
|---|---|---|
| `model` | the LLM that makes the decisions | any chat model |
| `system_prompt` | standing instructions, sent before everything else | "You are a market researcher…" plus its skill ([[Skills in LangChain]]) |
| `tools` | the functions the model may ask for | `get_returns`, `get_volatility` ([[Tools in LangChain]]) |
| `response_format` | the exact form of the final answer, checked by code | none: it answers in text (the strategist has one: [[Schemas in LangChain]]) |
| `context_schema` | data your code passes in that the model **never sees** | `Mandate`: the funds and the equity limit ([[Runtime in LangChain]]) |
| `middleware` | your own code, run at fixed moments of the loop | mandate in the prompt, retries, limits ([[Middleware in LangChain]]) |

Only the model is required. The writer has no tools and no `response_format`: it answers once, in text. That's still an agent, whose loop lasts one [[What is a turn|turn]].

## Example

```python
from langchain.agents import create_agent

researcher = create_agent(
    model,
    system_prompt="You are a market researcher. Measure the funds' returns and risk.",
    tools=[get_returns, get_volatility],
    context_schema=Mandate,
    middleware=[mandate_prompt, ...],
)
result = researcher.invoke(
    {"messages": [{"role": "user", "content": "Research the funds."}]},   # the request
    context=Mandate(funds=("equity", "bond", "cash"), max_equity=0.40),
)
notes = result["messages"][-1].content    # "Equity: +4% over 90 days, volatility 15%. ..."
```

`invoke` starts the loop with the first **message** (the request, as if typed by a user) and the **context**. When the loop ends, it returns the agent's final **state**, a dict with:
- `messages`: the whole conversation (request, every tool call, every result, the final reply);
- `structured_response`: the checked answer, only if there's a `response_format`.

## Watch out
- **Prompt or context?** What the model must *read* goes in the prompt or the messages. What it must *not change* (the funds, the limit) goes in the context; middleware can still show it to the model ([[Runtime in LangChain]]).
- **No answer is possible.** If a limit stops the agent, `structured_response` is `None`. Check it before using it.

## Related
[[LangChain Overview]] · [[The running example]] · [[What is a tool]] · [[What is a loop]] · [[The agent loop in LangChain]] · [[Runtime in LangChain]] · [[Tools in LangChain]] · [[Schemas in LangChain]] · [[Middleware in LangChain]] · [[State in LangChain]] · [[Workflow in LangChain]] · [[LangChain or LangGraph]]

## My notes & questions
