
# Ensuring transaction kv-tuples are committed before ingesting into cN

> TODO: This RFC is mostly complete but a few open questions remain, which are marked with "TODO"

## Problem

One of the invariants in cN is that newer kvsets contain newer data.
Since cN kvsets are searched in order from newest to oldest, this
invariant allows cN queries to terminate on the first kvset with a
suitable match.

Recent changes in c0 allow transactions to store uncommitted kv-tuples in the
active c0kvms (strut c0_kvmultiset).  Uncommitted kv-tuples do not yet have
sequence numbers. If they were ingested into cN and later assigned a sequence
number, the assigned number would be higher than sequence numbers for other
unrelated kv-tuples that have not yet been ingested into cN.  This would
violate the invariant.

## Requirements

- Design must not prevent concurrent ingest operations

## Non-Requirements

- None?

## Terminology and review

- **_kv-tuple_** -- A key and an associated value, tombstone or prefix
  tombstone.  Note that (prefix) delete operations are mutations that
  add kv-tuples with (prefix) tombstones to the KVDB rather than
  actually removing a kv-tuple.
- **_c0kvms (struct c0_kvmultiset)_** -- A collection of kv-tuples in c0.
  Each KVDB has one **_active c0kvms__** and zero or more **_frozen
  c0kvms_**'s.  The active c0kvms stores incoming kv-tuples (from HSE puts,
  gets, etc.).  When the active c0kvms becomes "full", it is frozen and a new
  active c0kvms is created.
- **_ingest_** refers the process of migrating kv-tuples from the oldest
  c0kvms into cN, at which point it becomes the newest cN kvset.
- **_dgen_** - A data generation number assigned to each
  c0kvms.  The initial active c0kvms is assigned a dgen of 1.
  Each new active c0kvms is assigned a dgen one greater than
  the dgen of the previous active c0kvms.  Dgen numbers persist
  across KVDB open/close so they are never reused for the
  life of a KVDB.  Dgen numbers follow data into cN.  cN kvsets
  creating during ingest are associated with a single dgen (the same
  dgen as the ingested c0kvms).  cN kvsets created during
  compaction of M kvsets with dgen numbers *x* to *x\+M\-1* are
  associated with that range of dgen numbers.
- A **_txn kv-tuple_** is a a key-value pair created by a transaction.
  A txn kv-tuple can be active, committed or aborted (i.e., in the
  same state as the transaction that created it).

## Solution

### Overview

This RFC proposes a new data structure referred to as the **_Late Commit_**
structure, or simply **_LC_**.  There will be one LC per KVDB.

During ingest, active txn kv-tuples will be stored in LC instead of being
ingested into cN.  As transactions are aborted and committed, kv-tuples in LC
will become aborted and committed.

The ingest operation will scan LC to copy committed kv-tuples into
cN. Ingested LC kv-tuples must eventually be garbage collected along with
aborted LC kv-tuples.

LC provides snapshot versioning so that reads have a stable view of the
kv-tuples in LC.  Snapshots are based on dgens. Each ingest operation produces
a new snapshot.  LC snapshots will not require extra kv-tuple or LC structure
copies.

Txn reads will search LC for uncommitted kv-tuples belonging to the txn as
well as for committed kv-tuples produced by other txns.  Non-txn reads will
search LC for committed kv-tuples.  Both txn and non-txn reads will provide a
view seqno and a view dgen when searching LC.  The dgen identifies that LC
snapshot being searched.

### State Diagrams

This RFC uses state diagrams as shown below to help explain how LC functions.

