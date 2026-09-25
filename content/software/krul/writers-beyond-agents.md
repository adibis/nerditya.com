---
title: "Writers Beyond Agents: Regression, Telemetry, and Bucketing as First-Class Citizens"
weight: 4
date: 2026-09-21
publishDate: 2026-09-21
draft: false
description: "Every production AI-memory system surveyed assumes a writer is an LLM making a decision. A regression suite reporting a new failure signature is a writer too, and it changes the concurrency and schema story in ways worth designing for on purpose."
prev: /software/krul/concurrency-models
next: /software/krul/pheromone-invalidation
---

Read Mem0's write pipeline again, this time for what it assumes rather than what it does: extract candidate facts, search for similar existing memories, have an LLM tool call decide whether to add, update, delete, or do nothing. Read Graphiti's `add_episode` the same way: an episode arrives, gets processed through extraction and entity resolution, LLM calls throughout. Every system in the previous article's survey — Mem0, Letta, Graphiti, LangGraph — shares one architectural assumption so deep it's never stated as an assumption at all: a writer is an agent, or a pipeline built to look and behave like one, making a semantic decision about what to remember.

That assumption is false for an entire category of writer that matters more at tier 3's production scale than agents do, by volume: automated systems that report what already, factually, happened, with no language model anywhere in the write path. A regression suite that just ran ten thousand tests and found three new failures isn't deciding whether to remember something — it's reporting a fact, at a speed and scale no LLM call could sustain, and it should be a first-class writer, not something bolted onto a system designed around agent semantics.

![Two write pipelines side by side: an agent's multi-hop read-LLM-decide-write path with a wide gap where another writer can interleave, versus a regression result's single-step observe-and-append with no gap and no lock needed](/images/krul/04-writer-pipelines.svg)

## A regression run is a writer

Chip design verification already has a concrete example of exactly this shape: a daemon-triggered analysis that fires when a regression completes, correlating RTL changes against which tests are likely to catch a given commit. The natural instinct is to treat that daemon's *output* — the triage result — as the interesting artifact. Look instead at the *input*: the regression run itself is generating structured facts continuously, at a volume dwarfing anything a human or an agent could produce by hand. Every test that passes or fails is a potential write. Every failure with a signature that doesn't match a known bucket is a write that matters more than most agent-generated memories, because it's new information about the system under test, not a summary or an inference — the actual ground truth a whole downstream triage pipeline depends on.

Multiply this by what tier 3 of the scale ladder actually looks like: hundreds of thousands of writers isn't hypothetical once you count regression executions, CI jobs, and telemetry events alongside agent sessions. A single verification team running continuous regression can generate more structured facts per day from test results than every engineer on that team's agents combine to write in a month. Any architecture that treats "writer" as synonymous with "agent" has, by construction, under-designed for the majority of its actual write volume.

## What changes about the write itself

The interesting part isn't just that there's a new category of writer — it's that this category's writes have a fundamentally different shape than an agent's, and that shape has real, favorable concurrency consequences the previous two articles didn't get to use.

An agent's write, in every system surveyed, is a multi-hop process: read existing state, call an LLM (sometimes more than once), decide, then write. That's exactly the shape that produced Mem0's TOCTOU bug — the race window is wide because there are `await` points scattered through several round-trips between the read and the write, and anything can happen to the underlying state in that window. A regression result doesn't have that shape at all. "Test `axi_write_stress_042` failed at commit `a3f8c19`, signature matches bucket `write-buffer-race`" is a fact that's fully known the instant the test finishes. There's no LLM call between observing it and recording it, no multi-step decision process, no wide `await` gap for another writer to invalidate assumptions inside. It's closer to a pure append than to a read-modify-write — the same shape as a plain event log that appends one line per record: opened with `O_APPEND` on a local, POSIX-conformant filesystem, [the offset update and the write happen as a single atomic step](https://man7.org/linux/man-pages/man2/open.2.html), so a pure append needs no lock at all, unlike a counter that has to be read, incremented, and written back. That guarantee is real but not universal — the same documentation is explicit that it doesn't hold reliably on NFS, where the client kernel has to simulate append mode and can't do so without a race condition, so it's worth checking rather than assuming once the store moves off a local disk. Pure appends are concurrency-easy in a way read-modify-writes fundamentally aren't, and a huge fraction of a production system's actual write volume — regression results, telemetry events, raw error reports — is naturally append-shaped if the schema doesn't force it into something else.

