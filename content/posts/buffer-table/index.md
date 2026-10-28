+++
title = "Reading pgrust, part 2: the hash table that finds every page"
date = 2026-10-27
description = "How pgrust's buffer mapping table answers \"is this page in memory?\": 128 locked partitions, linear probing, a 75% grow threshold, and deletes that leave no tombstones."

[taxonomies]
tags = ["postgresql", "rust", "storage"]
series = ["Reading pgrust"]

[extra]
part = 2
+++

In [part 1](@/posts/buffer-pool/index.md) I described the buffer pool: an array of 8 KiB slots,
and a hash table that maps each page to the slot holding it. That table gets one lookup for
every page Postgres touches, so it's one of the hottest pieces of code in the database.

This is a part of pgrust that deliberately *doesn't* copy C. This post walks through the whole
file, {{<src path="crates/backend/storage/buffer/bufmgr/src/buf_table.rs" text="buf_table.rs" />}}:
lookup, insert, grow and the clever bit, delete.

<!-- more -->

## Why not just port C?

C Postgres builds this table on `dynahash`, its general-purpose hash table, which chains
entries through linked lists. The comment at the top of the Rust file says what pgrust does
instead, and why:

```rust
// Dense open-addressed replacement for C's dynahash buffer table (van de Meent
// RFC, upstream-unshipped-ideas #2): 24-byte contiguous entries, linear probe,
// ~1 pointer chase per warm lookup vs dynahash's 4. Stalled upstream on C's
// non-resizable shared memory; ours is process-local and grows per partition
// under the exclusive partition lock. External behavior (tag equality, insert
// collision return, delete error surface, partition-lock contract) is C-exact.
```

Two things stand out. The idea already existed upstream, but stalled because C's shared memory
can't grow. And the *interface* matches C exactly, so the rest of the buffer manager can't tell
the difference.

## The data: 24-byte entries in 128 arrays

Each entry is a page's tag plus the slot it's in:

```rust
#[repr(C)]
struct BufferLookupEnt {
    key: buftag, // 20 bytes: tablespace, database, relation, fork, block number
    id: i32,     // the slot index (buf_id)
}

const _: () = assert!(core::mem::size_of::<BufferLookupEnt>() == 24);
```

The `const _` line is a compile-time check: if anyone changes the struct so it's no longer 24
bytes, the build fails.

The table is split into **128 partitions**. Each partition is a separate array of these entries
with its own lock, so sessions looking up different pages rarely wait on each other:

```rust
struct Partition {
    entries: *mut BufferLookupEnt, // the array
    mask: u32,                     // array size − 1 (size is a power of two)
    count: u32,                    // entries in use
    grow_at: u32,                  // count at which the array doubles
}
```

### Empty slots have no flag

There's no "occupied" bit. An empty slot is one whose block number is `InvalidBlockNumber`
(0xFFFFFFFF), a value no real page can have:

```rust
// key.blockNum == InvalidBlockNumber marks an empty slot; BufTableInsert's
// blockNum assert is load-bearing for this sentinel (C asserts it too).
const EMPTY_BLOCK: u32 = types_core::InvalidBlockNumber;
```

That keeps entries at 24 bytes. The "load-bearing" assert is what makes it safe: inserting a
real page with that block number would make it look empty.

## From page to slot: two uses of one hash

The caller hashes the 20-byte tag once with an FxHash-style mix
({{<src path="crates/backend/storage/buffer/bufmgr/src/buf_table.rs" line={207} text="BufTableHashCode" />}}).
The same 32-bit hash is then used twice:

```rust
// which partition: the low 7 bits (128 = 2^7)
partitions().add((hashcode % NUM_BUFFER_PARTITIONS as u32) as usize)

// which slot inside it: the bits above those 7
fn ideal_slot(hashcode: u32, mask: u32) -> u32 {
    (hashcode >> 7) & mask
}
```

Why skip the low 7 bits for the slot? Every key in a partition has the *same* low 7 bits,
because that's how the partition was chosen. Using them again to place keys would pile every
entry into the same few slots. The comment in the code says just that.

## Lookup: walk until you find it or hit a gap

