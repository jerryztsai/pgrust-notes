+++
title = "Reading pgrust, part 1: one i32 and the buffer pool behind it"
date = 2026-10-25
description = "What pgrust's Buffer type encodes, how the shared buffer pool finds, pins and evicts pages, and how big it is: 8 KiB pages, 16,384 slots by default, resizable while running."
# Drafts only show with `zola serve --drafts`. Remove this line to publish.
draft = true

[taxonomies]
tags = ["postgresql", "rust", "storage"]
series = ["Reading pgrust"]

[extra]
part = 1
+++

This is the line I keep coming back to while reading pgrust's storage code:

```rust
/// Buffer-pool slot index: positive = shared, negative = local, 0 = invalid.
pub type Buffer = i32;
```

It's in {{<src path="crates/_support/types/types_core/src/primitive.rs" line={116} text="types_core/src/primitive.rs" />}},
and nearly every function that touches a table page takes or returns one. This post covers what
those 32 bits encode, how the buffer pool behind them works, and how big that pool is.

<!-- more -->

## A port of a C typedef

The Rust line is a straight port of Postgres's `src/include/storage/buf.h`:

```c
/*
 * Zero is invalid, positive is the index of a shared buffer (1..NBuffers),
 * negative is the index of a local buffer (-1 .. -NLocBuffer).
 */
typedef int Buffer;

#define InvalidBuffer	0
#define BufferIsLocal(buffer)	((buffer) < 0)
```

So one integer names a slot in one of two pools:

| Value | Meaning | Index into the pool's arrays |
|---|---|---|
| `0` | `InvalidBuffer`: no buffer | none |
| `1 ..= NBuffers` | A slot in the **shared** pool, seen by every session | `buffer - 1` |
| `-1 ..= -NLocBuffer` | A slot in this session's **local** pool, used for temporary tables | `-buffer - 1` |

Packing it this way buys three things:

- **One type for both pools.** The heap and index code pass a `Buffer` around without caring
  where it lives. Only the buffer manager checks the sign and sends the call to the shared or
  local implementation.
- **Zero is free as "none".** Counting starts at 1, so `0` can mean invalid and nothing needs
  an `Option<Buffer>`.
- **It's 4 bytes.** It's cheap to copy, store in structs and pass between crates.

You can see the off-by-one in the code that turns a slot back into a `Buffer`, in
{{<src path="crates/backend/storage/buffer/bufmgr/src/buf_hdr.rs" line={201} end={203} text="BufferDescriptorGetBuffer" />}}:

```rust
pub fn BufferDescriptorGetBuffer(desc: &BufferDesc) -> Buffer {
    desc.buf_id + 1
}
```

### Why not a newtype?

Idiomatic Rust would write `struct Buffer(i32)`, so that a block number or an OID can't be
passed where a `Buffer` is expected. pgrust doesn't, and it keeps the C names too
(`InvalidBuffer`, `BufferIsValid`). That's a deliberate trade I see all over the codebase:
it gives up some type safety so the Rust reads almost line for line like the C it ports, which
makes checking it against the vendored C source in
{{<src path="crates/postgres-18.6-reference" />}} easy.

## What's behind the number: the shared buffer pool

Postgres never reads table or index pages from disk straight into a query. Every page goes
through the **buffer pool**, a fixed array of page-sized slots in memory. A `Buffer` is an
index into that array. The pool is three parallel arrays, all indexed by `buf_id`
(that is, `Buffer - 1`):

1. **Blocks.** The pages themselves, `BLCKSZ` bytes each. Getting a page's address is plain
   arithmetic, from {{<src path="crates/backend/storage/buffer/bufmgr/src/buf_hdr.rs" line={215} end={223} text="BufferGetBlockPtr" />}}:

   ```rust
   unsafe { blocks.add((buffer as usize - 1) * BLCKSZ) }
   ```

2. **Descriptors** (`BufferDesc`). One per slot, padded to 64 bytes (a cache line), saying
   which page the slot holds and what state it's in.
3. **I/O condition variables.** One per slot, for sessions waiting on a read or write of that
   slot to finish.

### The descriptor: which page, and what state

A descriptor's **tag** names the page the slot currently holds. It's a 20-byte `buftag`
({{<src path="crates/_support/types/types_storage/src/buf.rs" line={52} text="types_storage/src/buf.rs" />}}):
tablespace, database, relation file, fork (main data, free-space map or visibility map) and
block number.

Its **state** is one atomic `u32` that packs three things, so most updates are a single
compare-and-swap rather than a lock:

