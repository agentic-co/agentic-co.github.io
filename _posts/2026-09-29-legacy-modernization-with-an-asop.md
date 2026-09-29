---
layout: post
title: "Legacy Modernization with an ASOP"
date: 2026-09-29
series_order: 5
description: "Two slices of a real legacy .NET Framework app, ported to .NET 10 under a versioned, gated procedure. It found a real legacy bug — and caught two of its own fake-green runs along the way."
tags: [agentco, asop, legacy-modernization, strangler-fig, verification, open-source]
image: /assets/legacy-modernization-hero.png
---

# Liveness is not correctness, and a green build knows nothing about either

![Liveness ≠ correctness — the six-step strangler-slice-a-feature ASOP, with step 3 (characterize-legacy) failing on a real bug and step 4 (dotnet10-port) passing by reproducing it on purpose](/assets/legacy-modernization-hero.png)

Point an agent at a legacy codebase and tell it to modernize a slice of it,
and it will produce something that runs. That was never the hard part. The
hard part is knowing whether the thing that runs still does what the old
thing did — and an agent that wants to look done has every incentive to let
you believe "it builds" and "it's correct" are the same claim.

They aren't. A build passing is liveness. Behavior preserved is correctness.
So we ran a small, honest experiment against exactly that gap: fork a real
legacy .NET Framework app, write a procedure for modernizing it one slice at
a time, gate every step behind something a machine re-runs, and see what
breaks. Not "can an agent write .NET 10 code" — every model can do that now.
Whether the *procedure* around the agent catches the agent when it's wrong.

