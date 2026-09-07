---
layout: post
title: "What the Hub Is"
date: 2026-09-07
series_order: 2
tags: [agentco, hub, coordination, multi-agent, open-source]
---

# Your agents are individually excellent and collectively blind

Everyone on your team is running their own AI coding agent. Different tools,
different vendors, no two configured the same, and nobody is giving theirs up.
That last part is not a problem to be solved. It is the actual situation, and any
answer that starts with "first, everyone standardises on one tool" is not an
answer.

Individually they work fine. Together they cannot see each other:

- Two agents edit the same directory and nobody finds out until the merge.
- One builds against a spec that changed last Tuesday.
- An agent hits something it cannot decide, asks a human, and the question goes
  nowhere, because nothing guaranteed it was delivered.

**AgentCo Hub** is a coordination layer for organisations running more than one
agentic harness. It holds the three things nothing else holds.

## One: the claims people and agents make about each other's work

An agent about to work somewhere claims that scope first. Another agent claiming
overlapping scope gets told, with the identity of who holds it. Leases are
**fenced**: every claim carries a monotonic token, so a worker that stalls, wakes
up late and tries to write is refused by the token rather than by luck. That has
been proven across twelve real concurrent processes, not just in a unit test.

Crucially, the Hub is **advisory and never blocking**. It does not sit in your
build. It does not gate your commits. If it is down, your agents keep working and
lose visibility, not throughput. A coordination layer that can halt production is
a coordination layer people route around within a week.

## Two: the pointers you built against, so you are told when they move

An agent that builds against a spec records a pointer and a version token for it.
When that document moves, everything built against the old version is identified.

The Hub **never stores the document**. Only the pointer and the token. This is a
deliberate boundary: a coordination plane that accumulates copies of your
specifications becomes a data-governance conversation, and that conversation kills
adoption faster than any missing feature.

## Three: a router that puts a decision in front of a named human

The failure that costs the most is the quietest. An agent hits a question it
cannot answer, asks, and nothing carries the question anywhere. The work stalls
and looks exactly like an empty queue.

So a human gate names the person who answers it, the wait is on a clock, and a
step that has been ready past its actor's declared cadence with nobody pulling it
becomes an explicit finding that names the actor and the wait. The Hub judges its
own silence first before it reports anyone else's.

## And it is where ASOPs live

The organising idea is the ASOP — the versioned, verified, self-revising procedure
from the [first post in this series]({% post_url 2026-09-07-what-an-asop-is %}). The Hub is the plane that stores them,
versions them, records outcomes per version and per step, and routes their gates.

The pieces that matter in practice:

- **Runs are filed as trees.** One parent, one bead per step, each pinned to the
  version it came from, ordering carried as blockers.
- **Refusals happen before anything is filed.** A retired procedure, missing
  inputs, an unbound role, an unsatisfiable constraint, or a role bound to an
  actor the registry holds no key for — an actor that cannot authenticate can
  never pull, so filing work for it is filing into a void.
- **A run closes when its steps do**, in the same transaction that lands the last
  one. Only `done` counts, so a parked gate holds the run open honestly.
- **Who may verify and who may judge are declared by the operator**, in
  configuration, not claimed in a request payload. Undeclared adjudicators fail
  closed to humans, on the reasoning that what degrades when that fails open is
  the evidence base itself.
- **Bindings come from the caller.** The plane never invents one.

## The participation ladder

You do not have to adopt all of this to get value from any of it. A harness
declares its own level, and each one is useful alone.

| level | what it can do | cost to the owner |
|---|---|---|
| **observer** | reads a managed block spliced into a shared repo file | zero |
| **publisher** | appends a JSON line to an outbox in the repo; a local drainer signs and publishes it | zero |
| **worker** | pulls leased work, executes, reports, attests | one config line |
| **verifier** | declares the verify capability and answers judged gates | deliberate setup |

The bottom two rungs are the interesting design decision. A harness that
configured nothing can still *read*, and a harness that can write one line to a
file can still *publish* — no package installed, no client library, no network
call from the writer. That is the level a bash script or somebody else's agent can
reach without asking permission.

Because every property the contract is built on — pinning, gating, divergence —
requires a write. A layer that only the fully-configured can write to is an
audience, not a coordination layer.

Our own runtime is the reference implementation of the worker rung, and that is
the subject of the next post.

## What it is not

It is not agent memory. What an agent recalls is its own harness's business, and
that layer is already well served. The Hub coordinates what happens *between*
agents and the people who own them.

It is not a personal-AI framework. Those run one principal's assistant superbly.
This is the layer they plug into the moment an organisation runs more than one.

It is not a workflow engine that owns your pipeline. It holds claims, pointers,
leases, procedures and decisions. Your harness still runs your work.

## Honest about state

The repository is public, Apache-2.0, and its roadmap marks nothing as done
because it was designed. Its `known-issues.md` lists reproduced defects that are
deliberately not yet fixed, most of them with a failing test already written and
marked strict-expected-failure, so that fixing one turns the suite red until the
marker is deleted in the same commit.

There is a leak guard in CI whose whole job is to keep company-specific material
out of a public repository, and storage runs on JSONL by default, SQLite or
Postgres by configuration, conformance-tested identically against all three.

Current suite: 1,280 tests passing, plus 18 expected failures that each name an
open defect, plus 106 for the contract package on its own.

Next: **the Harness** — the runtime that actually executes the work, and why it is
a separate product.