```text
 31                    22 21      18 17                         0
┌────────────────────────┬──────────┬───────────────────────────┐
│ 10 flag bits           │ usage    │ refcount (pins)           │
│ LOCKED DIRTY VALID ... │ count    │ 18 bits                   │
│                        │ 0..5     │                           │
└────────────────────────┴──────────┴───────────────────────────┘
```

- **refcount**: how many sessions have the buffer *pinned*. A pinned buffer can't be evicted.
- **usage count**: how recently and how often the buffer was used, capped at
  `BM_MAX_USAGE_COUNT = 5`. Pinning a buffer bumps it.
- **flags**: `BM_DIRTY` (changed in memory, not yet written to disk), `BM_VALID` (the page
  contents are loaded), `BM_IO_IN_PROGRESS`, `BM_LOCKED` (a tiny spinlock on the header) and
  others.

### Finding a page: the mapping table

To read block 42 of some table, the buffer manager builds the tag, hashes it, and looks it up
in the **buffer mapping table**, a hash map from tag to `buf_id`. The table is split into 128
partitions, each with its own lock, so sessions reading different pages rarely contend.

This is one place where pgrust departs from C on purpose. The comment at the top of
{{<src path="crates/backend/storage/buffer/bufmgr/src/buf_table.rs" line={6} text="buf_table.rs" />}}
explains it: C uses its general-purpose `dynahash` table, while pgrust uses a dense
open-addressed table with 24-byte entries and linear probing. That's about one pointer chase
per lookup instead of four. Upstream Postgres considered the same idea but couldn't adopt it,
because its shared memory can't grow; pgrust's can, so each partition grows on its own.

### Reading a page, end to end

1. **Look up the tag** under the partition's lock, in shared mode.
2. **Hit:** pin the slot (refcount + 1, usage count + 1) and return `buf_id + 1`. No I/O.
3. **Miss:** pick a victim slot (next section). If the victim is dirty, write it out first,
   after the WAL describing its changes has been flushed. Then move the slot to the new tag in
   the mapping table, read the page from disk through the storage manager (`smgr`), and mark it
   `BM_VALID`.
4. The caller locks the buffer's *contents* (shared to read, exclusive to change) while it
   works on the page, then unlocks and unpins it with `ReleaseBuffer`.

### Evicting a page: clock sweep

When the page isn't in memory, some other page has to give up its slot. Postgres uses a
**clock sweep**, an approximation of "least recently used" that's cheap to run. A shared "clock
hand" walks the slots in a circle. From the loop in
{{<src path="crates/backend/storage/buffer/bufmgr/src/freelist.rs" line={247} end={275} text="StrategyGetBuffer" />}}:

```rust
if local_buf_state & BUF_REFCOUNT_MASK == 0 {
    if local_buf_state & BUF_USAGECOUNT_MASK != 0 {
        UnlockBufHdr(buf, local_buf_state - BUF_USAGECOUNT_ONE);
        trycounter = crate::extent::span();
    } else {
        // ... usage count is 0: this slot is the victim
```

At each slot the hand passes:

- **Pinned:** someone is using it, so skip it.
- **Usage count above 0:** knock it down by one and move on. A hot page used 5 times survives
  5 passes of the hand.
- **Usage count 0 and not pinned:** that's the victim.

If the hand goes all the way round without finding anything to evict, every buffer is pinned,
and the query fails with "no unpinned buffers available".

Big sequential scans and `VACUUM` don't sweep the whole pool. They use a small **ring** of
buffers (`BufferAccessStrategy`) and recycle it, so one large table scan can't push every hot
page out of the cache.

## How big is the pool?

### Page size: 8 KiB

```rust
pub const BLCKSZ: usize = 8192;
```

({{<src path="crates/_support/types/types_core/src/primitive.rs" line={69} text="primitive.rs" />}}.)
Each slot holds one 8 KiB page, the same as stock Postgres. It's a compile-time constant
because it's also the on-disk page size: data files are a sequence of 8 KiB blocks.

### Number of slots: `shared_buffers`

The `shared_buffers` setting is the number of slots. It's registered in
{{<src path="crates/backend/utils/misc/guc_tables/src/tables.rs" line={927} text="guc_tables/src/tables.rs" />}}
with the same values as C:

| | Slots | Memory for pages |
|---|---|---|
| Default | 16,384 | 128 MiB |
| Minimum | 16 | 128 KiB |
| Maximum | 2<sup>30</sup> − 1 (≈ 1.07 billion) | ≈ 8 TiB |