{{<src path="crates/backend/storage/buffer/bufmgr/src/buf_table.rs" line={276} end={292} text="BufTableLookup" />}}
is the whole read path:

```rust
/// Caller holds the partition lock (shared or better).
pub fn BufTableLookup(tag: &buftag, hashcode: u32) -> PgResult<i32> {
    unsafe {
        let part = &*partition_for(hashcode);
        let mask = part.mask;
        let mut slot = ideal_slot(hashcode, mask);
        loop {
            let ent = &*part.entries.add(slot as usize);
            if ent.key.blockNum == EMPTY_BLOCK {
                return Ok(-1);          // a gap: the page isn't cached
            }
            if ent.key == *tag {
                return Ok(ent.id);      // found it
            }
            slot = (slot + 1) & mask;   // someone else's page: try the next slot
        }
    }
}
```

This is **linear probing**. When a key's ideal slot is already taken at insert time, the key
goes in the next free slot after it. So a lookup starts at the ideal slot and walks forward:

- **Empty slot:** the page isn't in memory. `-1` means "not found", the same convention as C.
- **Same tag:** found. Return the slot index. The caller adds 1 to make the `Buffer` from part 1.
- **Different tag:** keep walking. `& mask` wraps from the last slot back to 0 without a `%`,
  because the array size is a power of two.

Two details matter:

