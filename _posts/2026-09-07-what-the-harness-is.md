---
layout: post
title: "What the Harness Is"
date: 2026-09-07
series_order: 3
tags: [agentco, runtime, beads, agents, open-source]
---

# The Harness: something has to actually do the work

The Hub coordinates. It holds claims, pointers, procedures and decisions, and it
is deliberately advisory. It never executes anything.

Something has to execute. That is the **Harness**: a standalone runtime for
agentic work. A local work store, a heartbeat cycle that decomposes, triages,
dispatches and verifies, human executors as first-class assignees, and a doctor
that tells you what is broken before the first cycle silently does nothing.

It runs alone. A solo operator uses it with no server at all, and nothing in it
knows about any particular company, calendar, mailbox or ticket system.

## Every unit of work is a bead

A bead is one line of JSON. The store is append-only and quarantine-preserving: a
corrupt line is reported by the doctor, never executed and never dropped.
Watermarks and heartbeats move only on genuine completion, which is the difference
between a system that reports progress and a system that makes it.

Beads carry their own definition of done. A bead with a gate does not reach `done`
by being marked done; it reaches a verify state and stays there until the gate is
answered. A human-assigned bead never enters the agent dispatch loop at all and
shows up in a queue that answers one question: what depends on you.

The whole loop is one command you can run by hand or leave running forever:

```sh
agentic-co init                 # config + store in the current directory
agentic-co tasks create "..."   # every unit of work is a bead
agentic-co cycle                # recurring → triage → dispatch → verify, once
agentic-co doctor               # preflight, classified by consequence
agentic-co sop run ID --bind implementer=claude
```

## The doctor is the most underrated part

Most runtime failures in this class of system are not crashes. They are silent
no-ops: a queue full of work assigned to a name nothing can dispatch, a scheduler
pointed at a directory that moved, a config key that got clobbered.

So the doctor's checks are classified by consequence and its exit code is the
worst class present, never a count. It refuses to let an aggregate mask a broken
check.

This is not theoretical. One config key going missing once cost fifty failed
work items and roughly ninety-five duplicate incident reports before anyone
noticed, because every failure spawned an investigation and every investigation
was the same investigation. The runtime now detects the shape of that defect on
the execution path itself and stalls the affected work rather than burning it,
because the fix is in configuration and the next cycle will pick the work up for
free.

## Extension seams instead of your pipelines

The Harness descends from a private monolith that had its owner's personal
pipelines wired directly into the cycle. Extracting it meant deciding what a
runtime *is* versus what one operator happened to need.

What is left is three registries and a backend seam. You register a handler for a
kind of work, a hook that runs after a bead lands, or a source that produces
events. And a **backend** is what an ASOP role binding names: bind `implementer`
to a name and that name must be something the runtime can execute. Registering one
declares how it runs and what data classification ceiling it runs under, and from
then on that name is dispatchable, known to the doctor, and subject to the egress
gate under the route it declared.

That last part is the useful bit. It means "this data may not go to that vendor"
is enforced at the moment of dispatch, which is where it becomes true, and a
refusal blocks the work rather than failing it, because the work is fine and only
the routing is wrong.

## Connecting to the Hub is optional and off by default

Set a plane URL and the Harness becomes a full worker on the plane: it pulls leased
work, mirrors each item as a local bead carrying the procedure's own words and the
gate that will judge it, executes, reports the outcome and attests the gate it
actually ran.

It never files work on the plane. Publishing is off, deliberately: the plane owns
the queue. And a plane that is unreachable is a plane that is advisory, so a
failed report can be retried by a later `agentic-co hub sync` sweep and the bead
is done locally regardless.

We found a real defect building this, and it is a good illustration of why you
run the thing rather than reason about it.

The runtime mirrored each pulled item under the name it uses to identify itself on
the plane. That reads fine until you notice those are two different concepts. The
name you are known by *there* is an identity. The thing that runs work *here* is a
backend. They are almost never the same word, so every pulled item failed the next
cycle with "unknown agent" and spawned an incident report apiece.

The scripted end-to-end test never saw it, because that test completed the items
itself and never dispatched. Only running the real dispatch path found it. The fix
separates the two names, refuses a configured plane that names no local runner
before the first pull rather than one cycle after it, and hands the executor the
step's own words plus the directory its gate runs in — because a bare step name
like "write-tests" is not something a model can act on.

## What "working" means here

The end-to-end driver stands up a plane, authors a procedure, files one run bound
to three different participants, and drives it to completion while checking every
claim the contract makes. It runs in five modes, and all five are green:

| mode | what it proves | result |
|---|---|---|
| deterministic | the pipeline, with no model in the loop | 22/22 |
| real agents over the tool surface | two vendors' CLIs as independent participants | 22/22 |
| live | the runtime dispatches a real model for its own steps | 22/22 |
| judged gate | a declared verifier answers, and three ways of usurping that are refused | 25/25 |
| all of it at once | every participant a real agent, live dispatch, judged gate | 25/25 |

Two details worth stating, because they are the difference between a demo and a
test.

In live mode the prepared answer is **not** written to the target repository. The
first live run left it there and the implementing model read it, which proved
dispatch and proved nothing about solving anything. Unaided, the model wrote its
own implementation and the gate passed on an independent re-run.

And the judged-gate mode checks the rails before it checks the verdict: an
undeclared actor claiming the verify capability is refused, the party that
executed the step is refused, and even the declared verifier is refused if it does
not claim the capability. Three refusals, then one verdict. A gate you can only
observe passing is not a gate.

## One standard, two implementations

The Hub and the Harness are separate repositories with separate suites, and they
share exactly one thing: [the ASOP contract](https://github.com/mabidoli/asop),
which is a third repository owned by neither of them. The Harness depends on it
without depending on the Hub — at the import level, and now at the packaging
level too, which is a distinction worth being honest about. For a while the
runtime installed the contract from the Hub's repository URL, so the sentence was
true of the code and false of the install. Moving the specification out is what
made both true at once.

That is the whole architecture. A standard in the middle, a coordination plane on
one side, an execution runtime on the other, and a deliberate refusal to make any
of the three require another.

Both are public and Apache-2.0. If you are running more than one agent and you
have started to feel the coordination cost, start with the contract and see
whether the three properties describe a problem you have.
