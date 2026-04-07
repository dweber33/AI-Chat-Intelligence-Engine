# Lessons Learned: AI Prompt Engineering, Model Cognition, and Chain-of-Thought Reasoning

> A field study from building a production AI intelligence engine on top of real IT operations chat data.

---

## Background

When I started building the Chat Intelligence Engine, the goal was straightforward: take the raw, unstructured conversation happening inside Microsoft Teams incident channels and turn it into structured, actionable shift intelligence, automatically, every twelve hours.

What I didn't anticipate was that the hardest engineering problem wouldn't be the pipeline. It would be understanding how different AI models *think*, and discovering that the way you write a prompt is not a stylistic choice. It is a direct reflection of the cognitive architecture of the model you're talking to.

This document captures what I learned by running the same operational data through different LLMs, observing where each one succeeded and failed, and iterating the prompts until the output was reliable enough to hand to engineers coming on shift and leadership reviewing weekly gaps.

---

## The First Mistake: Assuming Models Are Interchangeable

The early assumption was that swapping one LLM for another was roughly equivalent to changing an API endpoint. Same input, same prompt, roughly similar output, just with different latency and cost profiles.

That assumption was wrong, and it was wrong in a way that matters operationally.

When you send a complex IT shift log to different models with the same prompt, you do not get the same response with different formatting. You get fundamentally different *reasoning outputs*, because different models have different cognitive architectures. They have different relationships to constraints, different thresholds for inference, and critically, different capabilities for reasoning through a problem before producing an answer.

Understanding this changed how I think about AI integration entirely. You are not configuring a software module. You are working with an engine that has distinct reasoning traits, and your prompt is the mechanism through which you try to direct those traits toward a useful output.

---

## Two Cognitive Profiles in Production

### Google Gemini 2.5 Flash, The Inferential Engine

Gemini reads between the lines. When given a prompt with inclusion rules and exclusion rules, it does not apply them literally, it applies them *interpretively*, filling semantic gaps that the prompt author never explicitly defined.

In practice, this looked like this: the prompt asked Gemini to identify "Major Incidents" and listed trigger keywords including `Bridge`, `Down`, `Critical`, and `Impact`. A shift log came in referencing a "Call Center Bridge" without using the word "Major Incident" anywhere in the text.

Gemini elevated it. Without explicit instruction to do so, it reasoned that a Call Center Bridge implied business impact, that business impact implied severity, and that severity warranted inclusion in the Executive Impact Summary. It bridged the semantic gap between a symptom in the logs and a business consequence that was never spelled out.

This is powerful. It is also unpredictable if you do not account for it. Gemini's inferential style means it can surface things you did not ask for, in ways that are sometimes exactly right and occasionally require tighter boundaries to prevent over-classification.

The prompt that worked with Gemini was written as a set of *rules and principles*, exclusion criteria, inclusion triggers, channel priorities, and Gemini applied judgment to everything in between:

```
### 🛑 EXCLUSION RULES (IGNORE THESE):
1. NOISE: Ignore 'Circuit Flaps' and 'Interface Bounces'.
2. ROUTINE: Ignore successful, routine Changes unless they caused an outage.
3. REQUESTS: Ignore all RITMs.
4. SILENCE: Ignore banter, lunch breaks, or 'checking in' messages.

### 🚨 INCLUSION RULES (PRIORITIZE THESE):
1. CHANNELS: Focus heavily on conversations from Droids, Jedi, Empire, Rebel.
2. TRIGGERS: Look for keywords: 'Bridge', 'Down', 'Impact', 'Critical', 'Rollback', 'Failed', 'SITMAN'.
```

That was enough. Gemini filled in the rest.

---

### Anthropic Claude Sonnet, The Literal Engine

Claude is a constraint-bound model. It does not infer beyond what you explicitly permit. This is a direct consequence of how Anthropic has trained it, Claude is designed to be safe, honest, and precise, which means it defaults to the most conservative interpretation of any instruction rather than the most expansive one.

In production, this created a specific failure mode: Claude would receive a prompt containing the rule *"List only TRUE Major Incidents"* and then encounter a shift log describing a firewall link failure causing network degradation. Because the log entry did not contain the string "Major Incident", engineers were calling it a "link down" event and a "failover", Claude determined it did not qualify under the rule. It moved the entry to the Watchlist rather than the Executive Impact Summary.