- **The comparison is on the full tag**, not the hash. Different pages can share a hash.
- **`unsafe` is about two promises Rust can't check:** that `slot` is inside the array (the
  `& mask` guarantees it), and that nobody is changing the array (the caller holds the
  partition's lock).

A worked example, with an 8-slot partition and page P whose ideal slot is 5:

```text
slot:    0     1     2     3     4     5     6     7
       [ A ] [   ] [ B ] [   ] [ C ] [ D ] [ P ] [   ]
                                       ↑     ↑
                                    start  match
```

- Looking up **P**: slot 5 is D, keep going. Slot 6 is P, so return its slot.
- Looking up **Q**, which also starts at 5: D, then P, then slot 7 is empty, so return `-1`.

### How the read path uses it

In {{<src path="crates/backend/storage/buffer/bufmgr/src/read.rs" line={386} end={407} text="read.rs" />}},
a lookup runs with the partition lock in **shared** mode, so any number of sessions can search
the same partition at once:

```rust
let partition_lock = BufMappingPartitionLock(new_hash);
LWLockAcquire(partition_lock, LW_SHARED, ...)?;
let existing_id = BufTableLookup(&new_tag, new_hash)?;
```

On a hit, the caller pins the slot and it's done. On a miss, it picks a victim slot to evict,
then comes back to insert.

## Insert: same walk, write into the gap

Here is the whole of
{{<src path="crates/backend/storage/buffer/bufmgr/src/buf_table.rs" line={295} end={321} text="BufTableInsert" />}}:

```rust
/// -1 on success, existing id on collision; partition lock held exclusively.
pub fn BufTableInsert(tag: &buftag, hashcode: u32, buf_id: i32) -> PgResult<i32> {
    debug_assert!(buf_id >= 0);
    debug_assert!(tag.blockNum != types_core::InvalidBlockNumber);
    // SAFETY: exclusive partition lock — sole accessor of this partition.
    unsafe {
        let part = &mut *partition_for(hashcode);
        if part.count + 1 > part.grow_at {
            grow(part)?;
        }
        let mask = part.mask;
        let mut slot = ideal_slot(hashcode, mask);
        loop {
            let ent = &mut *part.entries.add(slot as usize);
            if ent.key.blockNum == EMPTY_BLOCK {
                ent.key = *tag;
                ent.id = buf_id;
                part.count += 1;
                return Ok(-1);
            }
            if ent.key == *tag {
                return Ok(ent.id);
            }
            slot = (slot + 1) & mask;
        }
    }
}
```

The probe loop is the same walk as lookup. What's different:

- **The two `debug_assert!`s** check the caller's side of the deal: a real slot index, and a
  real block number. The second one is the "load-bearing" assert from earlier. A tag whose
  block number is `InvalidBlockNumber` would be written into the table and then look like an
  empty slot, silently breaking every probe run through it. Like C's `Assert`, these only run
  in debug builds.
- **`&mut *partition_for(...)`, not `&*`.** Insert changes the partition, so it takes a mutable
  reference. That's only sound because the caller holds the partition's lock in **exclusive**
  mode, which the `SAFETY` comment states: no other session can be reading or writing this
  partition at the same time.
- **Grow first, then probe.** If one more entry would push the partition past 75% full, it
  doubles before searching (next section). Doing it first means the loop below always runs on
  an array that has room, so it's guaranteed to find a gap.
- **Writing into the gap.** The first empty slot on the walk is where the key goes: set the tag,
  the slot index, bump the count.
- **The return values are the opposite of lookup's.** For lookup, `-1` means "not found". For
  insert, `-1` means **success**: the entry is new. A value `>= 0` means the tag was *already
  there*, and returns the slot that holds it. The doc comment on the first line says so, and
  it's the same convention as C's `BufTableInsert`.

That second return handles a real race. Two sessions both miss on the same page, and both drop
the shared lock to go find a victim slot. Whichever inserts second finds the first one's entry
already there. The caller in `read.rs` handles it by giving back its own victim slot and using
the existing one:

```rust
let existing_id = BufTableInsert(&new_tag, new_hash, victim_desc.buf_id)?;
if existing_id >= 0 {
    UnpinBuffer(victim_desc);
    StrategyFreeBuffer(victim_desc.buf_id);
    let valid = PinBuffer(existing_desc, strategy);
    // ...
}
```

So the page is read from disk once, not twice.

## Who calls insert, and how they take the locks

`BufTableInsert` assumes its caller already holds the right partition lock exclusively. Across
the whole workspace there are only two callers outside tests, both in the `bufmgr` crate.

### 1. `BufferAlloc`: reading a page that isn't cached

{{<src path="crates/backend/storage/buffer/bufmgr/src/read.rs" line={370} text="BufferAlloc" />}}
runs on every buffer-cache miss. Its locking goes in this order:

| Step | Lock held | Code |
|---|---|---|
| 1. Look up the page | new page's partition, **shared** | {{<src path="crates/backend/storage/buffer/bufmgr/src/read.rs" line={391} end={392} text="read.rs:391" />}} |
| 2. Miss: let go | none | {{<src path="crates/backend/storage/buffer/bufmgr/src/read.rs" line={401} text="read.rs:401" />}} |
| 3. Find and evict a victim | the victim's content lock (only if dirty, to write it out), then the **old** page's partition, **exclusive**, to delete the old entry | {{<src path="crates/backend/storage/buffer/bufmgr/src/read.rs" line={403} text="read.rs:403" />}} → {{<src path="crates/backend/storage/buffer/bufmgr/src/read.rs" line={525} end={545} text="InvalidateVictimBuffer" />}} |
| 4. Insert the new entry | new page's partition, **exclusive** | {{<src path="crates/backend/storage/buffer/bufmgr/src/read.rs" line={406} end={407} text="read.rs:406" />}} |
| 5. Retag the slot | partition lock, plus the slot's header spinlock (`LockBufHdr`) inside it | {{<src path="crates/backend/storage/buffer/bufmgr/src/read.rs" line={419} text="read.rs:419" />}} |
| 6. Done | released | {{<src path="crates/backend/storage/buffer/bufmgr/src/read.rs" line={431} text="read.rs:431" />}} |

Three things about this order:

- **It never holds two partition locks at once.** The old page's entry is deleted under the old
  partition's lock, which is released before the new partition's lock is taken. If a session
  held both, two sessions could each hold one and wait for the other, a deadlock. Older C
  Postgres did hold both, and had to take them in a fixed order to avoid that. Postgres 16
  changed it to this one-at-a-time scheme, and pgrust follows it.
- **The slow work happens with no partition lock held.** Running the clock sweep and writing
  out a dirty victim can take milliseconds. Doing that while holding a partition lock would
  stall every other lookup in that 1/128th of the table.
- **That gap is why the collision case exists.** Between steps 2 and 4, another session can
  miss on the same page and insert it first. Step 4 then gets back the existing slot and gives
  up its victim, as shown above.

The victim slot stays **pinned** the whole time, so nobody else can evict or reuse it while
this session holds no lock on it.

Inside a partition lock, the order is always partition lock first, then the slot's header
spinlock. The spinlock is only ever held for a few instructions.

### 2. `ExtendBufferedRelShared`: adding new pages to a table

When a table grows, new blocks need buffer slots too.
{{<src path="crates/backend/storage/buffer/bufmgr/src/extend.rs" line={226} text="ExtendBufferedRelShared" />}}
does it in this order:

1. **Get the victim slots first, with no locks held**
   ({{<src path="crates/backend/storage/buffer/bufmgr/src/extend.rs" line={244} text="extend.rs:244" />}}),
   and zero them. The comment explains: the pins keep them safe, and this keeps slow eviction
   work out of the next lock.
2. **Take the relation extension lock**
   ({{<src path="crates/backend/storage/buffer/bufmgr/src/extend.rs" line={252} text="extend.rs:252" />}}).
   This is a heavyweight lock on the table itself, not an LWLock. It makes sure only one session
   at a time decides what the next block numbers are.
3. **For each new block:** take its partition lock exclusively, insert, retag the slot, release
   the partition lock, and mark the slot as having I/O in progress
   ({{<src path="crates/backend/storage/buffer/bufmgr/src/extend.rs" line={310} end={355} text="extend.rs:310" />}}).
   Again only one partition lock is held at a time.
4. **Grow the file on disk** (`smgr_zeroextend`) and release the extension lock
   ({{<src path="crates/backend/storage/buffer/bufmgr/src/extend.rs" line={361} end={364} text="extend.rs:361" />}}).

Step 3 comes before step 4 on purpose. The comment there says: *"Insert into the mapping table
(IO-in-progress) before smgrzeroextend: once the relation grows, other backends may read these
blocks."* If another session reads one of the new blocks the moment the file grows, it finds the
mapping entry and waits for this session's I/O to finish, instead of reading the block from disk
into a second slot.

Here, a collision means a leftover: a buffer for this block number already exists, from an
earlier extension that failed or a read past the end of the file. The code reuses it, but only
if the page is empty. Otherwise it raises *"unexpected data beyond EOF"*, with the hint that
this has been seen with buggy kernels.

## Grow: never more than 75% full

The lookup loop has no counter. If every slot were full and the key missing, it would spin
forever. The table makes sure that can't happen by doubling a partition before it gets 75% full:

```rust
grow_at: (cap - cap / 4) as u32,
```

{{<src path="crates/backend/storage/buffer/bufmgr/src/buf_table.rs" line={330} text="grow()" />}}
allocates an array twice the size and reinserts every entry at its new ideal slot
({{<src path="crates/backend/storage/buffer/bufmgr/src/buf_table.rs" line={342} text="rehash_into" />}}).
It's marked `#[cold]` and `#[inline(never)]` so the compiler keeps it out of the hot insert path.

In practice it rarely runs. The starting size
({{<src path="crates/backend/storage/buffer/bufmgr/src/buf_table.rs" line={196} text="partition_cap_for" />}})
leaves plenty of room:

```rust
fn partition_cap_for(size: i32) -> usize {
    let per_part = (size.max(0) as usize / NUM_BUFFER_PARTITIONS as usize) + 1;
    (per_part * 2).next_power_of_two().max(16)
}
```

With the default 16,384 buffers, the table is sized for 16,512 entries, which is 130 per
partition. Doubled and rounded up to a power of two, that's **512 slots per partition**: 12 KiB
each, about 1.5 MiB for all 128. A full pool puts about 128 entries in each partition, so each is
only about 25% full, and probe runs stay very short. Growing matters when the pool is resized
larger while running, or when the hash spreads unevenly.

## Delete: the clever part

Deleting from a linear-probing table is where it's easy to get things wrong. Back to the example:

```text
slot:    4     5     6     7
       [ C ] [ D ] [ P ] [   ]       P's ideal slot is 5
```

If deleting D just emptied slot 5, a lookup for P would start at 5, see a gap, and report "not
cached", even though P is sitting in slot 6. The lookup rule "stop at the first gap" only works
if a key is never separated from its ideal slot by a gap.

The usual fix is a *tombstone*: a "deleted" marker that lookups walk past. Tombstones build up
over time and make lookups slower, and the table has to be cleaned up now and then. pgrust uses
**backward-shift deletion** instead, which leaves no marker
({{<src path="crates/backend/storage/buffer/bufmgr/src/buf_table.rs" line={453} end={471} text="BufTableDelete" />}}):

```rust
// Backward-shift deletion (linear probing has no tombstones): pull
// forward any entry whose ideal slot lies cyclically at-or-before the
// hole, preserving every probe chain.
let mut hole = slot;
let mut j = slot;
loop {
    j = (j + 1) & mask;
    let ent = &*part.entries.add(j as usize);
    if ent.key.blockNum == EMPTY_BLOCK {
        break;
    }
    let ideal = ideal_slot(BufTableHashCode(&ent.key), mask);
    if (j.wrapping_sub(ideal) & mask) >= (j.wrapping_sub(hole) & mask) {
        *part.entries.add(hole as usize) = core::ptr::read(part.entries.add(j as usize));
        hole = j;
    }
}
(*part.entries.add(hole as usize)).key.blockNum = EMPTY_BLOCK;
```

After removing an entry, there's a hole. The loop walks the entries after it, up to the next
empty slot, and asks one question about each: **may this entry move back into the hole?**

It may if the hole is between the entry's ideal slot and where the entry is now. Then moving
it is just a shorter walk from the same starting point. The `if` measures that with two
distances, both going backwards from slot `j` and wrapping around the end of the array:

- `j - ideal`: how far the entry is from its ideal slot
- `j - hole`: how far it is from the hole

If the first is at least the second, the hole is on the entry's path, so the entry moves into
it, and its old slot becomes the new hole. If not, moving it would put it *before* its ideal
slot, where lookups would never find it, so it stays. When the walk reaches an empty slot, the
last hole is marked empty.

An example with wrap-around. In an 8-slot partition, delete D:

```text
             slot:   4    5    6    7    0    1
before:            [C4] [D5] [P5] [E7] [F6] [  ]     (letter + ideal slot)
remove D:          [C4] [  ] [P5] [E7] [F6] [  ]     hole = 5
```

- **P** (slot 6, ideal 5): distance from ideal is 1, distance from hole is 1. 1 ≥ 1, so P moves
  to 5. Now hole = 6.
- **E** (slot 7, ideal 7): distance from ideal is 0, from hole is 1. 0 < 1, so E stays. It's
  already in its ideal slot.
- **F** (slot 0, ideal 6, wrapped around the end): distance from ideal is 2 (6 → 7 → 0), from
  hole is 2. 2 ≥ 2, so F moves to 6. Now hole = 0.
- **Slot 1 is empty**, so stop and mark slot 0 empty.

```text
after:             [C4] [P5] [F6] [E7] [  ] [  ]
```

Every entry can still be reached from its ideal slot without crossing a gap. P is now in its
ideal slot, and so is F. The table looks exactly as if D had never been inserted.

If the walk to find the tag hits a gap first, the tag was never there. C reports that as
corruption, and so does pgrust, with the same message: "shared buffer hash table corrupted".

## Why it's fast

- **Entries are contiguous.** A hit usually touches one or two neighbouring 24-byte entries,
  often in the same cache line. That's the "~1 pointer chase vs dynahash's 4" from the header.
- **Low load.** About 25% full by default, never more than 75%, so probe runs are short.
- **Nothing extra in the loop.** The hash is computed once by the caller, the lock is already
  held, and nothing is allocated.
- **Readers run together.** Lookups take a partition's lock in shared mode; only inserts and
  deletes need it exclusively, and even then only 1/128th of the table is blocked.
- **No tombstones.** The table never fills with dead markers, so it never slows down with age.

And because the interface matches C exactly (the same `-1` conventions, the same collision
return, the same error), the rest of the buffer manager is a straight port that has no idea
the table underneath is completely different.

*Read at pgrust v0.4-beta, commit 38634b4.*
