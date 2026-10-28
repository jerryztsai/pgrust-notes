+++
title = "Reading pgrust, part 3: the jsonb columns that are stored twice"
date = 2026-10-28
description = "pgrust's columnar tables store each table once, except for jsonb: every document is kept whole, and its common paths are also pulled out into typed columns that queries can scan instead."

[taxonomies]
tags = ["postgresql", "rust", "columnar", "jsonb"]
series = ["Reading pgrust"]

[extra]
part = 3
+++

pgrust v0.4 added columnar tables, and the first question I had about them was: does a columnar
table keep a row-format copy too? Is everything stored twice?

The answer is no, with one deliberate exception. That exception is `jsonb`, and it's the
subject of this post.

<!-- more -->

## One table, one format

Columnar storage in pgrust is a table access method, `pgrcolumnar`, and you choose it per table:

```sql
CREATE ACCESS METHOD pgrcolumnar TYPE TABLE HANDLER heap_tableam_handler;  -- once per database

CREATE TABLE events (id bigint, at timestamptz, doc jsonb) USING pgrcolumnar;
```

Every column of that table is stored in columnar *parts*: immutable, compressed files, one
stream per column. There's no per-column switch, and no hidden heap copy of the table. You can
move a table between formats with `ALTER TABLE ... SET ACCESS METHOD`, but that converts it; it
doesn't keep both.

Two things can look like double storage but aren't:

- **The row-store front.** Small inserts land first in a ring of ordinary heap segments
  (`pg_toast.pgrc_seg_<table oid>_<n>`, see
  {{<src path="crates/backend/catalog/pgrc_companion/src/lib.rs" text="pgrc_companion" />}}),
  because rewriting a compressed part for one row would be absurd. A background worker
  seals each full segment into a columnar part and reclaims it. A row lives in the front *or*
  in a part, not in both.
- **Delete vectors.** `UPDATE` and `DELETE` don't rewrite parts. They mark rows dead, and
  compaction later rewrites the part without them.

Now the exception.

## Why jsonb is the hard case

A columnar format wins because a query reads only the columns it needs, and each column is a
dense array of one type that compresses well and can be skipped using min/max "zone maps".

A `jsonb` column throws most of that away. Each value is a whole document, so a query like

```sql
SELECT count(*) FROM events WHERE doc->>'kind' = 'click';
```

has to read every document and parse it just to pull out one field. Stored as one opaque blob
per row, the column compresses poorly and has no useful zone maps.

The usual fix, used by systems like ClickHouse's JSON type and Snowflake's VARIANT, is
*shredding*: pull each common path out of the documents into its own typed column. pgrust does
this too, and it does it on top of keeping the whole document.

## The dual-store rule

The rule is stated at the top of
{{<src path="crates/backend/access/pgrcolumnar/pgrc_write/src/shred.rs" line={1} end={25} text="shred.rs" />}}:

> **Dual-store**: the verbatim IMAGE lane (path_ord 0 — the parent column itself, byte-exact
> per the detoast law) is ALWAYS emitted; typed path lanes are additional row-aligned
> substreams ... the image lane is the read truth.

So inside each columnar part, a `jsonb` column is stored as:

- **The image lane**: every document, byte for byte, exactly as Postgres would store it. This
  is always written, and it's what any query that needs the actual document reads.
- **Zero or more path lanes**: one typed column per chosen path, such as `kind` as text or
  `user.id` as a number. Each has exactly one entry per row of the part ("row-aligned"), with
  NULL where the document doesn't have that path.

That's the one place pgrust really stores data twice: the value of `doc->>'kind'` sits inside
the document in the image lane, and again in the `kind` lane.

Keeping the full copy is what makes the extra copy safe. A path lane only has to be right
about one thing, and if anything about it is doubtful, the reader can always go back to the
document. You'll see that idea again and again below.

(`shred.rs`'s header also says the shredding engine "has NOT been vendored yet". That comment is
out of date. The engine now lives in
{{<src path="crates/backend/utils/adt/jsonb_shred" text="crates/backend/utils/adt/jsonb_shred" />}},
and {{<src path="crates/backend/access/pgrcolumnar/pgrc_write/src/shred_jsonb.rs" text="shred_jsonb.rs" />}}
plugs it into the writer. The code after this point is the real path.)

## You don't have to turn it on

The first surprise: shredding is automatic. When a part is written, pgrust registers **every
`jsonb` column of the table** for shredding, from
{{<src path="crates/backend/access/pgrcolumnar/pgrc_am/src/schema.rs" line={538} end={546} text="jsonb_attnos" />}}:

```rust
pub fn jsonb_attnos(rel: &Relation<'_>) -> Vec<u32> {
    use types_core::catalog::JSONBOID;
    rel.rd_att
        .attrs
        .iter()
        .filter(|a| !a.attisdropped && a.atttypid == JSONBOID)
        .map(|a| a.attnum as u32)
        .collect()
}
```

That list goes into the shred source in
{{<src path="crates/backend/access/pgrcolumnar/pgrc_am/src/ingest.rs" line={510} end={548} text="shred_source_for" />}}.
A table with no `jsonb` columns registers nothing and is never shredded.

## Choosing paths: the election

For each `jsonb` column, the writer chooses which paths get their own lane at the moment it
seals a part. pgrust calls this an *election*. It happens separately for every part, so a part
of old data and a part of new data can have different lanes if the documents changed shape.

The election is
{{<src path="crates/backend/utils/adt/jsonb_shred/src/elect.rs" line={104} end={205} text="elect_manifest_auto" />}}.
It works like this:

1. **Count.** Walk every document in the part, down to a depth of 4 keys. For every path, count
   how many documents contain it and how often the value is a string, a number or a boolean.