That's not true of every non-agent writer, and it's worth being precise about where the easy case ends. Bucketing — clustering a new failure against known failure signatures, deciding whether it's a recurrence or something novel — is itself a write, and it's not a pure append. It reads the existing set of known signatures, compares, and either reinforces an existing bucket's count or creates a new one. That's a second-order writer, consuming raw telemetry as its input and producing structured knowledge-base entries as its output, and it has exactly the read-modify-write shape that needs the concurrency discipline the earlier articles covered — the same TOCTOU risk, the same need for either a lock at tier 1's scale or a transactional guarantee at tier 2 and 3's. The distinction that actually matters for design purposes isn't "agent versus not-agent." It's "pure append versus read-modify-write," and non-agent writers turn out to split across that line in genuinely useful proportions: raw fact reporting (a test result, a telemetry event) is almost always append-shaped; anything that aggregates, dedupes, or clusters (bucketing, trend detection) is read-modify-write-shaped regardless of whether an LLM or a rule-based classifier is doing the aggregating.

## What the two records actually look like side by side

Abstract talk about "writer shape" is easy to nod along with and hard to actually design against. Concretely, an agent's memory write — the shape every system in the previous article's survey is built around — tends to look something like this, whether it's Mem0's fact-extraction output or Graphiti's resolved episode:

```json
{
  "type": "agent_fact",
  "content": "The axi_write_buffer module's occupancy check has a one-cycle sampling gap during the read-grant window",
  "source": {"model": "claude-sonnet-5", "conversation_id": "c-88a1", "confidence": 0.7},
  "created_by_decision": "ADD"
}
```

Notice what's load-bearing in that record: a natural-language claim, a model identity, and an explicit decision type an LLM tool call produced. None of that maps cleanly onto what a regression harness actually has to report:

```json
{
  "type": "regression_result",
  "test": "axi_write_stress_042",
  "commit": "a3f8c19",
  "outcome": "fail",
  "bucket": "write-buffer-race",
  "duration_ms": 4211
}
```

There's no `"confidence"` field here worth having, because there's nothing probabilistic about it — the test either passed or it didn't. There's no `"created_by_decision"`, because nothing decided anything; the harness observed and reported. Forcing this record through a schema designed around the first shape means either inventing a fake confidence score for something that doesn't have one (a lie the schema tells to satisfy its own structure) or leaving fields null in a way that quietly signals "this record doesn't really belong here" every time it's queried. Neither is a good outcome, and both are avoidable by admitting up front that these are two different record shapes belonging to two different writer types, not one shape with optional fields.

## Why the schema has to say this on purpose

None of the four systems surveyed in the previous article carry a first-class notion of "this write came from a non-agent process reporting a fact, not an agent making a judgment." Mem0's ADD/UPDATE/DELETE/NOOP decision is something an LLM tool call produces; there's no documented path for a regression harness to just assert a fact without going through that machinery, and the whole design assumes it. Graphiti's episodes are things an agent's conversation produces. This isn't a criticism of any specific system's scope — none of them were built for chip verification or its equivalents in other engineering domains, and general-purpose agent-memory tooling has no obligation to anticipate every domain's telemetry shape. But it means the assumption has to be corrected deliberately in a design that does need it, not inherited by accident from tooling that never considered the question.

Concretely, that means the schema for this series' eventual reference architecture needs a writer-type field that isn't LLM-shaped at all: `agent`, `regression`, `telemetry`, `human`, with each carrying whatever provenance actually applies to it — an agent write carries a model identifier and maybe a conversation ID; a regression write carries a commit SHA and a test name; a telemetry write carries a source system and a timestamp; a human write carries whoever typed it. None of these need to be forced through a shared "the writer is an agent that decided X" shape just because that's the shape every existing system assumes by default. And it means the write path itself needs to branch on that shape: a pure-append fact report shouldn't be routed through the same multi-hop, LLM-mediated decision pipeline an agent's semantic memory write goes through, both because it doesn't need that machinery and because forcing it through anyway would import exactly the wide-await-gap concurrency risk that pipeline shape creates, for writes that never needed to carry that risk in the first place.

## What this sets up

The next article in this series takes up invalidation — deciding, automatically, what a shared knowledge base should stop trusting over time — and the writer-beyond-agents distinction matters there too, more than it might look like at first. A regression result that hasn't recurred in six months is a different kind of stale than an agent-written fact nobody's queried in six months; the first is genuinely good news (the bug got fixed), the second is ambiguous (maybe nobody needed it, maybe nobody found it). Getting invalidation right depends on knowing which kind of writer produced a given fact in the first place — one more reason the distinction this article makes has to be structural, carried in the schema from the start, rather than something inferred after the fact from a fact's content alone.
