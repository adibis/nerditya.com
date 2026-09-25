---
title: "A Reference Architecture at Three Scales"
weight: 8
date: 2026-09-21
publishDate: 2026-09-21
draft: false
description: "Everything from the previous seven articles, assembled into one architecture described concretely at each tier of the scale ladder: a well-established primitive at the smallest scale, borrowed-but-real at the middle one, and honestly speculative at the largest."
prev: /software/krul/multi-model-multi-modal
---

Seven articles is enough separate pieces that it's worth resisting the temptation to let them stay separate essays. This one assembles them, deliberately organized by the scale ladder from the second article, because the single biggest risk in a series like this is producing something that reads coherently chapter to chapter but never actually specifies a system a reader could build. Three tiers, each concrete, each honest about how much of it rests on an established primitive versus a genuine proposal.

![Five dimensions across three tiers: storage, concurrency, writers, invalidation, and telemetry, with confidence decreasing left to right from a well-established primitive through a borrowed real precedent to an honestly proposed design](/images/krul/08-reference-architecture.svg)

## Tier 1: one user, several agents

This tier is the least speculative of the three, not because this series has personally validated it in production, but because the primitive it rests on, [`mkdir`'s atomicity on a POSIX filesystem](https://pubs.opengroup.org/onlinepubs/9699919799/functions/mkdir.html), is a documented operating-system guarantee, not a novel claim. Confidence here comes from standing on well-established ground, which is a different, more honest kind of confidence than "we proved this ourselves."

**Storage:** plain text files with structured frontmatter, no database, because a single serial writer (or a handful of bursty concurrent ones, all on one machine) has no concurrency problem a database's machinery is built to solve. Git provides versioning and the one guarantee a lone writer actually needs: no update silently vanishes.

**Concurrency:** narrow, targeted locks around the specific operations that read-modify-write shared state, built from `mkdir` rather than any external dependency: atomic on POSIX filesystems, no `flock` requirement, no database. Per-file locking, not a single global choke point, so unrelated writes never contend with each other. A stale-lock timeout that reclaims a lock a crashed process left held, rather than wedging the system indefinitely.

**Writers:** predominantly agents at this scale, with the pure-append writes (a tool-call ledger, a simple event log) needing no lock at all: small, [`O_APPEND` writes are atomic](https://man7.org/linux/man-pages/man2/open.2.html) on a local, POSIX-conformant filesystem, a distinction worth keeping even at this small scale because it's free, and worth re-checking rather than assuming the moment the store isn't a local disk anymore.

**Invalidation:** a single incrementing counter per entry, bumped on genuine recall, checked against age by an occasional human-reviewed sweep. Not evaporation yet: at this scale, with a corpus small enough for good filenames to beat any ranking algorithm, a full decay-curve implementation would be solving a problem this tier doesn't have.

**Telemetry:** retrieval-time counter, nothing more. This tier doesn't need consolidation weighting or a retraining loop; it needs exactly the mechanism the opening research found industry-standard at the smallest scale, correctly sized down.

This tier is buildable today, by one person, in an afternoon: every piece of it is off-the-shelf, well-understood, and small enough to verify with a direct load test rather than trusting it on faith.

## Tier 2: a small team

This tier hasn't been built end-to-end in this series. Every piece of it, though, is borrowed from a real system checked directly against its own source, not invented from scratch. That's the discipline the concurrency-models article argued for explicitly: don't design bespoke machinery for agent memory specifically, borrow a transactional guarantee deliberately and know exactly how far it extends.

**Storage:** a real transactional database (Postgres is the default choice this series has checked directly and would reach for first) replacing flat files specifically because the writer count has crossed from "a lock resolves rare collisions cheaply" into "collisions are routine enough to need a system built for exactly this." MVCC or an equivalent gives every writer the property no lock-based design provides at this scale: readers never block writers, writers never block readers, same-row conflicts detected and resolved automatically instead of requiring manual reconciliation.

**Concurrency:** the specific lesson from surveying four production systems, held onto deliberately: a correct database layer doesn't guarantee a correct system above it. LangGraph's own scheduler race, sitting above a correct Postgres layer, is the concrete warning: every point where application code touches shared state outside the database's own transaction boundary is a candidate for exactly the class of lost-update bug tier 1's naive read-modify-write invites, just with higher stakes because more is now depending on it.

**Writers:** a first-class writer-type field (agent, regression, telemetry, human), each carrying its own provenance shape, with the write path branching accordingly. Pure-append fact reports (a regression result, a telemetry event) skip the heavier agent-decision pipeline entirely, both because they don't need it and because forcing them through it would import concurrency risk they never had.

**Invalidation:** real evaporation: a strength value per entry, reinforced on genuine use, decaying continuously as a function of elapsed time rather than requiring a periodic batch sweep to do anything. Decay rate varies by writer type, since a regression result going quiet means something different than an agent-written fact going quiet. A deliberate counter-mechanism against reinforcement-loop entrenchment, so new entries get a real chance rather than starting in a hole popularity-ranked systems are prone to digging for anything unproven.

**Telemetry:** usage-weighted consolidation, the specific, concretely available opportunity the telemetry article identified as real but apparently unbuilt anywhere checked: connecting the reinforcement signal evaporation already tracks to which entries a consolidation pass actually prioritizes, rather than the purely time-triggered consolidation every system surveyed uses instead.

**Models:** no architectural change required. The memory layer stores structured text; text is the one format every model reads identically; routing decisions live entirely in a layer above the store, informed by cost and task difficulty, with zero coupling back into how memory is organized.

This tier is real engineering, achievable by a small team with the discipline to borrow correctly rather than invent, and every individual piece has a working precedent to check against.

## Tier 3: production, hundreds of thousands of writers

This is the tier the series set out to reach, and it's the one where honesty about what's proposed versus proven matters most, because nothing checked anywhere in this research operates at this scale for agent memory specifically.

**Storage:** no single database instance, by necessity — a hundred engineers running a hundred agents apiece is ten thousand writers before CI firing on every commit, nightly regression suites, and sanity bots get counted, and that combination routinely lands in the hundreds of thousands of writes a day, bursty around commit activity and synchronized nightly windows. That's not a volume any single instance was built to serialize, whichever database it is: the limit is finite CPU, connections, and IOPS on one machine, not a specific vendor's tuning default. Sharding or federation, with no single node expected to serialize transactions from every writer in the system, is a different architecture problem than "which database has good locking," inherited from general distributed-systems practice rather than anything specific to agent memory. Naming an actual mechanism rather than just the word "sharding": the writer-typed schema from earlier in this series gives a natural shard key already. Scope or project is the boundary writers actually respect (an agent's writes and a regression suite's writes about the same project land on the same shard; unrelated projects never need a cross-shard transaction), which keeps the common case single-shard even at production scale. What that doesn't solve, and what's still open: a query that needs to reason across scopes. Same honest caveat as the telemetry row below: not a solved problem, just a narrower and more specific one than "figure out sharding" in the abstract.

**Concurrency:** the same transactional discipline as tier 2, replicated across shards, with the same warning about application-layer gaps above the database: now at a scale where an undetected race has far more surface area to hide in before it produces a bug report someone notices.

**Writers:** the full range from the writers-beyond-agents article, at real production volume: regression suites, telemetry pipelines, and human corrections outnumbering agent writes by orders of magnitude, exactly as that article predicted once every automated system in an organization counts as a writer rather than just the ones running a language model.

**Invalidation:** evaporation as designed for tier 2, now load-bearing rather than optional. At this scale, a human-reviewed staleness sweep was never going to keep pace, and the entire point of a decay function computed at read time rather than by a batch process is that it costs nothing extra as the corpus grows, because it was never a separate pass over the data in the first place.

**Telemetry:** the retraining mechanism itself isn't the speculative part: DeepSeek-R1's compiler-graded RL and Cursor's production-outcome-to-weight-update loop are real, shipped, and copyable. What's proposed here is wiring that same shape to a store with an open, uncontrolled set of writers, which is a combination nobody checked in this research has built: every real retraining pipeline found belongs to a product that owns both the signal and the model: Cursor's own users, not a shared knowledge base other systems also write to. That's a narrower, more specific gap than "telemetry-to-retraining is unsolved," and it's the one place in this entire reference architecture that deserves genuine skepticism rather than the confidence the rest of it has earned.

**Modality:** a content-addressed artifact store sitting alongside the text-and-caption pattern every production vendor checked in this series actually uses. Not a claim to have solved multi-modal retrieval, which remains open industry-wide, but a refusal to discard the one thing a future solution would need, the way every system surveyed currently does.

## Knowing when to move, not just where to move to

A reference architecture at three tiers is only useful if there's a real answer to the question every tier boundary eventually raises in practice: how do you know it's time to move up, rather than just assuming bigger is safer and migrating early out of anxiety? The scale-ladder article's own answer generalizes cleanly here: the trigger isn't a volume metric, it's a writer-count metric, and it's checkable the same way this series checks everything: look at what's actually running, don't estimate from a design document.

The tier 1 to tier 2 signal is concrete: the moment a system regularly has more than a handful of independent writers touching the same scope in overlapping windows (not peak-burst independence, sustained, ordinary-day independence), the lock-based approach that was cheap and correct at tier 1 starts costing real wall-clock time in contention, and it's worth checking directly rather than guessing: instrument the lock's wait time, and if writers are routinely queuing rather than acquiring immediately, that's the signal, not a calendar date or a headcount milestone. The tier 2 to tier 3 signal is less about contention and more about a hard ceiling: whatever a single instance's actual connection pool and transaction throughput headroom measures out to, on the hardware actually running it, is a real number to watch directly rather than assume. A system approaching a meaningful fraction of that headroom regularly, on one instance, is a system that needs to plan its sharding story before it arrives at the ceiling under load, not after.

Migrating early, before either signal actually fires, isn't free caution. It's the same tier-mismatch error the second article warned against, just aimed in the opposite direction: importing tier 2's transactional complexity to solve a tier 1 problem costs real engineering time maintaining infrastructure a lock would have handled, and costs something less obvious but just as real: confidence in the system. Every added moving part is one more thing that can silently break in a way a five-line lock function can't. The right migration timing is exactly when the signal fires, checked directly, not before.

## What this series actually is

Return, for a moment, to the premise the opening article checked as rigorously as a public search allows: no system found anywhere combines concurrent multi-agent shared memory, a real telemetry-to-retraining loop, multi-modal memory, and memory-aware model routing. That gap now has a specific shape, tier by tier: the smallest tier resting on a well-established primitive, the middle tier assembled from real, separately-verified precedents, and the largest tier honestly marked as proposed rather than demonstrated.

The next real step isn't another article. It's building tier 2 for real, against an actual multi-writer workload, and finding out which parts of this reference architecture survive contact with writers that don't behave the way a design document assumed they would. A bug found there, specific to this system, is real evidence about this architecture and belongs in an update to the relevant chapter above, not a new chapter of its own, and not a war story about the discovery.