2. **Admit declared paths first.** Paths you asked for (covered below) are admitted
   unconditionally.
3. **Admit common paths.** Every other path must appear in at least 1/64 of the documents
   (`floor = max(1, nrows / 64)`). The survivors are sorted most common first and admitted
   until there are 64 lanes. Each gets the type that most of its values have: text, numeric or
   bool.
4. **Keep it prefix-free.** A path is skipped if it's a prefix of one already admitted, or the
   other way round. So you can't have both a `user` lane and a `user.id` lane. Whichever was
   admitted first wins.

The defaults are in `ShredBudgets::default()`:

```rust
ShredBudgets {
    max_paths: 64,        // lanes per column, declared paths included
    max_depth: 4,         // longest key chain that can be elected
    nbuckets: 8,
    min_presence_div: 64, // a path must appear in >= 1/64 of rows
}
```

The presence floor exists for documents that are mostly unique keys, like a map from session
IDs to values. Without it, every one of those keys would become a nearly-empty lane. With it,
nothing gets elected and the column is stored as documents only.

## The rule that keeps lanes honest

This is the part I found most interesting. After the shredder fills a lane, the writer decides
whether to actually keep it, in
{{<src path="crates/backend/access/pgrcolumnar/pgrc_write/src/shred_jsonb.rs" line={135} end={231} text="JsonbShredSource::shred" />}}.

The rule is called *exception freedom*: a lane is written for a part **only if every document in
that part that has the path holds the lane's type there.** Suppose `kind` is a string in 9,999
documents and the number `3` in one. Then the part gets no `kind` lane at all.

Why throw away a nearly perfect lane? Because of what a NULL in the lane has to mean. The read
side wants to treat "the lane is NULL here" as "`doc->>'kind'` is SQL NULL here", without
opening the document. That's only true if a NULL in the lane can come from just three things:

- the whole `doc` is SQL NULL,
- the document has no `kind` key, or
- `kind` is JSON `null`, which `->>` also returns as SQL NULL.

If an odd value like the number `3` were also allowed to sit as a NULL in the lane, the reader
would have to check the document for every NULL. The lane would stop being a reliable shortcut.

JSON `null` is the one exception that doesn't cost the lane, because it gives the same answer
through `->>`. The code comments mention that an earlier version also dropped the lane for
nulls; the fix is marked "P3a".

Two other fallbacks follow the same thinking:

```rust
if manifest.elected.is_empty() {
    // Not shreddable (OD-4 arm 2): the text plane carries it.
    return Ok(Vec::new());
}
```

```rust
// A shred failure must never fail an ingest (the image lane is
// whole regardless — dual-store law): fall to the text plane.
Err(_) => return Ok(Vec::new()),
```

If no path qualifies, or the shredder fails for any reason, the part is simply written with the
documents only. That's always correct, because the documents are always there. A shredder bug
can cost you speed but not data.

## Asking for a path: `pgrc_shred_paths`

You can force paths with a table option, using the syntax defined by
{{<src path="crates/_support/types/rel/src/reloptions.rs" line={248} end={283} text="ShredPathDecl" />}}:

```sql
CREATE TABLE events (id bigint, doc jsonb) USING pgrcolumnar
  WITH (pgrc_shred_paths = 'user.id:numeric, kind:text, session:uuid');
```

Each entry is a dotted path of object keys, with an optional type: `text` (the default),
`numeric` (`int`, `bigint` and similar are accepted as aliases), `bool` or `uuid`. A declared
path is elected in **every** part, at that type, ahead of the 64-lane cap and without the
presence floor.

It's a request, not a guarantee. A declared lane still has to pass exception freedom, so if one
document in a part has `user.id` as a string, that part has no `user.id` lane and queries read
the documents for it. Declaring a type is mainly useful for a path that's rare, and so would miss
the presence floor, or for `uuid`, which the automatic election never picks because it only
chooses text, numeric or bool.

A path in `sorted_by` is forced too. If you sort a table by `doc->>'ts'`, the writer always
elects `ts` at whatever type its values have, so the zone maps for the sort key exist in every
part.

There's also a lane budget option, `pgrc_shred_max_paths`, but `ingest.rs` notes that it isn't
connected yet: the writer always uses `ShredOptions::default()`, with its 64-lane budget.

## How queries use the lanes

On the read side, Gravity's columnar scan treats each lane as just another column
({{<src path="crates/gravity/hosts/src/ops/scan/pgrc/granule_source.rs" line={80} end={95} text="granule_source.rs" />}}).
A filter on `doc->>'kind'` can then:

- read only the narrow `kind` column instead of every document,
- use the lane's zone maps and bloom filters to skip whole chunks, and
- compare typed values directly, without parsing JSON.

Lanes are chosen separately for each part, so the scan works out per part which lanes exist.
In a part without the lane, it reads the documents for that query. You get the same results
either way; only the speed differs.

## So, is it stored twice?

| What | Stored as |
|---|---|
| A `heap` table | Rows, once. |
| A `pgrcolumnar` table | Columns, once. New rows wait briefly in the row-store front first. |
| A `jsonb` column in a `pgrcolumnar` table | The full documents, **plus** a typed column for each common or declared path that passes exception freedom. |

The duplication only covers individual paths inside the documents, never whole tables. It's
also what makes shredding safe: because the original documents are always there, a lane can be
dropped whenever it's in doubt, and a reader can always go back to the documents.

A note on how far I checked: everything here comes from reading the code at v0.4-beta. I didn't
build pgrust and measure how much space the lanes take or how much faster queries get. If I do,
that will be its own post.
