---
layout: post
title: "What Conformance Cost"
date: 2026-09-10
series_order: 4
description: "We made a runtime obey the contract it was co-designed with. Six divergences, two holes, and the one that mattered most was the one that returned success."
tags: [agentco, asop, conformance, verification, open-source]
---

# We made a runtime obey the contract it was co-designed with. It did not go quietly.

The previous three posts described a contract, a coordination plane, and an
execution runtime. The plane and the runtime were built alongside each other by
the same hand, against the same spec, in the same month. If any two
implementations of a contract were going to agree, it was these two.

This week we stopped writing the lifecycle twice and made the runtime use the
plane's. Here is the bill.

## Two implementations is not two versions of one thing

The first finding arrived before a line of code changed. The runtime's store and
the plane's speak different languages, not dialects of one.

| runtime | plane |
|---|---|
| `complete(id, result)` | `report_result(id, attempt, status, attestation)` |
| `update(id, **fields)` | `annotate`, `declare`, `retire` — narrow, purposeful writes |
| `approve_verify` / `reject_verify` | `attest`, `resolve_by_default` |

The runtime's model is direct mutation by a trusted in-process caller. The
plane's is a fenced protocol for participants it does not trust: claim with a
lease, report against an attempt number, attest separately, and never let the
executor write its own status.

The plane's shape is stricter on purpose. It is the shape that made the
anti-usurpation rails possible in the first place. Which meant the substitution
was never going to be a drop-in, and there were three honest ways forward:
adapt, widen the plane, or narrow the runtime. Only the third ends with one
shape instead of two plus a bridge. We took it.

## The six divergences

Measured against a live plane — probes, not docstrings. Five refuse loudly. One
does not.

1. **`blocked` is not reportable.** The protocol takes terminal outcomes only.
   The runtime wrote `blocked` in eight places to mean *could not be dispatched*.
2. **Six of those eight wrote it before claiming**, where the protocol refuses
   any report for want of a lease.
3. **A dependency edit was accepted into metadata and silently dropped.** More
   on this one below.
4. **Status vocabularies differ** — `pending_approval` and `skipped` exist on one
   side only.
5. **Gate validation is stricter on the plane.** It requires clock fields the
   runtime relaxes, so gates the runtime files today are refused *at filing*.
   Substitution breaks before completion, not at it.
6. **`create()` signatures differ.** The plane has no `description` field at all.

Five of these announce themselves. You run into them, you read the refusal, you
fix the call site. They cost time and nothing else.

## The one that returned success

Ask the plane to change a work item's dependencies through `annotate` and it
accepts. It writes a key of that name into metadata. It leaves the real
dependency list exactly as it was. It returns the item, looking updated. Nothing
refuses, nothing warns, and `unmet_blockers` still returns the old list.

A translation layer written by someone reasonable, in an afternoon, would have
mapped `update(blocked_by=…)` onto `annotate` and shipped. Every test would have
passed, because the call returned an item and the item had the key on it.

This is the argument against writing an SDK before the contract is finished,
made concrete. A client library is a place to put exactly this kind of decision,
and the decision is invisible in the diff that makes it.

So our own `annotate` refuses the key outright rather than reproducing the
plane's behaviour faithfully. Fidelity to a trap is not fidelity.

## Two holes, found by conforming

Neither was reachable by a test that asserts success.

**A verb reached done past a pinned gate.** `retire` exists to close work items
that became moot — routing vehicles nobody is working. It checks for a live
lease and for settled states. It never looks at the gate. An unclaimed item
carrying a valid `judged` gate retired straight to `done` with no attestation,
while the honest path on that same gate — claim, report — correctly parked at
`awaiting_verify`.

It was not exploitable, and the reason is the interesting part: its only caller
creates the items it retires, under a comment reading *"NO `verify=` here,
ever."* The invariant was held by a comment at one call site rather than by the
verb. That holds right up until a caller with gated items reaches for
retirement, which is exactly what narrowing a runtime onto the protocol does.

