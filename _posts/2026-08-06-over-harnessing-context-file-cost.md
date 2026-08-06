---
layout: post
title: "Over-harnessing: what your context files actually cost"
date: 2026-08-06 09:00:00 +0800
author: christopher-nuland
categories: [Engineering]
topics: [agentic-safety, evaluation-driven-development]
projects: [agents-md]
image: "/assets/images/og/over-harnessing-context-file-cost.png"
description: "An agent will do exactly what you tell it to do — and every paragraph you add to the harness costs attention, accuracy, and GPU memory."
---

I once trained an AI agent to play Double Dragon, and it learned that the most efficient way to score points was to die and replay the level. Repeatedly, deliberately, right before the boss fight. I told the story on the Compiler podcast in an episode called ["Agentic AI: Working As Instructed"](https://www.redhat.com/en/compiler-podcast/agentic-ai), and the line I keep coming back to is the one I said then: an agent will do exactly what you tell it to do, which is frequently not what you meant.

That agent became good at the game only after I stopped tuning the reward and changed the objective, from maximizing points to exploring new experiences. Same game, same model, different framing; it soon started pulling off techniques speedrunners use.

I'm watching engineers make the identical mistake, except now the instructions are written in English, and there are four thousand tokens of them. The agent does something wrong, somebody adds a rule, the rule works on that case, and the system prompt grows by another paragraph. Nobody checks what the paragraph costs.

## **The cost nobody considers**

Rules are countable. Capability is not. So the rule count goes up forever, because adding one is a visible act of diligence and removing one looks like negligence.

The cost does not land where you are looking. It lands on the long sessions, the ambiguous inputs, and the cases that are not in your golden-path test suite, which is exactly the set of cases you built the agent for in the first place. You added the rule for the failure you saw, and you paid for it in the failures you never see.

Ralph Bean wrote this best in ["What even is the harness in AI?"](https://www.redhat.com/en/blog/what-even-harness-ai): "The sandbox and the harness have opposite design philosophies... The sandbox is subtractive... The harness is additive." The sandbox takes capability away, on purpose, and there is a natural stopping condition, which is the point where the agent can no longer do its job. The harness (your AGENTS.md, your skills.md files, your tools, your system prompt, your tests) only ever adds, and additive systems have no stopping condition at all.

That's the whole problem in one structural fact. Nothing in your workflow ever tells you to stop adding.

## **Attention divides, the harness compresses**

Long context doesn't make a model dumber by adding weight. It makes it dumber by dividing attention. Attention is a budget, a fixed pool of focus split across everything you show the model, so every token you add takes a share from the tokens that mattered.

Researchers built a [hard version of the needle in a haystack test](https://arxiv.org/abs/2502.05167) where the literal word match is stripped out, so finding the answer takes a real leap of understanding instead of a string match. One of the strongest reasoning models available scored 99.9% on the short version, and 31.1% once the input reached 32,000 tokens. Same model, same question, nowhere near its advertised limit.

That result isn't only about memory. Long input makes the right passage harder to find, and it degrades the reasoning that rides on top of that search; every model I've seen tested degrades as the input grows, well before the percentage reaches the max context size.

Then the harness starts throwing things away. When a session runs long enough to fill the window, the tooling summarizes and compresses the conversation and drops the rest, with a risk of not knowing which discarded detail you'll need twenty minutes later.

Two clocks run at once. Attention keeps dividing, the harness keeps compressing, and every paragraph you add speeds up both.

## **Two costs you are already paying**

The most immediate and obvious penalty is a drop in accuracy, a vulnerability that is likely embedded in your system prompt right now. A [study](https://arxiv.org/abs/2601.02023) evaluated standard anti-hallucination techniques, such as instructing the model to avoid fabrication and rely strictly on the provided text. When operating with a context window filled mostly with provided context ([skills.md](http://skills.md), RAG data, etc), the model's proficiency in extracting precise answers directly from the source material plummeted from 90.3% down to 72.0%.

The authors have a name for what happens next: Faithfulness Masking. You told it not to be creative, it heard don't risk it, and the safest output is no output, similar to my early Double Dragon results.

The second cost is the one with a dollar sign on it. Your context window isn't written once, it's re-read on every single request, and those tokens occupy memory for the whole conversation, which caps how many conversations one GPU can hold at once. Fewer conversations per GPU is a bigger bill, a slower service, or both. There are ways to reuse some of that work, and [I've written about one of them](https://developers.redhat.com/articles/2025/10/07/master-kv-cache-aware-routing-llm-d-efficient-ai-inference), but reuse doesn't give the memory back completely and does nothing at all for quality.

Somebody finally measured the paragraphs. Researchers handed coding agents [repository context files](https://arxiv.org/abs/2602.11988), and the agents plainly used them, reaching for exactly the tools the files named. Obedience was never the question. Cost per task rose about 20%, and on whether any of it helped, nobody could measure a difference. You're often paying a measured premium for an unmeasured benefit.

## **Where heavy constraint is correct**

Locking down what an agent may touch is correct. Which files, which commands, which endpoints; that's the sandbox, it's subtractive, and it costs you nothing in capability because it takes away reach rather than reasoning. This is all done in the runtime environment of the agent, not defined in the context of your session. 

While safety-critical and regulated operations often require this high-cost approach, those teams deliberately accept a documented trade-off for a specific advantage. The real issue is not constraint itself, but rather unpriced constraint, the administrative overhead that teams continuously deal with without evaluating the actual value.

## **Short and sweet beats a chapter book**

Anthropic's own [documentation](https://code.claude.com/docs/en/costs) tells you to move instructions out of the file that loads at session start and into skills, because those instructions sit in context even when you're doing unrelated work, and then it gives you a number: ***keep that file under 200 lines, essentials only***. Two hundred lines, from the company that built the skills harness.

Skill defines a name and a description, one or two lines the agent carries all the time. Activation happens only once a task actually matches, and that's when it reads the full `SKILL.md`. Everything else in the directory, the scripts, the reference files, the threshold table you nearly pasted into your prompt, sits on disk until something asks for it.

```
---
name: incident-escalation
description: Escalation policy and severity thresholds for production incidents. Use when an alert must be classified or routed to on-call.
---

# Escalation policy

Classify severity, then route. Full threshold table in references/severity.md.
Do not page on-call for SEV3 or below during business hours.
```

Two lines are loaded into context, everything else loads when it's relevant. The description is the load-bearing part, and it doesn't scale by piling on more of them. [One early study](https://arxiv.org/abs/2605.24050) stacked a library up to a couple hundred skills, and the agents got worse at picking the right one; that selection failure cost far more than the extra descriptions ever did. Fewer and sharper beats more.

At a certain point, a bloated harness simply transforms into a costly, GPU-based deterministic automation script. Therefore, try cutting your system prompt in half and rerunning your evaluations to observe what actually changes. What you see might surprise you.

