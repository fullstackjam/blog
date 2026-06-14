+++
title = "Claude Code + Codex + OpenCode: My AI Coding Toolkit"
date = 2026-06-08
description = "A developer's real-world AI coding workflow: Claude Code for main development, Codex for cross-model review, OpenCode for scheduled tasks and Feishu-dispatched work. Why one tool can't do it all."
tags = ["AI", "coding-tools", "workflow", "Claude Code", "Codex", "OpenCode", "productivity"]

[extra.comments]
issue_id = 8
+++

"I just use Claude Code for everything. Why would I need anything else?"

If that's what you're thinking, I get it. A month ago I thought the same. Claude Code is strong — one tool for all your daily dev work, no problem.

But after a while, I found three problems that one tool can't solve.

<!--more-->

## Three Problems You Can't Avoid

One is cost. Claude Code's non-interactive mode (`-p` flag) moves to a separate credit pool after June 15, billed at API rates — essentially extra charges. Fine for occasional use, but for scheduled tasks and automated chores, costs become unpredictable.

One is model blind spots. When the same model reviews its own code, it'll most likely say "looks good," even rationalize why it's correct. Every LLM does this. Same-model self-review has blind spots.

The last is model matching. Formatting files or running routine checks with Opus is overkill. Opus isn't fast either, and some tasks just need something cheap and quick.

Once I thought through these three things, my tool combination naturally took shape.

## My Three-Tool Setup

<img src="/images/ai-workflow-overview.en.svg" alt="AI coding toolkit: Claude Code 70% main development + OpenCode 30% miscellaneous tasks" style="width:100%;max-width:900px;" />

Three tools, three roles, each handling their own lane.

## Main: Claude Code

Lots of people use Claude Code for daily dev. I'll just share my personal preferences: Opus 4.8 for heavy lifting, Sonnet 4.6 for daily work.

Many assume Claude Code requires Opus full-time. It doesn't. I use Sonnet 4.6 for the bulk of my tasks — batch renames, boilerplate, writing tests, simple bug fixes. Fast, stable, enough. Claude Code lets you switch models anytime via `/model`, and Sonnet is my default.

Opus 4.8 is reserved for tasks that need deep reasoning: complex architecture design, tricky bug debugging, cross-file refactoring.

One more thing: `CLAUDE.md` — it defines project conventions and persistent instructions, auto-loaded every session. I put coding style and commit conventions in there, so both Opus 4.8 and Sonnet 4.6 produce consistent output. OpenCode can also read this file — more on that later.

Claude Code handles the heavy work, but its output still needs an independent pair of eyes. That's where the second tool comes in.

## Sidekick: Codex Cross-Review

Same-model review has blind spots. The fix is simple: have another model at the same level review it.

The key is "same level." Opus 4.8 and GPT 5.5 are both SOTA, and a weaker model won't catch Opus's mistakes.