```

        .--  active c0kvms
        |
        V
    +----------+     +----------+     +----------+
    | c0kvms[3]|---->| c0kvms[2]|---->| c0kvms[1]|  <---- c0kvms[i] is the c0kvms
    |----------|     |----------|     |----------|        with dgen == i.
    | k/13..9  |     | k/9..4   |     | k/4..1   |
    | t1:b/*   |     |          |     | t1:a/*   |
    +----------+     +----------+     +----------+
                `.               `.
           .-->  :                :
   active  |     :   +----------+ :   +----------+
   search -'     `-->| LC[2]    | `-->| LC[1]    | <----  LC[i] is the snapshot of LC
   path              |----------|     |----------|        immediately after ingesting
                     | t1:a/*   |     | t1:a/*   |        c0kvms[i].
                     +----------+     +----------+
                          |                |
                          V                V
                     +----------+     +----------+
                     |cnkvset[2]|---->|cnkvset[1]| <----- cnkvset[i] is the cN kvset
                     |----------|     |----------|        with dgen == i.
                     | k/9..4   |     | k/4..1   |
                     +----------+     +----------+

Non-txn kv-tuples:
  - 'k/18..13', 'k/13..9', etc. indicate the range of sequence numbers for non-txn
    kv-tuples in a c0kvms, LC or cN kvset.

Txn kv-tuples:
  - Txn kv-tuples are indicated by 't1:a/*' where t1 represents the transaction, and
    'a/*' represents a kv-tuple that has not been assigned a sequence number.  When t1
    is committed, a sequence number is assigned and the annotation would be 't1:a/15' if
    the sequence number were 15.

Search paths:
  - Search paths are shown with arrows and form a directed acyclic graph on the objects.
    Multiple search paths are shown (two in this example).  All search paths start with
    the active c0kvms.  The active search path is the one that drops to the
    newest version of LC.

                  -----------------------------------------------
                  Figure 1: Example state diagram and explanation
                  -----------------------------------------------
```

Note that each *LC[i]* is a different view of the same LC object instance,
while *c0kvms[i]* and *cnkvset[i]* are distinct object instances.

Two events can change the active search path: 1) creation of a new active
c0kvms, and 2) creation of a new LC version after each ingest.

KVS get requests traverse the active search path once.

When cursors are created and updated, they obtain references on the objects in
the active search path.  As the search path evolves, cursors left holding
references on objects no longer in the active search path.

### Details

New per-KVDB state:
- *KVDB.c0kvms_active_dgen*: the dgen of the active c0kvms

New per-c0kvms state:
- Each c0kvms has a dgen number assigned when the c0kvms is made active. Dgen
  numbers are unique across the life of a KVDB and can be used to identify a
  particular c0kvms instance.  *c0kvms[d]* identifies the c0kvms with dgen *d*.
- *c0kvms[d].max_non_txn_seqno*: the maximum seqno of all non-txn kv-tuples in
  *c0kvms[dgen]*.

New LC object:
- *LC.entries*: a set of LC entries

LC entry *e* attributes:
- *e.kv*: a kv-tuple
- *e.kv.c0snr*: the c0snr associated with to *e.kv*
- *e.dgen_c0*: the dgen of the c0kvms that originally contained *e.kv*
- *e.dgen_cN*: the dgen of the cN kvset into which *e.kv* was ingested, or 0
  if not yet ingested.

Let *txn(e)* refer to the transaction that created *e*.

LC is a versioned object:
- *LC[d]* denotes the version of *LC* that existed immediately after ingesting
  *c0kvms[d]*.

Invariants:
- **I1**: All non-txn kv-tuples in *c0kvms[i]* have sequence numbers less
  than or equal to all non-txn kv-tuples in *c0kvms[i+1]*.
- **I2**: All kv-tuples in *cnkvset[i]* have sequence numbers less than or
  equal to all kv-tuples in *cnkvset[i+1]*.
- **I3**: Search paths contain each kv-tuple in the KVDB exactly once.

Some of these invariants are stronger than necessary. For example, *I1* and
*I2* could be weakened to allow out of order kv-tuples for kv-tuples with keys
that can't shadow each other.  And *I3* could be weakened to allow a search
path to contain duplicate kv-tuples.  But the stronger versions create a
system that is in my opinion significantly easier to reason about without
adding too much complexity or overhead.


#### Ingest

Let *ingest(d)* be the operation that ingests *c0kvms[d]*, creating *LC[d]*
 and *cnkvset[d]*.  A high level view of the ingest operation is shown in figure 2.