This was not a bug. It was Claude doing exactly what it was told. The rule said "TRUE Major Incidents" and Claude had no basis to classify a "link down" event as one without explicit permission. Where Gemini would have inferred the escalation, Claude refused to assume it.

The initial Gemini-style prompt failed with Claude because it relied on implicit reasoning to bridge gaps. Claude does not bridge gaps. Claude stops at the boundary and waits for explicit instruction.

---

## The Prompt Engineering Shift: From Rules to Heuristics

The resolution was not to simplify the Claude prompt. It was to make it significantly more sophisticated, transitioning from writing rules that describe *what the output should contain* to writing heuristics that describe *how the model should think before producing output*.

This is the core lesson of this entire study: **for reasoning-capable models, the prompt is not a specification. It is a cognitive scaffold.**

Three specific techniques produced reliable results.

### 1. Explicit Intelligence Directives

Literal models require concrete, named triggers to override their default conservatism. Rather than relying on the model to infer that a Call Center Bridge implies critical severity, the prompt had to state it directly, and label it as an intelligence rule rather than a filter rule:

```
# 🚨 Executive Impact Summary
(INTELLIGENCE RULE: You must elevate high-priority threats here. This includes
anything with a 'Bridge', 'Call Center Impact', 'Critical Links Down', 'P2/P1 
Alerts', 'Hardware Overheats', or 'BGP Neighbor Drops'. Act like a senior 
manager; do not bury critical infrastructure risks in the watchlist.)
```

The phrase "Act like a senior manager" is doing significant work here. It is not decoration, it activates the model's professional-context reasoning. Claude has extensive training on management and operational communication. Invoking that context shifts the model out of rule-application mode and into judgment mode.

### 2. Defining the Negative Space with Absolute Precision

Claude will exploit any ambiguity in an exclusion rule. If an exclusion is written loosely, the model will find cases where the exclusion technically does not apply and include items that should have been filtered. The fix is to make the exclusion airtight and absolute:

**What failed:**
```
Only list failed changes or major data center upgrades.
```
The model listed successful major upgrades because the rule named "major data center upgrades" without requiring them to be failed.

**What worked:**
```
STRICT RULE: NEVER list a successful change in this report. Even if it is a 
major Data Center upgrade, if it succeeded, exclude it completely.
```

The word `NEVER`, the double-down on "even if," and the explicit end-state ("if it succeeded, exclude it completely") close every interpretive loophole. Claude respects absolute constraints. Write exclusion rules as absolutes.

### 3. Persona Injection for Synthesis Tasks

The GAP Analysis section of the output required something qualitatively different from the rest, not filtering or classification, but synthesis. Taking isolated events (an MTU mismatch on a vendor circuit, a failover that exposed a monitoring gap, a manual workaround that engineering shared in chat) and connecting them into a systemic vulnerability assessment.

Persona injection was the technique that unlocked this. A single sentence in the prompt:

```
Act as a Cloud Architect. Analyze the incidents above for systemic risks. 
Look specifically for vendor misalignments, lack of redundancy, or monitoring 
blind spots. Formulate a highly intelligent gap statement.
```

This works because large language models contain extensive training on domain-specific professional reasoning. The phrase "Cloud Architect" does not just set a tone, it activates a specific cluster of reasoning patterns: systems thinking, redundancy analysis, vendor risk assessment, infrastructure dependency mapping. The prompt is telling the model which of its capabilities to engage, not just what to produce.

The result was gap analysis output that read like it had been written by a senior infrastructure engineer rather than generated from a keyword match, because at the CoT level, that is effectively what was happening.

---

## Chain-of-Thought: The Capability That Changes Everything

Chain-of-Thought reasoning is the model's ability to work through a problem in structured intermediate steps before producing a final answer, rather than mapping input directly to output in a single pass.

For a summarisation task, "here are the logs, write a summary", CoT is not necessary. The model can produce a plausible output by pattern-matching against its training.

For an intelligence task, "here are the logs, identify which events represent systemic risks, classify them by severity, synthesise them into an executive assessment, and surface gaps in architecture and process", single-pass pattern matching fails. The model needs to evaluate evidence, apply constraints, weight competing interpretations, and reason toward a conclusion. That requires CoT.

This distinction matters because not all models have it. Several models evaluated during this project produced outputs for the same prompt that were essentially confident-sounding noise: fluent, well-formatted, and operationally useless. They could generate text *about* the logs, but they could not reason *through* them. There was no intermediate deliberation, no step where the model weighed whether a link failure with failover was a watchlist item or an executive escalation. It just produced a response that looked like a summary.

