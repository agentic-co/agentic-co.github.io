---
layout: default
title: About
permalink: /about/
---

# About

**[ASOP](https://github.com/mabidoli/asop)** is a procedure standard for agentic
work: versioned, verified, self-revising. It is a specification plus a
dependency-free reference package (`pip install asop-spec`), owned by no product.

**Agentic Co** builds two open-source implementations of it.

- **[agentic-co-hub](https://github.com/agentic-co/agentic-co-hub)** — the
  coordination plane: scope claims, snapshot pointers, fenced leases, ASOP
  storage, gate routing, and a human decision router. Advisory, never blocking.
- **[agentic-co-harness](https://github.com/agentic-co/agentic-co-harness)** —
  the execution runtime: beads, cycles, executors, schedules, and a doctor that
  reports what is broken before the first cycle silently does nothing.

All three are Apache-2.0. The Harness depends on the contract without depending
on the Hub; nothing in the Harness imports the plane; and the contract imports
neither.

ASOP is written and maintained by
[Marcelo Bidoli Fernandes](https://github.com/mabidoli).