The repo is public: [github.com/mabidoli/dasblog-asop-poc](https://github.com/mabidoli/dasblog-asop-poc),
a fork of Scott Hanselman's [DasBlog](https://github.com/shanselman/dasblog).
Two slices are done, both merged. Here's what happened, including the parts
that didn't go well.

## The procedure: an ASOP, not a prompt

An ASOP — Agentic Standard Operating Procedure — is a versioned, ordered
sequence of steps where each step names a gate that must pass before the
next one counts as reachable, and where divergences get adjudicated and
feed the *next* version. ("What an ASOP Is" covers the general case; this
is what happens when you point one at a codebase instead of a feature
request.)

Modernization is a better fit for this than most agentic work, because most
agentic work needs a judge to decide if the output is any good. Modernization
doesn't — the legacy build is sitting right there, an actual oracle. "Does
the new code produce the same output as the old code, byte for byte" isn't
an opinion. It's a diff.

The procedure — "strangler-slice-a-feature" — has six steps: **map-the-slice**
(entry points and dependencies, human sign-off); **extract-business-rules**
(every rule cites `file:line` and gets a test, checked by script);
**characterize-the-legacy-behaviour** (write those tests against the
*legacy* build, gate: legacy CI green); **implement-on-.NET-10** (port the
slice, gate: the same tests pass on the new build plus a golden-file diff);
**facade-and-route-traffic** (Strangler Fig's actual move — both sides agree
against the same golden files); **review-and-merge** (a named human
approves the PR).

The interesting decision isn't the step count. It's step 3 existing
*before* step 4. Characterizing the legacy behavior first, as its own
gated step, is what turns "the agent migrated a blog" into "the agent
proved what the blog does, then proved the port matches it."

## Slice 1: RSS feed generation, and three incidents

First slice: DasBlog's RSS 2.0 generation — close to pure logic, posts in,
XML out, which makes the oracle a golden-file diff. Sixteen business rules
extracted and ported, sixteen more explicitly deferred rather than silently
skipped. [PR #1](https://github.com/mabidoli/dasblog-asop-poc/pull/1),
merged. Getting the legacy side to build in CI at all cost 8 iterations
before any ASOP step started — that's infrastructure, not the procedure.
The procedure's own steps cost more, and produced three incidents worth
naming.

**A gate-proof that proved nothing.** To confirm the legacy CI gate
actually fires red, the agent broke a test's assertion on purpose and pushed it.
CI came back green. The test it had "broken" wasn't in the project's
`.csproj` compile list at all — dead code in the tree. The gate wasn't
broken; the proof was. A green run isn't evidence of anything unless you've
independently confirmed the thing you changed was in the path that ran.

**Four wrong guesses before the real one.** A new test project reference
kept failing with `CS0234: type or namespace 'Core' does not exist`,
surviving three plausible fixes — correct GUIDs, explicit build
dependencies, a different reference mechanism entirely. The real cause,
found only in a *detailed*-verbosity MSBuild log on the fourth attempt, was
`MSB3274`: the test project still targeted .NET Framework 3.5 against
projects already retargeted to 4.8. Every theory before that was genuinely
plausible — which is exactly why "plausible" isn't a stopping condition.

**A serializer quirk no amount of trying fixed.** The RSS XML matched
byte-for-byte except two auto-generated namespace declarations came out in
the opposite order from the legacy build. Every insertion order the agent tried on
.NET 10 produced the identical (wrong) result — proof the ordering is
hardcoded in .NET 10's `XmlSerializer`, not caller-controlled. The fix
wasn't a smarter order; it was normalizing the difference after the fact,
with the reasoning written down next to the code.

Slice 1 finished green — legacy suite at 48 tests, the .NET 10 port passing
on the first CI attempt after iterating locally. Two bad divergences went
into `ADJUDICATION.md`, the sharpest being that the agent executed steps 2
through 6 while step 1's human gate (my review) was still open. That became v2's input.

## v2, self-revision, and the Akismet bug

v1 is never edited — it's versioned, not fixed in place. `v2.yaml` applies
exactly the three fixes v1's adjudication proposed, no more:
**park-and-continue** for an unanswered human gate (downstream work proceeds
but stays marked PROVISIONAL until the gate closes, matching the harness's
own `PENDING_APPROVAL` semantics); a **gate-proof/test-reliance rule**
(confirm a test is actually compiled in, and the run's test count reflects
it, before trusting green); and **iterate locally before CI** where the
runtime allows it — v1's own numbers justified this (14 CI round-trips on
the legacy side versus 1 on .NET 10, once a local SDK was installed).

Second slice: the Akismet spam-check request mapping, chosen over
trackback/pingback parsing after reading both candidates' actual code —
trackback makes a live outbound HTTP call, pingback needs a real ASP.NET
hosting environment; Akismet mapping needed neither.
[PR #2](https://github.com/mabidoli/dasblog-asop-poc/pull/2), merged.

This is where the procedure earned its keep. The very first golden-file
fixture didn't produce output — it threw a `NullReferenceException`. Line
63 of `AkismetSpamBlockingService.cs`:

```csharp
feedback.TargetEntryId != null & feedback.TargetEntryId.Trim().Length > 0
```

A bitwise `&` where the code means `&&`. Both sides evaluate regardless of
the left one's result, so a null `TargetEntryId` gets `.Trim()`'d anyway and
the request crashes instead of treating a blank field as blank. Nobody
finds that reading the `if` statement — the line looks fine at a glance.
You find it by running the legacy code against a null and watching it
explode, which is the entire argument for characterization testing over
"read the code and vibe-check it." The port reproduces the crash on
purpose, as a named test (RULE-spam-10): Strangler Fig's discipline is
preserving behavior first, and fixing the bug is separate work for later.

A second incident, same slice: the workflow that captures golden files from
the legacy build reported success on every step — including "upload
artifact" — while a directory-depth bug meant it wrote its output one
folder too high and never found it again. The file-existence check didn't
catch it, because *older* golden files from slice 1 already satisfied the
glob. Same family of mistake as the dead test, one level up: a green
*workflow* isn't evidence it did the specific thing it claims, any more
than a green test is. You have to check the artifact, not the exit code.

## v1 vs. v2, confounders included

| | Slice | CI iterations (steps 3 / 4) | Bad divergences | Good divergences | Bugs found |
|---|---|---|---|---|---|
| v1 | feed | 14 / 1 | 2 | 1 | 1 (a dead `/atom.ashx` rewrite, found while mapping) |
| v2 | spam | 4 / 1 | 1 (new) | 3 | 1 (real, reproduced faithfully) |

It's tempting to read 14-versus-4 as "v2 is 3.5x faster." That's not good
evidence, and I'd rather say so than let the table imply it: v2 ran a
*different* slice, and v1's 14 include building the legacy CI oracle from
scratch, one-time cost v2 didn't repeat. What the comparison actually
supports is narrower: v2 made zero repeats of v1's exact two mistakes. The
gate-proof rule held. Park-and-continue worked disclosed instead of silent.
Smaller claim, but the evidence backs it.

## What didn't work, and what's next

**No independent adjudicator** — every divergence above was self-adjudicated,
the executing agent judging its own run. Disclosed as a limitation, not a result I'm
proud of. **"Green means nothing" recurred one level up the stack than
expected** — first a dead test, then a whole CI workflow — so v3's proposal
generalizes the rule: a workflow claiming to have produced an artifact
should assert something concrete about it (a file count, a fresh
modification time), not just that a glob matched. Not written or run yet.
**A third slice** would turn two data points into a trend — not attempted.
**Packaging the harness for Windows** surfaced that its CLI can't even be
imported on native Windows Python — several modules import the POSIX-only
`fcntl` unconditionally at load time. WSL2 works around it; native Windows
doesn't, yet. And **most of the actual feature is still deferred** — slice 1
is "plain entry" RSS generation only, sixteen more rules explicitly
postponed. Small enough to finish was the point of starting here.

## Why this, and not "the agent migrated a blog"

Anyone can point an agent at legacy code and get something that runs. The
narrower, more falsifiable claim is that the *procedure* catches the agent,
not just the codebase. It caught a fake gate-proof. It caught a workflow
lying about its own output, twice, two different ways. It caught a real bug
sitting in shipped code, unnoticed, because nobody had run it against
the one input that mattered. None of that was the agent being smart. It was
the gates doing the one job gates have — refusing to call something done
until a machine re-checks it.

If you're running an agent against your own legacy estate: how would you
know, today, whether the new version does what the old one did, and not
just what it was supposed to do?
