---
title: "How We Manage LLM Costs Across Billions of Tokens"
date: 2026-08-10T00:30:00+05:30
draft: false
tags: ["AI", "LLM", "Cost Management", "Engineering", "Platform", "AI Engineering"]
categories: ["AI Engineering"]
cover:
  image: "/images/llm-cost-management-banner.png"
  alt: "A scatter of small filled dots converging through a funnel into one solid square"
  caption: "Dozens of teams, billions of tokens, one sane bill"
description: "Thousands of AI interviews a day, dozens of teams calling LLM APIs, one sane bill. Here's the system."
showToc: false
TocOpen: false
---

*Thousands of AI interviews a day, dozens of teams calling LLM APIs, one sane bill. Here's the system.*

---

Our CTO has one of the cooler awards you'll see in an office: a plaque from OpenAI for crossing 10 billion tokens.

I love that plaque. It also mildly terrifies me. Ten billion tokens is not a milestone you hit by accident. It means AI is wired into everything we ship, and every one of those tokens was paid for. When you burn tokens at that scale, managing the budget well isn't a finance chore. It's the thing that decides whether you get to keep building.

I learned how fragile that can be the hard way. A while back, someone from leadership messaged me: "Can you share a TLDR of our AI costs in the next hour or two? Need to do a cost projection for a customer."

I had a spreadsheet with projections. I pulled the actual provider bill to double check. The two numbers didn't match. Not wildly off, but off enough that I couldn't explain the gap on the spot, and off enough that I've never forgotten the feeling.

That gap is the real problem with LLM costs. It's not that they're high. It's that they're invisible until someone asks.

## Why LLM cost management gets hard at scale

At a small startup, one person can hold the whole AI bill in their head. At our scale, we can't. We run AI interviews, document parsing, translations, recruiter copilots, career agents. Dozens of teams, each making LLM calls, each with a good reason.

The two obvious answers are both wrong.

Lock it down with a central review board, and you've killed the thing that makes AI features worth building: the speed of trying stuff. Let everyone call the API freely, and one enthusiastic batch job runs over a weekend and eats a quarter's budget. I've heard versions of both stories from friends at other companies.

So the interesting question isn't "how do we cap spending." It's "how do we make spending visible and owned, without slowing anyone down."

## Give every LLM call a named owner

No LLM call leaves our platform without a registered caller ID. The client library rejects calls from IDs it doesn't recognize. This sounds boring. It's the whole foundation. When the bill arrives, there is no "miscellaneous AI spend." Every dollar traces back to a named feature owned by a named team.

## Approve new AI features with a Slack message, not a committee

To get a caller ID, you post in an approval channel: what you're building, which model, estimated cost per call and per month. A few platform owners read it and reply. Most requests get approved the same day. But they read carefully. I watched one recent estimate get pushed back on and corrected from about $1,100 down to $400, because the requester had priced everything at flagship rates when half their models were cheaper. The review made the estimate better, not slower.

## Throttle token usage at runtime

We track tokens per minute for each pool of API keys. When a pool runs hot, calls spill into a reserve pool instead of failing. A spiky day degrades gracefully instead of taking features down.

## Isolate each team's budget with separate API keys

Each team's API key lives in its own provider project with its own budget. If something does run away, it drains one project, not the company.

## Monitor cost per feature and alert on outliers

Every call gets logged with its name and token counts. Cost per caller ID feeds internal dashboards with alarms on outliers, so a feature that suddenly spends far outside its normal pattern gets flagged early instead of surfacing weeks later on an invoice. Once in a while, someone also compares actual spend against what was promised in that approval message. That's how we've caught drift, including the gap that started this post.

## Why this doesn't slow innovation down

The teams here don't experience this as bureaucracy, and I think that's the trick. The expensive thing (tokens) is gated by a cheap thing (a Slack message with an estimate). Asking costs you ten minutes. And writing the estimate forces you to understand your own feature's economics before you ship it, which every team should want anyway.

That plaque at our office says 10 billion tokens. The reason it can keep counting up is that every single one of them has a name on it.
