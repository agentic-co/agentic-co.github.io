---
layout: post
title: "What an ASOP Is"
date: 2026-09-07
series_order: 1
tags: [agentco, asop, agents, verification, open-source]
---

# An SOP tells an agent what to do. An ASOP can prove it was done.

Every team running AI agents ends up writing procedures for them. You call it a
prompt, a playbook, a `CLAUDE.md`, a runbook. It says: read the requirement,
write the tests, implement, run the tests, validate.

Then you ship it and discover the three things it cannot do.

It cannot tell you whether the agent actually did those things or just said it
did. It cannot tell you whether version four of your procedure works better than
version three, because you never versioned it and you have no outcomes attached
to anything. And when a run goes sideways, the lesson lives in a Slack thread
and dies there.

An **ASOP** — Agentic Standard Operating Procedure — is what you get when you fix
all three. It is a versioned, ordered sequence of steps that accomplish one type
of task. Each step says what must be done, what must be true before starting, how
the result is proven, where the result is written, and which role does it.

A procedure is an ASOP when it has three properties. Missing one, it is
documentation.

## Versioned: outcomes attach to versions, not to vibes

Every revision is a distinct, immutable version. A run pins the procedure and
version it came from, and each step of that run pins the exact step it came from.
Outcomes are recorded against that triple.

So "did the v3 rewrite of step 2 help?" is answered by counting rows, not by
recollection. You get a success rate per version and per step.

There is a corollary that matters more than it looks. The template and the
instance are different things. The ASOP never enters a work queue; runs do. A
later version does not reach back into a run already in flight. That run finishes
under the version it started under, and it can ask whether its procedure has moved
underneath it.

This is the difference between a procedure and a prompt. A prompt is whatever was
in the file the day you ran it. Nobody can tell you what it said last Tuesday.

## Verified: every step carries its own gate, written by the author

Each step carries a gate, and the gate is part of the step — written by whoever
authored the version, immutable to whoever executes it or files the work.

There are three kinds, and their trust models are genuinely different:

| kind | proof | trust model |
|---|---|---|
| `deterministic` | a command exits zero, re-run fresh by the completing process | machine-checkable, but the executor attests its own result. A trust floor, not a proof. |
| `judged` | a fixed rubric, answered by a route other than the executor | the operator declares who may judge. Claiming the capability is not the authority. |
| `human` | a named person signs off | one named person's judgement, and nobody else's |

The subtle design decision is *where* the gate is authored. In the obvious design,
whoever files the work supplies the gate. That is fine until you notice that in
most organisations the filer is on the executor's side. Which means the executor's
side wrote its own definition of done. That is the exact failure the whole idea
exists to prevent, so the gate belongs to the procedure and to nobody else.

The second decision is separation of duties, and it is enforced at filing rather
than at completion. A procedure can declare that its validator must not be its
implementer. Bind both roles to the same agent and the run is **refused before
anything is filed** — not warned about, not flagged in a report afterwards. A
constraint that fails at the end is a report; a constraint that fails at the start
is a rail.

The plane that stores an ASOP never runs the check. It validates the shape of the
evidence and stores the claim. Someone else ran the command; the record says who,
what they ran, what it exited, and where. That distinction is what makes an
attestation disputable later, which is the only thing that makes it worth having.

## Self-revising: the divergence is the input to the next version

The third property is the one people skip, and it is the one that compounds.

When execution departs from the procedure, the departure gets adjudicated — by a
party other than the executor, always. A good adjudication says the procedure was
wrong and the executor was right, and it becomes a **proposal** on that specific
step of the next draft version. A bad one becomes a documented common mistake on
that step, capped so the list stays readable. A step that diverged twice in one
pass earns a structural proposal on the sequence itself.

Nothing here activates on its own. It drafts. A human decides.

Here is the flavour of it, from a run we did this week. The procedure said the
gate for step three was `pytest`. The repository needed `python3 -m pytest`. The
step succeeded anyway because the executor adapted. Under the old model that
adaptation is invisible — the step is green, everyone moves on, and the next
person hits the same wall. Under an ASOP the divergence was adjudicated as good,
and the next draft version carried the corrected check on step three, with the
evidence and the adjudicator's name attached.

That is a procedure that gets better because it ran. Not because someone
remembered to write a postmortem.

## What it looks like

```
ASOP  feature-dev  v3
  step 1  validate-requirements    role: analyst
  step 2  write-tests              role: implementer
  step 3  implement                role: implementer
  step 4  run-tests                role: implementer     gate: deterministic
  step 5  validate                 role: validator       gate: judged
                                   constraint: validator ≠ implementer
```

Filing work from that produces a tree: one parent, one bead per step, each pinned
to the version it came from, sequencing carried as blockers. Any harness can
execute it. The roles are bound at filing time by the caller, so the same
procedure can run with three different vendors' agents in three different steps.

We ran exactly that. One plane, one procedure, three participants: Claude Code as
the analyst, our own runtime as the implementer, and Google's agent CLI as the
validator. Three separate cognitive lineages, one procedure, one set of outcomes
counted against one artefact.

It is a standard in the literal sense. The same procedure, the same version, the
same gates, run by different harnesses in different organisations, producing
outcomes that can be compared *because they are counted against the same thing*.

## Why this is the interesting layer

The industry is spending its attention on making individual agents smarter. That
work is real and it is going well. But an organisation running five agents does
not have a smartness problem. It has a **coordination and evidence** problem: no
shared definition of done, no way to compare two approaches, no way to tell an
adaptation from a shortcut, and no mechanism that turns yesterday's failure into
tomorrow's procedure.

An ASOP is not a framework you adopt or a vendor you pick. It is a contract: a
gate schema, a record shape, and a refusal vocabulary. The specification and its
reference implementation live in [their own repository](https://github.com/mabidoli/asop)
and install as `asop` — standard library only, depending on no plane, no runtime
and no vendor.

It does name the two implementations whose gate schemas it reconciles, and
deliberately so. A standard that hides which systems it was derived from is
asking to be trusted about the one thing it can actually show you.

The next two posts cover the two halves that implement it: **the Hub**, which
stores procedures, versions them, records outcomes and routes gates, and **the
Harness**, which executes the work.
