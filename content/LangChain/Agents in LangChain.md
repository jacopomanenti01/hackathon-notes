---
tags: [note]
topic: langchain
sources:
  - https://docs.langchain.com/oss/python/langchain/quickstart
  - https://docs.langchain.com/oss/python/langchain/agents
  - https://docs.langchain.com/oss/python/langchain/runtime
status: current
created: 2026-10-30
updated: 2026-11-03
---
# Agents in LangChain

> An agent is a model that can use tools in a loop: it asks for data, reads it, and decides again, until it has an answer. `create_agent` builds one for you.

## What it is

A plain model call sends a request to a large language model (LLM) and gets one reply back, written from what the model already knows. An **agent** can do more: when it needs information it doesn't have, it asks your program to run a function ([[Introduction#What is a tool|What is a tool]]), reads the result, and decides what to do next. It repeats this ask-read-decide cycle, the **loop** ([[Introduction#What is a loop|What is a loop]]), until it can answer.

So an agent is a model, plus tools, plus that loop.

## Its components

| Component | What it is |
|---|---|
| **model** | the LLM that makes every decision |
| **instructions** | the system prompt: who the agent is and what it should do |
| **tools** | the functions the model may ask for ([[Tools in LangChain]]) |
| **answer form** (optional) | the exact shape the final answer must have, checked by code ([[Schemas in LangChain]]) |
| **context** (optional) | data your code passes in, which the model never sees ([[Runtime in LangChain]]) |
| **middleware** (optional) | your own code, run at fixed moments of the loop ([[Middleware in LangChain]]) |

	
## How LangChain implements it

You don't write the loop: you describe the agent with `create_agent`, and LangChain builds it.

| Parameter of `create_agent` | Component |
|---|---|
| `model` | the model (the only required part) |
| `system_prompt` | the instructions |
| `tools` | the tools |
| `response_format` | the answer form |
| `context_schema` | the shape of the context |
| `middleware` | the middleware, as a list |
| `state_schema` | extra keys in the agent's state ([[State in LangChain]]) |
| `checkpointer` | memory across several runs (a conversation) |

You run it with `agent.invoke`, giving the first **message** (the request, as if typed by a user) and the context. When the loop ends, `invoke` returns the agent's final state, a dict with two keys:
- `messages`: the whole conversation (the request, every tool call, every result, the final reply);
- `structured_response`: the checked answer, present only if there's a `response_format` ([[Schemas in LangChain]]).

**Without a schema**, the result has only `messages`, and the answer is the text of the last one:

```python
result = researcher.invoke(...)
result.keys()                  # dict_keys(['messages'])
[type(m).__name__ for m in result["messages"]]
# ['HumanMessage',                              the request
#  'AIMessage', 'ToolMessage',                  turn 1: a tool call and its result (an error)
#  'AIMessage', 'ToolMessage',                  turn 2
#  'AIMessage', 'ToolMessage', 'ToolMessage',   turn 3: two tool calls, two results
#  'AIMessage']                                 turn 4: the answer, in text
result["messages"][-1].content
# 'Equity: +4% over 90 days, volatility 15%. Bond: +1%, volatility 5%. Cash: +0.5%, almost no risk.'
```

**With a schema** (`response_format=ToolStrategy(Allocation)`), the result also has `structured_response`: the answer as an object of your schema, already checked. The last message is only LangChain's confirmation, so read the answer from `structured_response`:

```python
result = strategist.invoke(...)
result.keys()                  # dict_keys(['messages', 'structured_response'])
result["structured_response"]
# Allocation(weights={'equity': 0.55, 'bond': 0.3, 'cash': 0.15},
#            rationale='Equity has the best return for its risk.')
result["structured_response"].weights["equity"]      # 0.55: a number your code can use
```

## Example

Take a **researcher** agent that must report how three funds (equity, bond and cash) have performed and how risky they are. Its request is *"Research the funds."*

With a plain model call, the model has no access to the funds' prices, so it can only guess. As an agent, with two tools, one for the returns and one for the volatility, it can find out:

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

The run takes four [[Introduction#What is a turn|turns]]: the model asks for the returns over 500 days and is told the limit is 365, asks again for 90 days, asks for the volatility of equity and bond, and finally writes its notes. Each step of that cycle is explained in [[The agent loop in LangChain]].

An agent doesn't need tools. A **writer** that only turns approved weights into a short market note has no tools and no answer form: it answers once, in text. That's still an agent, whose loop lasts one turn.

## Watch out
- **Prompt or context?** What the model must *read* goes in the prompt or the messages. What it must *not change* (the funds, the limit) goes in the context; middleware can still show it to the model ([[Runtime in LangChain]]).
- **No answer is possible.** If a limit stops the agent, `structured_response` is `None`. Check it before using it.

## References
- [LangChain docs: Quickstart](https://docs.langchain.com/oss/python/langchain/quickstart)
- [LangChain docs: Agents](https://docs.langchain.com/oss/python/langchain/agents)
- [LangChain docs: Runtime](https://docs.langchain.com/oss/python/langchain/runtime)

## Related
[[A Brief Overview on LangChain]] · [[Introduction]] · [[The agent loop in LangChain]] · [[Runtime in LangChain]] · [[Tools in LangChain]] · [[Schemas in LangChain]] · [[Middleware in LangChain]] · [[State in LangChain]] · [[Workflow in LangChain]]
