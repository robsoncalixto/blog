---
title: "The Code Agent That Improves With Every Run"
date: 2026-05-16T00:00:00-03:00
draft: false
tags:
  - ai
  - agents
  - harness-engineering
  - llm
summary: "Reading the AHE paper forced me to rethink where agent improvement really happens. The largest gains did not come from better prompts. They came from memory, tools, and middleware. Here is what that means for real engineering teams."
---

Over the past few months I have been experimenting with agentic coding techniques and realized that in some scenarios, evolving the prompt simply isn't enough. It was during a casual conversation about execution flows that I floated the idea of creating a markdown log file to store executed actions. That approach works, to a point, but it doesn't cover every scenario. Reading the paper [AHE: Agentic Harness Engineering](https://arxiv.org/abs/2604.25850), published on arXiv in April 2026, and going through the benchmark results, I developed a different perspective on how to apply a memory model to my harness. On top of that, this paper confirmed something I already suspected: memory is more valuable than prompt optimization. If you have been investing most of your optimization effort in prompt engineering, this paper has something to say to you too. Keep reading. I consolidated the ideas and draw some parallels from my own experience trying to build a corporate harness.

## What's a harness and why does it matter

Before getting to the result, you need to understand what the paper is actually optimizing.

A harness is everything that wraps an LLM and makes it interact with the world: the system prompt, yes, but also the tools available to the agent, how those tools are implemented, the middleware that intercepts and validates actions, the memory that persists across tasks, and the rules governing how the agent decides it's done.

If you have worked with LangChain, LangGraph, Cursor, or Claude Code, you already build or customize harnesses. The prompts and tool definitions are part of it. But so is the execution loop, the retry logic, the error handling, the way outputs are parsed and fed back. The harness is the environment where the model operates, not just what the model reads.

This distinction matters more than it seems. You can have a perfectly written system prompt inside a broken harness and the agent will still fail. The model is one input. The harness determines what the model can observe, what it can do, and whether it can recover when something goes wrong.

## How Agentic Harness Engineering (AHE) works

The AHE framework automates harness evolution through a closed loop with three distinct actors.

The **Code Agent** is the agent being optimized. It starts as a minimal bash-only agent. It runs on benchmark tasks and generates execution traces.

The **Agent Debugger** reads those traces, which can reach tens of millions of tokens, and distills them into structured failure reports. It doesn't dump raw logs. It produces layered evidence: summary patterns, task-level breakdowns, root-cause analysis, all traceable back to the original trace. The paper makes a sharp distinction here: logging tells you what happened; observability tells you why, in a form that another agent can act on.

The **Evolve Agent** reads the Debugger's reports and edits harness files. But it operates under one fundamental constraint: every edit must come with a prediction. Before changing anything, the agent declares what it expects to improve in the next round. After the round runs, the system checks whether the prediction held.

The paper calls this a falsifiable contract. It's the mechanism that separates systematic evolution from trial and error. Without it, an agent modifying its own environment is just guessing in the dark. With it, every change is a hypothesis, every result is data, and the agent learns not just what works but what doesn't and why.

The harness itself is decomposed into seven independent file-level components: system prompt, tool descriptions, tool implementations, middleware, skills, sub-agent configurations, and long-term memory. Each is versioned and individually revertible. If a change degrades performance, you know exactly what changed and can roll it back without affecting anything else.

## The result that made me uncomfortable

The paper reports that after 10 iterations, the AHE harness improved pass@1 on Terminal-Bench 2 from 69.7% to 77.0%, beating a human-designed harness and two self-evolving baselines. The researchers then isolated each component and measured how much it contributed individually:

| Component | Individual contribution |
|---|---|
| Long-term memory | +5.6 pp |
| Tools | +3.3 pp |
| Middleware | +2.2 pp |
| System prompt | **-2.3 pp** |

Surprisingly, the system prompt edited in isolation made things worse.

The largest gain came from memory: the ability to accumulate experience across tasks and carry it forward. The second came from tools: making execution mechanisms more robust. The third came from middleware: intercepting risky or incorrect actions before they caused failures.

None of these are prompt work. They are engineering work.

This doesn't mean prompts don't matter. They do, and in the full evolved harness they contribute alongside everything else. But it suggests we may be treating agent improvement as primarily a prompting problem, when we are probably working on the wrong layer.

The failures that matter most in complex, multi-step coding tasks aren't fixed by better instructions. They're fixed by better mechanisms: memory that knows what has been verified, tools that handle edge cases gracefully, middleware that catches the agent before it declares success prematurely.

## The question I couldn't stop thinking about

Reading this paper as someone who has been implementing and coordinating harness creation and usage alongside spec-driven development, I kept running into a question the paper doesn't address. It's simply not its focus. To make sense of it, let me explain how I got there.

AHE uses benchmarks built on complex open source code, but these are mature repositories with many different test scenarios. In that context, the benchmark gives a clear pass or fail. The agent can evolve without human supervision because the quality of the criteria is high.

But in a real engineering team, with ambiguous requirements and acceptance criteria that exist because a business analyst negotiated them with a client, automated verification doesn't cover everything. The agent might pass all the tests and still build the wrong thing. Some guarantees require human judgment from people who don't always have the full context to decide, but still have to.

Does AHE break down in that context? I don't think so. I think the model changes, and I want to share a few thoughts on that scenario.

In a team with spec-driven development, the loop becomes: the agent observes, proposes, and the human approves. The agent runs tasks, the debugger distills what failed, the evolve agent drafts a memory entry: "this type of underspecified requirement consistently causes failures in this layer." The tech lead or architect reviews whether the pattern is real, whether the proposed rule makes sense, whether it should become an ADR, a spec template, or an update to the execution agent's prompt.

The agent contributes to the team's institutional knowledge without having final authority over guarantees that require human judgment. It's a different kind of evolution, slower and more constrained, but it applies the same principles: explicit components, distilled experience, predictions that get verified.

The falsifiable contract doesn't have to be automated to be valuable. A team that writes down what it expects a spec change to improve, and then checks whether it actually did, is doing decision observability manually. Most teams don't do this, which is exactly why the same types of failures keep recurring.

## What this means in practice

I'm not suggesting you drop everything and implement a self-evolving harness for your coding agent tomorrow. AHE is a research prototype that requires benchmarks, sandboxes, and careful experiment configuration.

But the principles are immediately applicable.

Make your harness file-addressable. If your agent's prompts, tool definitions, memory seeds, and runtime rules live as hidden constants scattered across your codebase, you can't evolve them systematically. They need to be explicit artifacts: inspectable, versioned, and attributable.

Treat tools and memory as first-class optimization targets, not afterthoughts. The benchmark results are a concrete reminder that the biggest performance gains might not come from the prompt at all.

When you change something about your agent's environment, write down what you expect to improve. Then check. This habit is exactly what I'm preparing to add to my own harnesses, because the value of falsifiable predictions is that they give intention to your artifacts. We also know that teams often generate artifacts through AI without reviewing them. This habit will tell you more about your system than any amount of casual iteration.

And if you lead a team: the same principles apply to your development process. Component observability means your team's shared knowledge, specs, guidelines, architectural decisions, agent prompts, should be explicit and traceable. Experience observability means your retrospectives should produce structured patterns, not just feelings. Decision observability means your process changes should come with predictions that get verified.

The point of this post isn't to suggest a new model based on the paper, but to draw a parallel with widely used techniques and provoke some reflection on how we can introduce an intentional process of verifying the intent behind our harness artifacts. I close more convinced than ever that there's a lot to explore beyond waiting for bigger models. This paper shows we still have room to improve environments, evidence, and engineering discipline around the systems that make models act.

## References

- [AHE paper: arXiv:2604.25850](https://arxiv.org/abs/2604.25850)
- [AHE official repository](https://github.com/china-qijizhifeng/agentic-harness-engineering)
- [Terminal-Bench 2.0](https://www.tbench.ai/)
- [SWE-bench Verified](https://www.swebench.com/verified.html)
