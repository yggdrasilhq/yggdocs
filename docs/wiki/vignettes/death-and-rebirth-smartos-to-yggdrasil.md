# Vignette: Death and Rebirth (SmartOS -> Yggdrasil)

Every serious infrastructure has a grief chapter.

For us, that chapter started when SmartOS stopped fitting the work we wanted to do next.
GPU pathways were painful, some operational moves felt boxed in, and each workaround added cognitive debt.

At first, the instinct was to patch harder.
Then came the real question: *Are we still serving the system, or is the system serving us?*

## The turning point

A journal entry in late 2023 captured the pivot:

- SmartOS gave stability and discipline.
- But the next phase needed different tradeoffs.
- Debian live-build + ZFS + LXC gave us more room to shape behavior directly.

In plain language: the old system was respected, then retired.

## What we kept from SmartOS

We did not discard its philosophy.
We carried forward the important parts:

- infrastructure as reproducible procedure
- ZFS-first thinking
- operational conservatism during migration
- journaling every non-trivial move

Yggdrasil is not a rejection of SmartOS.
It is what happened after learning deeply from it.

## What changed in Yggdrasil

- Build process is explicit and scriptable (`mkconfig.sh` + config profiles).
- Server and KDE images are both first-class artifacts.
- Smoke tests are a release gate, not a nice-to-have.
- Configuration can be generalized for public users and specialized locally.

## A line from the field (translated spirit)

The journal had many bilingual notes like:

- `ami ei korlam` -> “I did this step.”
- `bujhlam` -> “I realized.”

That tone matters.
It means the docs are not written from theory.
They are written from lived operations.

## Why this story is in the manual

Because readers are not only cloning a repository.
They are deciding whether to trust a migration path.

This vignette says: you can evolve systems without erasing their history.