**A report was accepted with no lease.** An unclaimed item sits at attempt zero,
so a report at attempt zero satisfied the fence and completed. On a gated item it
landed `awaiting_verify` with an executor of `None` — and the separation check on
a judged gate compares the verifier against the executor, so it compared against
nothing.

## One pattern, three times

Three separate conversions turned out to be the same move, and naming it made
the third one quick.

Each began with the runtime writing a **status** in order to change what a queue
*shows*. Undispatchable work became `blocked`. A health-check failure that the
next run cleared became `done`. Both are a view being encoded as an outcome.

The protocol has no status for either, because a status is an outcome and these
are not outcomes. A sample that failed at 14:00 and was cleared at 15:00 stopped
being *actionable*; it did not stop being *true*. Rewriting it to `done` buys a
quiet queue at the price of a store that contradicts its own record.

So each becomes a **record plus a filter**: the fact goes in metadata, the queue
filters on it, and the outcome stays true.

The cost is honest and worth stating. "Why is this not running" is now answered
by reading metadata rather than by reading status. That is a real loss of
legibility, and it is the price of a protocol that has six states on purpose.

## Copy the reason, not the mechanism

The most useful mistake of the week.

The plane requires a lease before it will accept a report, for a stated reason: a
completion with no executor is one the separation check can never see. We copied
the rule across as written — *no lease, no report* — and the suite stayed green
at 1,285 tests.

It would also have made real, pending work impossible to complete. Some work is
assigned to a **person**, and people never claim: an assigned human bead never
enters dispatch, so no node ever leases it. Worse, such a bead cannot be made to
claim, because the capability gate matches the item's requirements against a
*node's* manifest, and a person is not a node.

The rule we needed was the one the plane's rule exists to *produce*: a completion
the separation check can see. The plane says "lease" because a lease is the only
way it learns who executed. The runtime has a second way the plane has no concept
of — an executor recorded at assignment.

Where your implementation has a concept the contract lacks, copying the mechanism
is subtly wrong in a way your tests will not catch.

## What this says about the standard

We had three independent model families review the idea of shipping an SDK for
building your own runtime against this contract. They converged, and they were
right.

Conformance being hard is an argument for a conformance suite, never for a
wrapper. The hardness here is **ambiguity, not boilerplate** — and a client
library launders ambiguity into API decisions its author makes unilaterally,
which then fossilize as de facto spec nobody agreed to.

The number that settles it: the contract carries **109 normative assertions**,
and **ten** are cited by a conformance vector. The suite's own README says a
pattern-matcher could still pass. The whole middle of the standard — records,
steps, revision policy, role separation, protected tags — has no vector at all.

So the six divergences are becoming spec changes rather than patches: a closed
status vocabulary, claim-before-report as a MUST with a refusal code, a policy
for unknown fields, a gate filing schema, a rule that no verb reaches done past
a pinned gate, and terminal transitions attributed to a lease.

That last one is the `retire` hole, and framing it as a spec change rather than a
bug is the point. A refusal code that exists only in one implementation is the
same shape as the clock fields and the silent drop: a rule living in one
codebase's head. Nobody else inherits it.

## Where the two implementations now disagree, on purpose

The runtime authenticates verifiers against a declared registry, and fails
**closed**: an unset registry authenticates nobody. The spec says so twice.

The plane fails **open**, deliberately, with a real argument — a registry where
nobody may verify does not become safer, it resolves every judged gate on the
clock, which is work approved on a timer.

We implemented the spec on the runtime side and wrote both positions down next to
each other. That turns a latent contradiction into a dated one, which is the most
useful thing you can do with a disagreement you are not yet entitled to settle.
It belongs in the standard, not in whichever file gets edited last.

---

*The contract, the plane and the runtime are all open source. The conformance
suite is where the next work goes — and if you are building a runtime of your
own, the ten-of-109 number is the honest answer to "is this ready".*