On top of the pages, each slot costs a 64-byte descriptor plus a condition variable. For the
default pool that's about 1 MiB of descriptors next to 128 MiB of pages, under 1%.

pgrust can also **size the pool from the machine**. With `PGRUST_MEM_AUTOTUNE=1`,
{{<src path="crates/backend/utils/misc/guc/src/autotune.rs" line={68} text="autotune.rs" />}}
sets `shared_buffers` to 25% of RAM, never less than 128 MiB. On a 64 GiB machine that's
16 GiB, or 2,097,152 slots. There's a catch, explained at length in that file's header:
`initdb` writes an explicit `shared_buffers = 128MB` line into `postgresql.conf`, and an
explicit setting always wins over the automatic one. To get the 25% default, remove that line
or run `initdb` with `-c shared_buffers=...`.

### Local buffers: `temp_buffers`

Temporary tables are visible to only one session, so their pages skip the shared pool and its
locking entirely. They go in that session's local pool, which is where the negative `Buffer`
numbers point. The size is `temp_buffers` (registered at
{{<src path="crates/backend/utils/misc/guc_tables/src/tables.rs" line={939} text="tables.rs:939" />}}):
1,024 slots by default (8 MiB per session), at least 100.

## The part C Postgres can't do: resizing while running

In stock Postgres, `shared_buffers` is fixed at startup; changing it means a restart. pgrust
can grow and shrink the pool while the server runs. The design is described at the top of
{{<src path="crates/backend/storage/buffer/bufmgr/src/extent.rs" text="extent.rs" />}} and
{{<src path="crates/backend/storage/buffer/bufmgr/src/resize.rs" text="resize.rs" />}}:

- **Reserve address space up front.** At startup pgrust reserves virtual memory for the largest
  size the pool may ever reach, `pgrust.buffer_pool_max`. The pool's base addresses never
  move, so the hot path (the pointer arithmetic above) stays a single add.
- **Cut it into extents of 256 slots** (`EXTENT_SHIFT = 8`), which is 2 MiB of pages, the size
  of a huge page on x86-64 and Arm (`pgrust.buffer_pool_thp` can back the pool with
  transparent huge pages). The default 128 MiB pool is 64 extents.
- **Each extent has a state:** active, draining, retired or never used.
- **Shrinking** drains extents from the top down. Each one is marked as draining so the clock
  sweep stops handing out its slots. Then every buffer in it is written out if dirty, moved to a
  surviving slot if it's clean and recently used, or dropped. The emptied extent's memory goes
  back to the kernel. An extent holding a pinned buffer is skipped and retried until a deadline,
  so a long-held pin gives you a partial shrink rather than a hang.
- **Growing** reactivates the lowest unused extents.
- **Buffer numbers stay dense.** A retired extent keeps its descriptors; they just read as
  invalid, unpinned buffers. So a `Buffer` is still a plain index, and none of the code above
  the buffer manager needs to know resizing exists.

A superuser triggers it with:

```sql
CALL pgrust_buffer_pool_resize(target_bytes, timeout_ms);
SELECT * FROM pgrust_buffer_pool_stats();
```

With the default `pgrust.buffer_pool_max = 0`, the reservation equals `shared_buffers`, so the
pool can shrink and grow back but not past its starting size. Set it higher at startup to
leave room to grow. With its `buf_giveback` knob on, pgrust's memory manager can also use this
to give buffer-pool memory back when the server is short of memory (`giveback.rs`), with the
background writer doing the draining in 10 ms slices.

One more difference sits underneath all of this. C Postgres runs one process per connection, so
its buffer pool has to live in *shared memory* that every process maps. pgrust runs every
session as a thread in a single process, so the pool is ordinary process memory, which is part
of why resizing it is practical at all.

## Recap

- A `Buffer` is an `i32`: positive for the shared pool, negative for this session's temporary
  table pool, 0 for none. It's a 1-based index into the pool's arrays.
- The pool is arrays of 8 KiB pages, 64-byte descriptors and condition variables. A partitioned
  hash map from page tag to slot finds pages, and a clock sweep picks which page to evict.
- By default there are 16,384 shared slots (128 MiB) and 1,024 local slots per session (8 MiB),
  the same as stock Postgres.
- pgrust can size the pool from the machine's RAM, and grow or shrink it while running in 2 MiB
  extents, which stock Postgres can't do.

*Read at pgrust v0.4-beta, commit 38634b4.*
