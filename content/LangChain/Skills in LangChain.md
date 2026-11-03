---
tags: [note]
topic: langchain
sources:
  - https://docs.langchain.com/oss/python/langchain/multi-agent/skills
  - https://support.claude.com/en/articles/12512176-what-are-skills
  - https://docs.langchain.com/oss/python/langchain/context-engineering
status: current
created: 2026-10-30
updated: 2026-11-03
---
# Skills in LangChain

> A skill is a method written down in its own file: instructions for one kind of task, that an agent reads when it needs them.

## What it is

Tools say what an agent *can* do; they don't say *how* to do it well. A **skill** is that "how": a short, written method for one kind of task. The agent's role says *who* it is; each skill says *how* to do one thing. Keeping skills in their own files means one method lives in one place, several agents can share it, and you can add or remove one without touching the prompt.

## Its components

A skill is a folder with a `SKILL.md` file, made of:

| Component | What it is |
|---|---|
| `name` | a short identifier |
| `description` | one line: what the skill is for, so the agent can tell when it applies |
| **instructions** | the method itself, in plain text |

## How LangChain implements it

LangChain has no special feature for skills: a skill is just text, and there are two ways to deliver it to an agent.

|                      | In the system prompt                                                                       | Loaded on demand                                                                                           |
| -------------------- | ------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| **How**              | your code reads the files and appends them to `system_prompt`                              | the agent sees only each skill's name and description, and calls a `load_skill` tool to read the full text |
| **The model has it** | from the first call                                                                        | only after it asks for it                                                                                  |
| **Cost**             | turn]]<br>every skill is resent on every [[quartz/content/What is a turn\|What is a turn]] | only the skills it uses                                                                                    |
| **Good for**         | a few short skills the agent always needs                                                  | many skills, or long ones                                                                                  |

Loading on demand is called **progressive disclosure**: show a short list, load the details only when relevant. A `load_skill` tool is an ordinary tool ([[Tools in LangChain]]):

```python
from pathlib import Path
from langchain.tools import tool

@tool
def load_skill(name: str) -> str:
    """Load a skill: step-by-step instructions for one kind of task.

    Available skills:
    - return-and-risk: how to compare funds by return and risk.
    """
    return Path(f"skills/{name}/SKILL.md").read_text()
```

## Example

Take a researcher agent with two tools, `get_returns` and `get_volatility`, asked to compare three funds (equity, bond and cash). Its role only says *"You are a market researcher. Measure the funds' returns and risk."* The method comes from a skill, `return-and-risk`:

```markdown
---
name: return-and-risk
description: How to compare funds by return and risk.
---
1. Call get_returns once, for all funds, over 90 days.
2. Call get_volatility only for the funds whose price moves (cash barely does).
3. In your notes, give each fund its return, its volatility, and which looks best for its risk.
```

```python
researcher = create_agent(model, tools=[get_returns, get_volatility, load_skill],
                          system_prompt="You are a market researcher. Use load_skill when a skill fits the task.")
```

Following it, the researcher measures the returns of all funds first, then checks the volatility of equity and bond, but not of cash.

## Watch out
- **A skill is advice, not a rule.** The model may follow it loosely. A rule that must always hold belongs in code: a validator ([[Schemas in LangChain]]) or middleware ([[Middleware in LangChain]]).
- **Skills mustn't contradict each other or the role.** They end up in the same conversation, and the model can't tell which one wins.
- **Describe each skill well.** With loading on demand, the description is all the model sees when deciding whether to load it.

## Related
[[LangChain Overview]] · [[Agents in LangChain]] · [[Tools in LangChain]] · [[Middleware in LangChain]] · [[What is a turn]]

## My notes & questions