```
ingest(d) {

  Inputs:
    c0kvsm[d]; // NOT MODIFED
    LC[d-1];   // NOT MODIFED

  Outputs:
    LC[d]
    cnkvset[d]

  Algorithm:
    Create snapshot LC[d], initally empty;
    Create instance cnkvset[d], initally empty;

    Let merge(A, B) represent a the union of items in A and B sorted by key;

    For each item in merge(c0kvms[d], LC[d-1]) {
      if (is_ingestible(item, d))
        store item in cnkvset[d];
      else if (is_txn(item) && ! is_aborted(item))
        store item in LC[d];
      else
        null; // item is "dropped"
    }
}

                --------------------------
                Figure 2: Ingest operation
                --------------------------
```

In the above algorithm for *ingest(d)*, the definitions of *is_txn()* and
*is_aborted()* should be obvious.  But what about *is_ingestible()*?  When is
an item ingestible into cN?  The answer is driven by two requirements:
maintaining transaction atomicity and preserving invariant *I2*.

To maintain transaction atomicity, we must ensure that either all or none of a
transaction's kv-tuples are eventually ingested into cN.

Prior to the ingest pipeline work, atomicity was ensured because all kv-tuples
for a transaction were part of the same ingest operation and ingest operations
themselves are atomic.  In the new design, ingest operations are still atomic,
but a transaction’s kv-tuples can be spread among multiple ingest operations.

Two approaches come to mind:
1. Ensure the entire txn is ingested at once. Ingest kv-tuples belonging to
   transaction *t1* if and only if:
   - (A1.1) *t1* is committed, and
   - (A1.2) the union of *c0kvms[d]* and *LC[d-1]* contains all of *t1*’s kv-tuples.
2. Rely on the write-ahead log (WAL).  Ingest kv-tuples belonging to
   transaction *t1* if and only if
   - (A2.1) *t1* is committed, and
   - (A2.2) the WAL has persisted the *t1*’s commit and all of its kv-tuples.

This design uses the approach #1 since it requires no interaction with WAL and
it is "free" as long as invariant *I2* is required.  However, more information
will be needed with timestamped transactions that can use multiple c0snrs (LC
would need to know the maximum timestamp used by the transaction).

> TODO: Reconsider this decision.  Timestamped transactions might force us to
> weaken *I1* and *I2* as described above, and they may also force option #2.

Continuing with approach #1, condition *A1.2* is true when all kv-tuples in
all future ingests have seqnos greater than *t1*'s committed seqno.  Invariant
*I1* implies that non-txn kv-tuple seqnos between neighboring c0kvms's
can overlap by at most 1, therefore largest sequence number of the ingested
c0kvms can be used to detect the second condition as follows:

```
Ingest criteria:
    is_ingestible(kv, dgen) <==>
        (is_txn(kv) == false) ||
        (is_committed(kv.c0snr) && c0kvms[dgen].max_seqno > committed_seqno(kv.c0snr))
```

Note: Function *is_committed()* will be implemented as part of this work.


