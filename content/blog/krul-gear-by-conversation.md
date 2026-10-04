---
title: "krul's Gears Had No Safety Net"
weight: 1
date: 2026-10-03
publishDate: 2026-10-03
draft: false
description: "Turning a real review workflow into a krul gear meant building krul's first real safety net along the way — the gate, the hooks, a real memory leak caught by testing it. The gear itself was the easy, last part."
---

The ask was simple on its face: turn an existing code-review workflow into a [krul](/software/krul) gear. I run this workflow by hand today, on a Verilator fork I contribute patches to: a mechanical script (`gate.sh`) checks formatting, naming, copyright headers, and branch currency, plus a short, explicit list of judgment calls I still have to make on top of it — whether a new code path's full input-shape space actually got tested, for instance, something no script can check. Automate the mechanical half, hand the judgment half to an LLM, done.

What it actually took to get there is more interesting than the gear itself.

## The first draft looked fine, and was missing the only thing that mattered

My first pass at this looked fine on paper. [krul's gear format](/software/krul) is a short pipeline: a trigger phrase, a list of stages, each one either an `llm` call or a `process` shell command. Each stage's output is addressable by later stages through `{stage_id}` template tokens. A gear that runs `gate.sh`, feeds its output and a diff into a few LLM stages, and synthesizes a report is easy enough to describe, and I had it sketched out in about five minutes.

Then came the question that actually mattered: this workflow ends in a decision about whether to commit. Where, exactly, does "don't commit without review" get enforced once a gear is the thing deciding to run `git commit`?

That's when I realized I'd been relying on a hook that only works with Claude — not a reusable method at all. A `PreToolUse` hook can deny a commit on a repo unless some condition holds, but `PreToolUse` is Claude Code's own interception point, specific to that one harness. Point a different agent at the same repo and the hook simply isn't there. It has no reach into krul either, which runs as its own daemon process entirely outside any agent harness: when one of its gear stages runs a shell command, that's a direct subprocess call from krul's own executor, with no hook of any kind anywhere in the call path. A gear that happened not to contain a `git commit` stage wasn't a safety property. It was the absence of one.

## The wrong fix, and the one that actually generalizes

The first fix on the table was to harden krul's gear executor itself: scan a `process` stage's command for dangerous patterns and refuse to run it. I run AdGuard at home, and pattern-matching bad things out of a stream is a familiar move. But a blocklist baked into the tool's own code was the wrong answer here too: blocklists on shell command text are notoriously easy to defeat by accident, let alone on purpose — a different quoting style, an alias, a wrapper script, and the pattern match just doesn't fire.

What I settled on instead was the same thing I already rely on with Claude: hooks. krul needed its own version of what `PreToolUse` is for Claude Code — a real interception point the gear executor calls unconditionally before running anything, where policy lives as external, swappable *data*, not logic compiled into the daemon. A gear stays a pure workflow definition; it carries no opinion about what's safe, the same way krul's gears carry no opinion about what domain they're indexing.

That became a real `[gates]` section in `krul.toml`:

```toml
[gates]
process_script = "/path/to/a/policy/script"
```

Before every `process`-type stage runs, if a script is configured, krul calls it with the gear name, stage id, and the full command text as plain argv entries — deliberately not through a shell, so the command text can never be reinterpreted as shell syntax by the check itself. Exit 0 allows the stage. Anything else, a nonzero exit or the script failing to even launch, denies it. Fails closed, not open: a gate that can fail open isn't a gate.

A policy script for this specific repo just needs the same kind of stamp check a `PreToolUse` hook could do on the Claude Code side, applied here instead: hash the current tree (`HEAD`, working-tree status, and diff), compare it against whatever got written to `.git/gate-ok` the last time the mechanical gate passed cleanly, and deny the commit if they don't match. Same rule, now enforced somewhere that doesn't depend on which agent happens to be running it.

Proof the gate actually gates, not just that it compiled, meant a test that checks the thing that actually matters: not that a denied stage gets labeled failed, but that the underlying command never ran at all.

```zig
test "process stage is denied by gate script, and the command never actually runs" {
    hooks.init("tests/fixtures/gate_deny.sh");
    defer hooks.init("");

    const stage = gear.Stage{
        .id = "s", .kind = .process,
        .prompt = "echo x > " ++ canary_path,
    };
    const out = try runStage(ally, "g", &stage, "", "", &outputs);

    try std.testing.expect(out.failed);
    // Not just checking the reported status -- proves the shell command
    // itself never executed, not merely that the stage was labeled failed.
    try std.testing.expect(std.c.fopen(canary_path, "r") == null);
}
```

A canary file that only gets created if the command actually runs is a cheap, definitive way to tell "the gate said no" apart from "the gate said no, and it was lying."

The script only catches what it's written to catch, and today that's `git commit`, not `git push` — a command that assembled a push instead would sail straight through. Text-pattern matching is inherently incomplete anyway, since nothing requires a dangerous command to contain the literal string being matched against. The guarantee that would actually hold isn't a smarter pattern — it's making sure krul's own process never has push-capable credentials in its environment at all, so there's nothing to authenticate with even for a command that got past every check. Not built here, same discipline as the two completion hooks later in this piece: name the gap, don't hide it.

## What the gate's own test suite caught, unrelated to the gate

Writing that test suite turned up two things that had nothing to do with gating.

First, a real, pre-existing memory leak in the executor's own stage-output map. Every stage's id gets duplicated onto the heap as a hash-map key, and the cleanup code freed every stage's output content but never the key strings themselves. It had been there the whole time. Nothing had exercised that code path under a leak-detecting allocator before, because the gate's own tests were the first to call the full multi-stage runner instead of a single stage directly.

Second, a naming mismatch against documentation written *ahead* of the feature. krul's own README already had a table describing planned hooks named `on_gear_stage_complete` and `on_gear_complete`, written before either existed. The first version of this was called `on_stage_complete` instead — a plausible name on its own, but inconsistent with `on_gear_complete`'s "gear" prefix sitting right next to it, and silently wrong against what the docs had promised. The fix renamed the code to match the plan that existed first, not the other way around.

Writing the gear file by hand and calling it done would have caught neither.

## The gate wasn't the whole hook story

Denying a dangerous command is one shape of hook: it runs *before* an action and can block it. It isn't the only shape that matters, and conflating "hooks" with "a safety gate" would have left out something just as necessary: a hook that fires *after* a gear finishes, so something can react to the result — write a finding somewhere, trigger a follow-up action, tell a person it's done.

Both git and Claude Code's own hook systems already draw this exact line, consistently, across dozens of event types: a *pre* variant that can block, paired with a *post* variant that can only react. `pre-commit`/`post-commit`. `PreToolUse`/`PostToolUse`. `PreCompact`/`PostCompact`. The split isn't incidental; it shows up everywhere a lifecycle transition matters enough to name twice.

krul already had half of this: a fire-and-forget notification registry, `on_task_complete`, that an in-process plugin could react to. It just only fired from the background task queue, never from a gear finishing. The other half meant two new notifications, fired from the same mechanism, at the two points in a gear's lifecycle worth reacting to:

```zig
hooks.fire("on_gear_stage_complete", payload);  // after each stage
hooks.fire("on_gear_complete", payload);         // after the whole gear exits
```

Nothing registers against either yet. That's deliberate: building the hook point is a different decision from deciding what a handler does with it, and conflating the two would mean guessing at a design nobody's asked for yet.

## The actual gear, once there was somewhere safe to put it

With a real gate and a real completion signal in place, the workflow was finally safe to turn into an actual gear. I had Claude write it up in krul's own syntax: mechanical stages that run the real `gate.sh`, LLM stages that cover exactly the judgment gaps `gate.sh` itself documents as outside its own reach, and a synthesis stage that reads all of it honestly rather than assuming the mechanical half passed.

```yaml
stages:
  - id: mechanical
    type: process
    prompt: "cd <repo> && gate.sh --base origin/master 2>&1 | tail -40"

  - id: diff_text
    type: process
    prompt: "cd <repo> && git diff origin/master...HEAD -- '*.cpp' '*.h' | head -200"

  - id: comment_terseness
    type: llm
    prompt: "... flag any comment that isn't 1-3 lines, dense, matching the surrounding code's register ... {diff_text}"

  - id: shape_enumeration
    type: llm
    prompt: "... enumerate the full input-shape cross-product for any new code path, flag what this diff's tests don't cover ... {diff_text}"

  - id: standards_spec
    type: llm
    prompt: "... Standards axis and Spec axis, reported separately ... {diff_text}"

  - id: synthesize
    type: llm
    prompt: "... whether the mechanical gate passed at all, then each judgment finding, then one plain verdict ... {mechanical} {comment_terseness} {shape_enumeration} {standards_spec}"
```

Running the mechanical stage's command against the real repo surfaced something worth reporting honestly rather than smoothing over for a cleaner writeup: `gate.sh` itself currently fails in this environment, on a Python syntax error inside a copyright-check script, unrelated to any code under review. That's exactly the scenario `synthesize` has to handle correctly — report that the mechanical gate failed and why, not quietly skip ahead to the judgment stages as if nothing happened. A gear design only ever tested against a clean run wouldn't prove that.

## Where Claude actually came in

Everything up to the last section was mine to work out: the gap, the architecture, the test that proves the gate actually gates, the leak, the naming fix. None of it was obvious going in, and none of it would have shown up from writing a gear file by hand and calling it done — each piece only surfaced from taking the original ask seriously enough to ask where its one genuinely dangerous action, a commit, would actually get stopped if everything else went wrong.

Claude's part was narrower and came last: once the gate, the hooks, and the design were already settled, turning that design into krul's actual gear syntax, stage by stage. That's a real, useful thing to hand off — but it's the last mile, not the engineering. The gear in this post won't ship as a krul example either way; it's specific to my own Verilator fork, and krul stays deliberately unopinionated about any one domain. What's real and shipped is the mechanism it needed first: `[gates]`, the two completion hooks, and a test suite that proves a denied command never runs.

## Coming soon: wiring a gear's lifecycle into krul's own memory

The gate and the two completion hooks aren't just a fix for one repo's workflow — they're real additions to krul itself, and [the krul series covers them on their own terms](/software/krul/gates-in-hooks-out), past the one gear that happened to need them first.

The real reason I wanted a broader hook survey in the first place: [krul's own memory layer](/software/krul/pheromone-invalidation) — the knowledge graph this series has already covered, pheromone-style invalidation and reinforcement included — is exactly where a gear's output belongs. `on_gear_stage_complete` and `on_gear_complete` are the missing link: the lifecycle points a write, an invalidation, or a reinforcement signal should fire from. Nothing wires a gear's output into that graph yet. Once there's a real, tested handler doing it, that's its own article.
