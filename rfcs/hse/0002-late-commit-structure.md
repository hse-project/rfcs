# Ensuring transaction kv-tuples are committed before ingesting into cN

## Problem

One of the invariants in cN is that newer kvsets contain newer data.
Since cN kvsets are searched in order from newest to oldest, this
invariant allows cN queries to terminate on the first kvset with a
suitable match.

Recent changes in c0 allow transactions to store uncommitted
kv-tuples in the active c0_kvmultiset .  Uncommitted kv-tuples do not
yet have sequence numbers. If they were ingested into cN and later
assigned a sequence number, the assigned number would be higher than
sequence numbers for other unrelated kv-tuples that have not yet been
ingested into cN.  This would violate the invariant.

## Requirements

- Design must not prevent concurrent ingest operations

## Non-Requirements

## Terminology and review

- **_kv-tuple_** -- A key and an associated value, tombstone or prefix
  tombstone.  Note that (prefix) delete operations are mutations that
  add kv-tuples with (prefix) tombstones to the KVDB rather than
  actually removing a kv-tuple.
- **_c0_kvmultiset_** -- A collection of kv-tuples in c0.  Each KVDB
  has one **_active c0_kvmultiset_** and zero or more **_frozen
  c0_kvmultisets_**.  The active c0_kvmultiset stores incoming
  kv-tuples (from HSE puts, gets, etc.).  When the active
  c0_kvmultisets becomes "full", it is frozen and a new active
  c0_kvmultiset is created.
- **_Ingest_** refers the process of migrating kv-tuples from the oldest
  c0_kvmultiset into cN, at which point it becomes the newest cN kvset.
- **_dgen_** - A data generation number assigned to each
  c0_kvmultiset.  The initial active c0_kvmultiset is assigned a dgen of 1.
  Each new active c0_kvmultiset is assigned a dgen one greater than
  the dgen of the previous active c0_kvmultiset.  Dgen numbers persist
  across KVDB open/close so they are never reused for the
  life of a KVDB.  Dgen numbers follow data into cN.  cN kvsets
  creating during ingest are associated with a single dgen (the same
  dgen as the ingested c0_kvmultiset).  cN kvsets created during
  compaction of M kvsets with dgen numbers *x* to *x+M-1* are
  associated with that range of dgen numbers.
- A **_txn kv-tuple_** is a a key-value pair created by a transaction.
  A txn kv-tuple can be active, committed or aborted (i.e., in the
  same state as the transaction that created it).

## Solution

### Overview

This RFC proposes a new data structure referred to as the **_Late
Commit_** structure, or simply **_LC_**.  There will be one LC per
KVDB.

During ingest, active txn kv-tuples will be stored in LC instead of
being ingested into cN.  As transactions are aborted and committed,
kv-tuples in LC will become aborted and committed.

The ingest operation will scan LC to copy committed kv-tuples into
cN.  Committed LC kv-tuples are marked so they can be garbage
collected along with aborted LC kv-tuples.

LC provides snapshot versioning so that reads have a stable view of
the kv-tuples in LC.  Snapshots are based on dgens, and each ingest
operation produces a new snapshot.

Txn reads will search LC for uncommitted kv-tuples belonging to the
txn as well as for committed kv-tuples produced by other txns.

Non-txn reads will search LC for committed kv-tuples.

Both txn and non-txn reads provide a view seqno and a view dgen when
searching LC.  The dgen identifies that LC snapshot being searched.

Notes:
- LC snapshot versioning is probably not necessary, but I don't think
  it adds much complexity or overhead, and it greatly simplifies
  reasoning about LC.

### State Diagrams

This RFC uses state diagrams as shown below to help explain how LC functions.

```
+------------+    +------------+    +------------+    +------------+
| c0kvms[4]  |--->| c0kvms[3]  |    | c0kvms[2]  |    | c0kvms[1]  |
|------------|    |------------|    |------------|    |------------|
| k/18..13   |    | k/13..9    |    | k/9..4     |    | k/4..1     |
|            |    | t1:b/*     |    |            |    | t1:a/*     |
|            |    |            |    | t2:y/*     |    | t2:x/*     |
+------------+    +------------+    +------------+    +------------+
                        |
                        |           +------------+    +------------+
                        +---------->| LC[2]      |    | LC[1]      |
                                    |------------|    |------------|
                                    | t1:a/*     |    | t1:a/*     |
                                    | t2:x/*     |    | t2:x/*     |
                                    | t2:y/*     |    +------------+
                                    +------------+
                                          |
                                          V
                                    +------------+    +------------+
                                    | cnkvset[2] |--->| cnkvset[1] |
                                    |------------|    |------------|
                                    | k/9..4     |    | k/4..1     |
                                    +------------+    +------------+
```

The above state diagram shows 4 c0_kvmultisets, 2 versions of the LC structure, and 2 cN kvsets:
- c0_kvmultisets are labeled *c0kvms[n]*, where *n* indicates the
  c0_kvmultiset dgen.
- LC versions are labeled *LC[n]*, where *n* indicates the state of LC
  immediately after ingesting *c0kvms[n]*.
- cN kvsets are labeled *cnkvset[n]*, where *n* indicates the cN kvset dgen.

Note that *c0kvms[4]*, *c0kvms[3]*, etc. are distinct instances of c0_kvmultiset objects
and *c0kvset[2]*, *c0kvset[1]*, etc. are distinct instances of c0_kvmultiset objects,
but *LC[2]* and *LC[1]* are different views of the same instance of LC.

The annotations *k/18..13*, *k/13..9*, etc indicate the range of
sequence numbers for non-txn kv-tuples in a c0_kvmultiset, LC or cN
object.

Txn kv-tuples are indicated by *t1:a/\** where *t1* represents the
transaction, and *a/\** represents a kv-tuples that has not been
assigned a sequence number.  When *t1* is committed, a sequence number is
assigned and the annotation would be *t1:a/15* if the sequence number
were 15.

The search path is shown with arrows.  In the above diagram, the search path is:
```
c0kvms[4] -> c0kvms[3] -> LC[2] -> cnkvset[2] -> cnkvset[1]
```
which is usually abbreviated as follows:
```
c0kvms[4..3] -> LC[2] -> cnkvset[2..1]
```

KVS get requests will search each object in the search path.

Cursors will obtain references on each each object in the search path.

Objects not on the search path exists because cursors have references on them.  They would
be destroyed when the last reference is dropped.

### Invariants

This solution depends on the following invariants:
- **INV_01**: All non-txn kv-tuples in *c0kvms[i]* have sequence numbers less than or equal to all
  non-txn kv-tuples in *c0kvms[i+1]*.
- **INV_02**: All kv-tuples in *cnkvset[i]* have sequence numbers less than or equal to all
  kv-tuples in *cnkvset[i+1]*.
- **INV_03**: Search paths contain each kv-tuple in the KVDB exactly once.
  - This is stronger than necessary, but it is hygienic and doesn't appear difficult to maintain.

### Details

TBD.

## Failure and Recovery Handling                                       |

<!--How your RFC intends to handle failures-->

## Testing Guidance

<!--
  How would someone be able to confirm your RFC works assuming it is implemented
-->
