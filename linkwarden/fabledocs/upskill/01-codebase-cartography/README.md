# 01 — Codebase Cartography

Before you can change a system safely you must know where things live, who owns what, and how the main flows move. This module builds that map.

Read in order:

1. [01-system-map.md](01-system-map.md) — the monorepo's shape, ownership map, public vs private surfaces.
2. [02-file-reading-order.md](02-file-reading-order.md) — 28 files in the order that builds understanding fastest, with junior/mid/senior paths.
3. [03-domain-glossary.md](03-domain-glossary.md) — the product nouns (collection, preservation, monolith…) and where each lives in code.
4. [04-runtime-and-tooling-map.md](04-runtime-and-tooling-map.md) — yarn 4 workspaces, scripts, env vars, runtime boundaries.
5. [05-key-flows.md](05-key-flows.md) — **the module's core**: seven end-to-end traces the rest of the curriculum references constantly.

Transferable skill: cartography is a repeatable procedure, not repo trivia. On any new codebase: manifests → schema → one route end-to-end → the background/async story → tests/CI. That's exactly the order these five files follow; practice narrating it, because "how do you approach an unfamiliar codebase?" is a standard interview question ([08-interview-prep/06](../08-interview-prep/06-behavioral-star-stories.md), Story 1).
