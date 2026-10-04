---
title: "Gates In, Hooks Out: How the Orchestrator Actually Touches Memory"
weight: 8
date: 2026-10-03
publishDate: 2026-10-03
draft: false
description: "Being a daemon instead of an agent session is the whole premise of this series' orchestrator. Nothing about that comes with built-in safety, and nothing about it comes with a way to write back into memory either. Both had to be built."
prev: /software/krul/multi-model-multi-modal
next: /software/krul/reference-architecture
---

Every article so far has treated the orchestrator as settled: a daemon, not an agent session, dispatching stateless LLM calls and persisting the state those calls can't. That choice isn't incidental to this series, it's load-bearing, and it's worth stating plainly why before going further: a session is bounded by a context window and by the lifetime of the process holding it, and "hundreds of thousands of writers" over a production system's lifetime isn't a session, it's forever. [Anthropic's own guidance on long-running agents](https://anthropic.com/engineering/effective-harnesses-for-long-running-agents) makes the identical point independently, down to the phrasing: *"session is not the context window."* Their fix is the same inversion this series already makes — durable state lives outside the agent, in a progress file, a feature registry, a git log, with the agent reading and writing it, not holding it in its own head. [Their own multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system) puts a number on why: orchestrator-worker coordination runs at roughly 15 times the token cost of a single chat turn, for a task that completes in one sitting. Routing every write, invalidation check, and reinforcement signal in a continuously-running memory store through an LLM call instead of deterministic code doesn't scale by that multiplier, it compounds it indefinitely.

So the daemon is the right call. But being a daemon instead of an agent session means something uncomfortable: none of the safety machinery built for agent harnesses has any reach into it at all.

## The gap nobody built

Claude Code has a `PreToolUse` hook mechanism: a point where a tool call can be inspected and denied before it runs. It's a real, useful primitive, and it's scoped entirely to Claude Code's own tool calls. The daemon this series has been designing runs as its own process, dispatching gears whose `process`-type stages execute shell commands as direct subprocess calls, with no agent harness anywhere in that path. A `PreToolUse` hook, or any harness-specific equivalent, simply isn't there to ask.

That's not a gap specific to one bad gear definition. It's structural: a gear that happens not to contain a dangerous command isn't a safety property, it's the absence of one. Any daemon dispatching actions on its own needs its own interception point, because nothing upstream of it is going to provide one.

## Gates in: a real interception point, not a blocklist

The tempting fix is to harden the gear executor itself: scan a command for dangerous patterns and refuse to run it. That's a blocklist baked into the orchestrator's own code, and blocklists on shell command text are easy to defeat by accident, let alone on purpose — a different quoting style, an alias, a wrapper script, and the pattern match doesn't fire.

The fix that actually generalizes borrows the shape of `PreToolUse` itself rather than its scope: a real interception point the executor calls unconditionally before running a `process` stage, where policy lives as external, swappable data, not logic compiled into the daemon. A gear stays a pure workflow definition; it carries no opinion about what's safe, the same way a gear carries no opinion about what domain it's indexing. Concretely, a `[gates]` section in `krul.toml` points at a policy script. Before every `process` stage runs, the executor calls it with the gear name, stage id, and the full command text as plain argv entries, deliberately not through a shell, so the command text can never be reinterpreted as shell syntax by the check itself. Exit 0 allows the stage; anything else, including the script failing to launch, denies it. Fails closed, not open — a gate that can fail open isn't a gate.

Proof this actually gates, not just that it compiles, means checking the thing that matters: not that a denied stage gets labeled failed, but that the command underneath it never runs. A test built around a canary file (a `process` stage whose command would create a file if it ran) makes this checkable directly: deny the stage, assert the file was never created. That's a stronger claim than checking an exit code, because it rules out the gate reporting a denial while the command runs anyway.

A policy script only catches what it's written to catch. One scoped to a single kind of dangerous command, an unreviewed commit, say, says nothing about a different one, like a push. Text-pattern matching is inherently incomplete regardless of scope, since nothing requires a dangerous command to contain the literal string being matched against. The guarantee that would actually hold against a cleverer failure isn't a smarter pattern, it's making sure the daemon's own process never has the *capability* in its environment at all, no credential to push with, so there's nothing to authenticate against even for a command that gets past every check. That's not built here.

## Hooks out: the other half nobody built either

Denying a dangerous command is one shape of interception: it runs before an action and can block it. Conflating that with the whole idea of a "hook" would have missed something just as necessary, a point that fires *after* a gear finishes so something can react to the result.

Both git's own hook system and Claude Code's draw this exact line, consistently, across dozens of event types: a *pre* variant that can block, paired with a *post* variant that can only react. `pre-commit`/`post-commit`. `PreToolUse`/`PostToolUse`. `PreCompact`/`PostCompact`. The split isn't incidental to either system; it shows up everywhere a lifecycle transition matters enough to name twice. The daemon already had the fire-and-forget half of this for one event, `on_task_complete`, firing when an entry in the background task queue finishes. Nothing fired when a gear itself, as opposed to a background task, actually finished. Two more notifications close that: `on_gear_stage_complete` after each stage, `on_gear_complete` once the whole gear exits, either through its termination condition or by hitting `max_iterations`. Same mechanism, same shape, two more places it fires from.

## Where this is actually supposed to lead

Nothing registers against either hook yet. That's deliberate, not an oversight: building the point a handler fires from is a different decision from deciding what a handler does with it, and conflating the two would mean guessing at a design nobody's specified. But the intended destination isn't a mystery. [This series' own memory layer](/software/krul/pheromone-invalidation) already has a shape built for exactly this: strength that rises on reinforcement, decays on neglect. A gear's output is a writer in [the sense the fourth article argues for](/software/krul/writers-beyond-agents) — a coverage-closure gear's finding, a triage gear's hypothesis, these are facts a shared knowledge base should hold the same way a regression result or a human correction does. `on_gear_stage_complete` and `on_gear_complete` are where that writer would actually touch the graph: a write when a gear produces a finding, a reinforcement signal when a gear's result confirms something already stored. No handler does this today. It's the obvious next thing to build, and it's substantial enough to earn its own article once there's a real, tested one to write about rather than a design sketch.

What's real as of this article: the gate, proven by a test that checks the command never ran, not just that the stage was labeled failed; the two completion hooks, firing from the same mechanism already serving the task queue; and an honest account of what neither one covers yet. The next article assembles everything this series has built, across all three scales. This is one more real piece of what it has to assemble.