Example:
```
+----------+     +----------+     +----------+     +----------+
| c0kvms[4]|---->| c0kvms[3]|---->| c0kvms[2]|---->| c0kvms[1]|
|----------|     |----------|     |----------|     |----------|
| k/20..13 |     | k/13..9  |     | k/9..4   |     | k/4..1   |
|          |     |          |     |          |     | t1:a/3   |
|          |     |          |     |          |     | t2:b/4   |
|          |     |          |     | t3:q/10  |     | t3:p/10  |
|          |     | t4:z/*   |     | t4:y/*   |     | t4:x/*   |
+----------+     +----------+     +----------+     +----------+
            `.               `.               `.
             :                :                :
             :   +----------+ :   +----------+ :   +----------+
             `-->| LC[3]    | `-->| LC[2]    | `-->| LC[1]    |
                 |----------|     |----------|     |----------|
                 | t4:x/*   |     | t3:p/10  |     | t2:b/4   |
                 | t4:y/*   |     | t3:q/10  |     | t3:p/10  |
                 | t4:z/*   |     | t4:x/*   |     | t4:x/*   |
                 |          |     | t4:y/*   |     |          |
                 +----------+     +----------+     +----------+
                      |                |                |
                      V                V                V
                 +----------+     +----------+     +----------+
                 |cnkvset[3]|---->|cnkvset[2]|---->|cnkvset[1]|
                 |----------|     |----------|     |----------|
                 | k/13..9  |     | k/9..4   |     | k/4..1   |
                 | t3:p/10  |     | t2:b/4   |     | t1:a/3   |
                 | t3:q/10  |     |          |     |          |
                 +----------+     +----------+     +----------+

Transaction t1:
  - Ingested directly from c0kvms[1] into cnkvset[1] because at the time of
    ingest its seqno (3) < c0kvsm[1].max_non_txn_seqno (4 -- see "k/4..1").
    Txn t1 was never stored in LC.

Transaction t2:
  - Stored in LC[1] during ingest(1) because, based on its seqno, we can't
    be sure it doesn't have another kv-tuple in c0kvms[2].
  - Ingested from LC[1] to cknvset[2] during ingest(2) because its seqno (4) <
    c0kvms[2].max_non_txn_seqno (4).

Transaction t3:
  - One tuple was saved in LC[1] during ingest(1), second tuple was saved in
    LC[2] during ingest(2), both due to seqno being too low.  Both ingested
    during ingest(3).  Atomicity preserved.

Transaction t4:
  - Remains in LC since it hasn't been committed.

          -----------------------------------------------
          Figure 3: Example sequence of ingest operations
          -----------------------------------------------
```


#### Garbage Collection

An entry can be removed from LC when it is no longer needed in any LC
snapshots.  Let *e* be an entry in LC.  Then *e* can be removed from LC when:
- (GC1) *t* has been aborted,
or:
- (GC2a) *e* has been ingested into *c0kvms[d]*, and
- (GC2b) snapshots *LC[i]*, for *d < i <= e.dgen_cN*, cannot be accessed by cursors, and
- (GC2c) snapshots *LC[i]*, for *d < i <= e.dgen_c0*, cannot be accessed by point queries.

For correct garbage collection, HSE must be able to detect when the above
conditions are satisfied.  This will be implemented as follows:

- Function *is_aborted()* will be implemented to determine when *GC1* is true.
- If *e.dgen_cN > 0* then *e* has been ingested and *GC2a* is true.
- Since cursors cannot access *LC[i]* after *c0kvms[i+1]* has been destroyed, we can use
  *c0kvms* destruction a safe and reliable hint that *GC2b* is true.

> TODO: For *GC2c* (point queries), do we append *LC[i]* to the end of the RCU list of
> c0kvms's? Still thinking about this.

> TODO: Determine when events will trigger GC and on what thread garbage
> collection will take place.  It could be triggered when any of conditions
> become true, or perhaps monitoring a subset of conditions is sufficient.  It
> could also be a periodic task, or based on internal stats of LC.

#### Queries

LC queries originate from the following operations:
- Point queries:
  - hse_kvs_get
  - hse_kvs_prefix_probe
- Cursor operations:
  - hse_kvs_cursor_create
  - hse_kvs_cursor_update
  - hse_kvs_cursor_seek
  - hse_kvs_cursor_read
  - hse_kvs_cursor_seek_range

> TODO: Finish this section.

#### Task List

* Mint dgens and assign to active c0kvms
  - Must get dgen from cN now.  Should eventually persist dgen in WAL.
* Track min/max non-txn seqnos in c0kvms
  - We only need max, but it seems easy enough to initialize min to same value
    as previous c0kvms max.  Having both will allow for stronger safety checks
    in code to verify invariant *I1*.
* Update kvs_cursor to have LC cursor in addition to c0 and cN cursors.
  In theory, let it slide right into KVDB cursors as another source of
  kv-tuples.  TODO: Dig deeper into this theory.

> TODO: more tasks

## Failure and Recovery Handling

> TODO

## Testing Guidance

> TODO
