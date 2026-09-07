---
layout: default
title: About
permalink: /about/
---

# About

AgentCo is two open-source products either side of one contract.

- **[agentic-co-hub](https://github.com/agentic-co/agentic-co-hub)** — the
  coordination plane: scope claims, snapshot pointers, fenced leases, ASOP
  storage, gate routing, and a human decision router. Advisory, never blocking.
- **[agentic-co-harness](https://github.com/agentic-co/agentic-co-harness)** —
  the execution runtime: beads, cycles, executors, schedules, and a doctor that
  reports what is broken before the first cycle silently does nothing.

Both are Apache-2.0. The Harness depends on the ASOP contract package without
depending on the Hub; nothing in the Harness imports the plane.
