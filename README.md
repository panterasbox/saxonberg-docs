# Saxonberg — public documentation

**Generated. Do not edit here.** This repository is a curated, read-only
mirror of the documentation in
[gitlab.com/panterasbox/saxonberg](https://gitlab.com/panterasbox/saxonberg),
which is the canonical source and where changes are made.

Saxonberg is an immersive multiplayer role-playing educational platform —
a simulation substrate with real units and a working polity on top. See
[docs/vision.md](docs/vision.md), and [INDEX.md](INDEX.md) for the map.

## What is here, and what is not

It exists to give language models a corpus in which every sentence is
meant to be **current**. So it carries the authoritative half of the
documentation and deliberately omits the deliberative half:

| | |
|---|---|
| **published** | subsystem references (the source of truth for shipped behaviour) + the standing top-level docs |
| **withheld** | design slates, video scripts, deployment/process docs, and the source tree |

The omission is not secrecy — the canonical repository is public, and the
withheld material is all there. It is that a *slate* records proposals
including the ones that were argued down, so a retrieval system reading
one cannot tell a live decision from a discarded one. Subsystem docs
carry no such ambiguity.

## Caveats worth stating

- **Designed is not built.** A subsystem doc describes what exists; the
  roadmap describes what does not yet. If an answer matters to you,
  check [docs/roadmap.md](docs/roadmap.md) and each doc's own
  *"What's NOT in this build"* section.
- **This is a snapshot**, republished as one squashed commit. The
  canonical repository holds the history.