The practical test for CoT capability is not benchmarks. It is whether the model can correctly handle a case that requires resolving a conflict between two of its instructions. A model that can do this has an internal reasoning layer. A model that cannot will simply apply whichever instruction it encountered most recently, or default to its training distribution, and the output will be unreliable at exactly the moments it matters most, complex, ambiguous events that do not match a clean pattern.

Enterprise operational intelligence is almost entirely complex and ambiguous. Do not deploy a model without CoT capability for this class of task.

---

## The Prompt Comparison

Below are the core system prompts used with each model. Reading them side by side makes the cognitive gap visible.

**Gemini, Rule-based, inferential fill expected:**
```
You are the Executive Incident Manager for the Enterprise IT Account. Your job 
is to filter noise and present only Major Incidents and Critical Changes.

EXCLUSION RULES: Circuit Flaps, Interface Bounces, successful routine changes,
RITMs, banter.

INCLUSION RULES: Focus on Core-Routing, Server-Ops, Database-Admins,
Security-Ops. Triggers: Bridge, Down, Impact, Critical, Rollback, Failed, MIM.
```

Gemini reads the intent behind these rules and applies it contextually. The gaps between the rules are filled by inference.

**Claude, Heuristic-based, explicit directives required:**
```
You are a SENIOR Executive Incident Manager. Your job is to filter noise, use 
deep operational intelligence, and present a highly polished executive summary. 
You must use deductive reasoning to elevate systemic threats.

STRICT RULE: NEVER list a successful change. Even if it is a major Data Center 
upgrade, if it succeeded, exclude it completely.

INTELLIGENCE RULE: Elevate anything with a Bridge, Call Center Impact, Critical 
Links Down, P2/P1 Alerts, Hardware Overheats, or BGP Neighbor Drops. Act like 
a senior manager; do not bury critical infrastructure risks in the watchlist.

Act as a Cloud Architect. Analyze for systemic risks: vendor misalignments, 
lack of redundancy, monitoring blind spots.
```

The Claude prompt is longer, more explicit, and more directive, not because Claude is less capable, but because its reasoning style requires the author to do more of the semantic work upfront, then hand that scaffolded context to the model to reason within.

---

## What This Means for AI Integration

The conclusions from this study are specific enough to be actionable.

**Match the prompt style to the model's reasoning profile.** An inferential model like Gemini needs clear boundaries at the edges, tell it what to exclude and what to prioritise, and let it fill the middle. A literal model like Claude needs the middle spelled out too, through intelligence directives and persona injection.

**If the output is wrong, diagnose before you iterate.** Before changing the prompt, determine whether the model is failing due to a missing instruction, an ambiguous instruction, or a lack of reasoning capability. The fixes are different in each case. A missing instruction requires adding a rule. An ambiguous instruction requires tightening the negative space. A lack of reasoning capability requires a different model.

**Treat CoT as a baseline requirement for intelligence tasks.** Not every task requires reasoning. Summarisation, formatting, translation, these can be handled by models without deep CoT capability. The moment the task involves classification under ambiguity, conflict resolution between constraints, or synthesis of isolated events into systemic patterns, CoT is not a nice-to-have. It is the minimum viable capability.

**The prompt is a cognitive architecture document.** Once you accept that different models reason differently, prompt engineering stops being about finding the magic words that make the model behave. It becomes about understanding how a specific model processes constraints and context, and designing a prompt that works with that processing style rather than against it. The output quality ceiling is determined by the match between the prompt's structure and the model's cognitive architecture.

---

## Conclusion

Building this system taught me that integrating generative AI into enterprise operations is not a software problem. It is a reasoning problem. The infrastructure, the Lambda bridge, the Power Automate flows, the SharePoint buffer, is largely mechanical. The hard work is understanding the cognitive profile of the model you're deploying, designing prompts that work with its reasoning style, and knowing the difference between a prompt that produces plausible output and a prompt that produces *reliable intelligence*.

The shift from Gemini's inferential rules to Claude's explicit heuristics was not a step backward. It was a maturation, from telling the model what to produce to teaching the model how to think. The outputs that resulted were qualitatively different: not just better formatted, but genuinely more useful to the engineers and leaders reading them.

That gap, between a model generating text about a problem and a model reasoning through it, is where the real value of this technology lives.
