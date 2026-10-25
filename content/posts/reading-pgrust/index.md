+++
title = "Reading pgrust, part 0: the map"
date = 2026-10-25
description = "Why I'm reading pgrust, and how its source tree maps onto Postgres."
# Drafts only show with `zola serve --drafts`. Remove this line to publish.
draft = true

[taxonomies]
tags = ["postgresql", "rust"]
series = ["Reading pgrust"]

[extra]
part = 0
+++

<!-- DRAFT: replace this outline with your own writing. -->

Why I'm reading pgrust, and what I want to learn from it.

<!-- more -->

## How the tree is laid out

pgrust mirrors the Postgres source layout. The original C code is vendored in
{{<src path="crates/postgres-18.6-reference" />}}, so every Rust crate has a C counterpart
to compare against. Storage, for example, lives in {{<src path="crates/backend/storage" />}}.

## The path of a query

The plan for this series: follow one query from the wire protocol down to the disk.

1. Wire protocol and connection handling
2. Parser
3. Planner
4. Gravity, the vectorized JIT executor
5. Buffer manager and `smgr`
6. Files on disk

## Code excerpts

Quote short excerpts (pgrust is AGPL-3.0) and link to the exact lines:

```rust
// excerpt goes here
```

<small>Source: {{<src path="crates/backend/storage/smgr/smgr/src/lib.rs" line={1} />}}</small>
