---
title: "Beginner's Guide to Using Agentic AI in Python"
seoTitle: "Agentic AI in Python: A Beginner's Guide"
seoDescription: "Learn how to implement a simple Agentic AI setup in Python using LangChain, GPT, and more. Perfect for beginners exploring AI agents"
datePublished: 2025-10-11T13:32:35.294Z
cuid: cmgmbex5q000002jma24m44ga
slug: beginners-guide-to-using-agentic-ai-in-python
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1760189301774/90c3ee7f-db92-4b2c-a409-81b665ec9ab5.jpeg
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1760189507436/c15f6e24-1386-4787-86dc-d86a290508fe.jpeg
tags: python, coding, langchain, aidevelopment, agentic, agenticai

---

Agentic AI is an exciting approach in which AI systems act as independent agents. These agents pursue goals, make decisions, and execute actions independently – often in a complex environment.

In this post, I'll show you how to implement a simple agent setup with Python in just a few steps – ideal for initial experiments with Agentic AI.

---

## Requirements 🔧

Before you start, you should have the following installed:

* Python 3.8+
    
* `openai` (e.g., for GPT agents)
    
* `langchain` (framework for agent workflows)
    
* `dotenv` (for API keys)
    

Installation:

```bash
pip install openai langchain
python-dotenv
```

---

## Simple agent setup with LangChain 🚀

```python
from langchain.agents import initialize_agent,
load_tools
from langchain.llms import OpenAI
import os
from dotenv import load_dotenv

load_dotenv()

llm = OpenAI(temperature=0,
openai_api_key=os.getenv("OPENAI_API_KEY"))

tools = load_tools(["serpapi", "llm-math"],
llm=llm)

agent = initialize_agent (tools, llm, 
agent="zero-shot-react-description",
verbose=True
)

response = agent.run("What is the square
root of 144?")

print(response)
```

---

## What's happening here? 🧠

* The agent uses GPT models from OpenAI.
    
* It can decide independently whether to research or calculate.
    
* Tools like \`serpapi\` (web search) or \`llm-math\` (math operations) help it with this.
    

---

## Conclusion 💡

That was a very simple first introduction. In the next posts, I will show you

* How to integrate your own tools
    
* How to define goals for agents
    
* What a complete Agentic AI workflow looks like
    

---

*#Python #AgenticAI #LangChain #Coding #AIDevelopment #Beginner*