I use [Codex in Claude](https://github.com/openai/codex-plugin-cc), an official OpenAI plugin that calls Codex directly from Claude Code for review. Setup:

```
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

Two main commands: `/codex:review` for standard review, `/codex:adversarial-review` for adversarial review — the latter actively challenges your design decisions, sharper than regular review. For complex tasks you want to offload, there's `/codex:rescue`.

Different architectures and training data mean different focus areas. Where Claude sees no issue, GPT might raise a flag. Same principle as human code review: the author's self-check is never as good as a fresh pair of eyes.

## Grunt Work: OpenCode

Development and review are covered. The remaining question: who runs the daily chores — scheduled tasks, small fixes, things not worth firing up Claude Code for? That's OpenCode's spot.

### Why OpenCode

[OpenCode](https://opencode.ai/) is a Go-based open-source AI coding tool with 170k+ GitHub stars. TUI and CLI modes, supports 75+ LLM providers.

Why I chose it for grunt work:

- **Multi-model support.** I pair it with [Oh my openAgent](https://omo.dev) (OMO for short) as a plugin. OMO is an agent orchestration framework with 11 specialized agents that auto-match models to task types. Blazing fast.
- **Solid fallback.** When the primary model goes down, it auto-switches to backup. Critical for unattended scheduled tasks.
- **Great compatibility.** OpenCode natively supports `AGENTS.md`; without one, it falls back to reading `CLAUDE.md`. It also recognizes files under `.claude/skills/`. Claude Code project configs work out of the box — zero extra setup.

Running OpenCode with cheaper models for these chores neatly sidesteps the Claude Code non-interactive mode pricing issue.

My primary models on OpenCode are GPT 5.5 and DeepSeek V4 Pro. I also have a GitHub Copilot subscription for trying other models when needed.

### Scheduled Tasks: Multica

I hand scheduled task orchestration to [Multica](https://multica.ai/).

Multica is an open-source coding agent management platform (35k+ GitHub stars) that treats agents as team members. Agents get their own profiles, show up on task boards, can create issues, leave comments, and proactively report blockers.

It currently supports 12 coding agent tools including Claude Code, Codex, OpenCode, and Gemini. I mainly use its **Autopilots** feature for scheduled triggers: cron expressions or webhooks create tasks on a schedule, auto-routed to OpenCode for execution.

Tasks have full lifecycle management: queued, claimed, running, completed — every step tracked. It also has a **Squads** feature for grouping multiple agents under a leader for task distribution.

Compared to writing your own cron jobs, Multica has two clear advantages: agents proactively report problems instead of failing silently, and **every execution is fully logged** — reviewable after the fact.

### Feishu Dispatch: Lark Coding Agent Bridge

This is a tool built on top of the open-source [lark-coding-agent-bridge](https://github.com/zarazhangrui/lark-coding-agent-bridge). The upstream project supports Claude Code and Codex; my fork adds OpenCode support so it can be called from Feishu (Lark) as well.

**Repo:** [lark-coding-agent-bridge](https://github.com/fullstackjam/lark-coding-agent-bridge)

The principle is straightforward:

<img src="/images/lark-bridge-flow.en.svg" alt="Lark Coding Agent Bridge message flow: Feishu → WebSocket → Bridge daemon → OpenCode" style="width:100%;max-width:780px;" />

Send a message to the Bot on Feishu, Feishu's server pushes the event via WebSocket to the local Bridge daemon, Bridge forwards it to OpenCode for execution. The process streams back to Feishu as real-time card updates.

Fork additions: each Feishu chat window maps to an independent OpenCode session with no context bleed; permission prompts surface as interactive cards with Allow/Reject buttons; OMO background tasks proactively push notification cards when they finish.

What kind of work gets dispatched via Feishu? Small things not worth opening a terminal for:

- Quick lookup: "What's the return format for this API?"
- Simple file edit: "Change the timeout from 30 to 60 in config"
- Status check: "Run lint, any new warnings?"
- Small request: "Add parameter validation to this function"

Each one is trivial alone, but the cumulative context-switching adds up. Feishu dispatch lets me handle them on the go — walking, in meetings, whenever.

## How It Feels After a Month

After a month with this setup, the biggest takeaway: division of labor between tools matters more than any single tool's strength.

A real example. Claude wrote some concurrent logic once — I reviewed it and thought it was fine, Claude's self-review agreed. I threw it to Codex for adversarial review, which flagged a race condition: two goroutines writing to the same map without lock protection. Claude looked again and confirmed the bug. Without cross-review, this would likely have shipped to production.

Once the work is split up, Claude Code writes code, Codex catches mistakes, and OpenCode handles chores. Higher efficiency, lower cost.

This is my workflow. If you only do interactive development, one tool is enough. But once you add scheduled tasks, cross-model review, and mobile dispatch, a single tool hits its limits fast. Spending time finding the right tool mix pays off more than forcing everything through one tool.
