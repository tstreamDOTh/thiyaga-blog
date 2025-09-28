---
title: "The 90/10 Rule: The Inconvenient Truth About Agentic AI - It’s All Plumbing, No Brain"
date: 2025-09-10T10:30:00+05:30
draft: false
tags: ["AI", "Software Development", "Agentic AI", "Engineering", "Machine Learning"]
categories: ["AI Engineering"]
cover:
  image: "/images/agentic-ai-plumbing.jpg"
  alt: "Agentic AI Plumbing"
  caption: "The 90/10 Rule: Building Agentic AI Systems"
description: ""
showToc: false
TocOpen: false
---

### The Real Challenge in Building AI Agents Isn't the AI - It's Everything Else


The AI industry has a marketing problem. We’ve become so infatuated with the "intelligence" in artificial intelligence that we’ve forgotten the most important truth about building agentic AI systems: 

**90% of the work is software engineering, and only 10% is actually about the AI model itself.**

This isn’t just a hot take - it’s a hard-learned lesson from intense 11 months of building AI interview and agentic systems at Eightfold AI, where this realisation became both the problem statement and a career pivot. While everyone’s debating which foundation model has the highest benchmark scores, the real battles are being fought in error handling, state management, and API integrations.

### What Are We Actually Talking About? A Human-Friendly Guide

Before diving deeper, let’s clarify what we mean when we throw around terms like "agents," "agentic workflows," and "agentic AI" - because the industry loves to use these interchangeably when they’re actually quite different.

#### AI Agents

Autonomous systems that can perceive their environment, make decisions, and take actions to achieve goals.

#### Agentic Workflow

Structured processes where AI systems can make decisions and take actions at various steps, but within predefined guardrails. Imagine a smart assembly line where each station can adapt based on what it receives.

#### Agentic AI

The broader category encompassing both - any AI system that exhibits agency, whether it’s making simple decisions in a workflow or operating as a fully autonomous agent.

**The key distinction?** Autonomy and scope. A workflow might let AI decide which email template to use, while a full agent might decide whether to send the email at all, whom to send it to, and what follow-up actions to take.

### The Inconvenient Truth: It's All About the Plumbing

Here’s what no one wants to admit: building production-ready agentic AI feels less like training neural networks and more like being a really sophisticated plumber. You’re not crafting algorithms - you’re connecting pipes, managing flow, handling scale, and making sure nothing leaks when the system is under load. The problem gets further complex when you are building realtime AI systems.

Consider what happens when you build even a simple AI agent that can book meetings:

* **Error handling**: What happens when the calendar API is down? When the AI misunderstands the request? When there are conflicting time zones?
* **State management**: How do you track the conversation context across multiple interactions? What if the user gets disconnected, how does the agent is aware of what the user is seeing?
* **Integration management**: How do you handle rate limits across different APIs? Authentication? Data format mismatches?
* **User experience**: How do you provide feedback when the agent is thinking? How do you handle failures gracefully ?

The AI model itself - the part that actually understands language and generates responses - is often the easiest part to implement. It’s everything else that takes months to get right.

### Where the Real Moats Are Built - The 90%

While everyone’s focused on model performance, the real competitive advantages in agentic AI are being built in the boring stuff. Here’s where the actual moats lie:

#### Context Management

Most AI models have context windows, but real conversations don’t fit neatly into these limits. Building systems that can maintain relevant context across extended interactions, summarize previous conversations intelligently, and surface the right information at the right time - that’s pure software engineering.

The best agentic systems don’t just remember everything; they remember the right things. This requires sophisticated context compression, relevance scoring, and retrieval systems that would make a search engineer proud.

#### Memory Systems

Human memory isn’t just storage - it’s reconstruction, association, and forgetting. Building AI agents that can form lasting memories, make connections across time, and even strategically forget irrelevant information requires complex data architecture, caching strategies, and information architecture.

#### Workflow Integration

Real agents don’t live in isolation - they need to work within existing business processes, connect to enterprise systems, and hand off seamlessly to humans when needed. This means building robust integration layers that can handle:

* **API orchestration**: Managing complex sequences of API calls across different services
* **Data transformation**: Converting between different data formats and schemas
* **Business rule enforcement**: Ensuring the agent operates within company policies and compliance requirements
* **Handoff protocols**: Smoothly transferring control between AI and human operators

#### Safeguards and Reliability

The more autonomous your AI agent, the more ways it can fail catastrophically. Building production-ready agents means implementing multiple layers of safety:

* **Output validation**: Checking that the agent’s responses are appropriate, accurate, and safe
* **Action verification**: Confirming that planned actions align with user intent and business rules
* **Rollback mechanisms**: Being able to undo actions when things go wrong
* **Monitoring and alerting**: Detecting when the agent is behaving unexpectedly
* **Circuit breakers**: Automatically stopping the agent when error rates spike - Gracefully handle the user experience when there are upstream service issues

#### User Experience Design

Users don’t care about your model’s perplexity scores - they care about whether the agent feels helpful, trustworthy, and predictable. Great agent UX requires:

* **Transparent communication**: Users should understand what the agent is doing and why
* **Graceful degradation**: When things go wrong, fail in a way that maintains user trust
* **Feedback loops**: Allow users to correct the agent and learn from these corrections

#### Distribution and Deployment

The best agent in the world is useless if users can’t access it where and when they need it. This means building distribution systems that can:

* **Multi-channel deployment**: Work across web, mobile, Teams, Slack, email, and other channels
* **Scalable infrastructure**: Handle varying loads without degradation
* **Analytics and instrumentation**: Understand how agents are actually being used

### The 10% That Actually Matters: Foundation Models and Rapid Evolution

So what about that 10% that’s actually AI? The foundation model landscape evolves so rapidly that adaptability trumps picking today’s "best" model.

Take voice AI as an example: OpenAI’s Whisper dominated speech-to-text, but Deepgram’s Nova emerged with superior accuracy and speed. Meanwhile, ElevenLabs revolutionised text-to-speech with natural-sounding voices, forcing everyone to reconsider their audio pipelines.

In the past year, ElevenLabs went from primarily text-to-speech to launching real-time conversational AI, while LiveKit evolved from basic WebRTC infrastructure to offering AI-powered voice agents and multimodal capabilities.

**The smartest teams build model-agnostic systems** that can adapt as new options emerge, because today’s state-of-the-art becomes tomorrow’s baseline.

### The Bottom Line

The uncomfortable truth is that companies building successful agentic AI are software companies first, AI companies second. While some chase benchmark scores and build impressive demos, the winners focus on reliability, user experience, and seamless integration. The future belongs to those who master the plumbing, not just the brain.

---

*Want to dive deeper into building production-ready AI systems? I will be sharing a detailed series on Voice Agents in the upcoming weeks.*

---

*Originally published on [DEV.to](https://dev.to/thiyagarajt/the-9010-rule-the-inconvenient-truth-about-agentic-ai-its-all-plumbing-no-brain-5ca3)*
