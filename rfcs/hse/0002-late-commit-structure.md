
# Ensuring transaction kv-tuples are committed before ingesting into cN

## Problem

One of the invariants in cN is that newer kvsets contain newer data.
Since cN kvsets are searched in order from newest to oldest, this
invariant allows cN queries to terminate on the first kvset with a
suitable match.

Recent changes in c0 allow transactions to store uncommitted kv-tuples in
"struct c0_kvmultiset" objects long enough to be considered for ingest into
cN.  Uncommitted kv-tuples do not yet have sequence numbers. If they were
ingested into cN and later assigned a sequence number, the assigned number
would be higher than sequence numbers for other kv-tuples that have
not yet been ingested into cN.  This would violate the invariant.

## Requirements

- Design must not prevent concurrent ingest operations

## Non-Requirements

- Design must not ingest uncommitted data

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

Txn reads will search LC for uncommitted kv-tuples belonging to the txn as
well as for committed kv-tuples produced by other txns.  Non-txn reads will
search LC for committed kv-tuples.  Both txn and non-txn reads will provide a
view seqno when searching LC.

### Terminology

- **_kv-tuple_** -- A key and an associated value, tombstone or prefix
  tombstone.  Note that (prefix) delete operations are mutations that
  add kv-tuples with (prefix) tombstones to the KVDB rather than
  actually removing a kv-tuple.
- **_c0kvms (struct c0_kvmultiset)_** -- A collection of kv-tuples in c0.
  Each KVDB has one **_active c0kvms__** and zero or more **_frozen
  c0kvms_**'s.  The active c0kvms stores incoming kv-tuples (from HSE puts,
  deletes, etc.).  When the active c0kvms becomes "full", it is frozen and a new
  active c0kvms is created.
- **_ingest_** refers the process of migrating kv-tuples from the oldest
  c0kvms into cN, at which point it becomes the newest cN kvset.
- **_dgen_** - A data generation number assigned to each
  c0kvms.  The initial active c0kvms is assigned a dgen of 1.
  Each new active c0kvms is assigned a dgen one greater than
  the dgen of the previous active c0kvms.  Dgen numbers persist
  across KVDB open/close so they are never reused for the
  life of a KVDB.  Dgen numbers follow data into cN.  cN kvsets
  created during ingest are associated with a single dgen (the same
  dgen as the ingested c0kvms).  cN kvsets created during
  compaction of M kvsets with dgen numbers *x* to *x\+M\-1* are
  associated with that range of dgen numbers.
- A **_txn kv-tuple_** is a a key-value pair created by a transaction.
  A txn kv-tuple can be active, committed or aborted (i.e., in the
  same state as the transaction that created it).

Each c0kvms has a dgen number assigned when the c0kvms is made active. Dgen
numbers are unique across the life of a KVDB and can be used to identify a
particular c0kvms instance.
- **_c0kvms[d]_** : identifies the c0kvms with dgen *d*
- **_c0kvms[d].txn_seqno_frozen_** : the commit seqno of a txn that was committed
  before c0kvms[d] was frozen.
- **_c0kvms[d].txn_seqno_min_** : alias for *c0kvms[d-1].txn_seqno_frozen*, or *0* if *d==1*.

For each kv-tuple *kv*, define:
- **_is_txn(kv)_**       : true if and only if *kv* originated in a transaction

For kv-tuples *kv* such that *is_txn(kv) == true*:
- **_kv.txn_**         : the transaction that originated *kv*
- **_kv.c0snr_**       : the c0snr entry associated with *kv.txn*
- **_kv.c0snr.dgen_**  : the max dgen of all c0kvms's containing kv-tuples associated with *kv.c0snr*
- **_kv.ingested_**    : true if *kv* is in LC and has already been ingested

The following definitions are defined so they apply in a logical way to
non-txn and txn kv-tuples:
- **_is_active(kv)_**    : *is_txn(kv)* and *kv.txn* is active
- **_is_aborted(kv)_**   : *is_txn(kv)* and *kv.txn* has been aborted
- **_is_committed(kv)_** : *!is_txn(kv)* or *kv.txn* has been committed

All committed kv-tuples have a concrete sequence number.  For txn kv-tuples, it is assigned
when the transaction commits.  For non-txn kv-tuples, it is assigned when the kv-tuple in
inserted into the active c0kvms.
- **_kv.seqno_** : if *is_committed(kv)*, the concrete sequence number assigned to *kv*, else undefined

For all kv-tuples *kv*, define:
- **_overlap(kv1,kv2)_** : *kv1* and *kv2* have the same key, or one is a prefix
  tombstone that matches the other's key

Each KVDB has single LC object, where:
- **_LC.entries_**: a set of kv-tuple objects
- **_LC.view_seqno_min_**: a lower bound on the view seqno used by current and
  future LC queries, increases monotonically over time

### Seqno Ordering

In release 1.9, given two cN kvsets, every kv-tuple in the newer kvset
has a seqno greater than or equal to every kv-tuple in the older kvset.
This RFC relaxes this so the ordering only applies to keys that overlap.
Informally, the new ordering invariant is:  given two kv-tuples in the same kvs, the kv-tuple in
the newer c0kvms/LC/cnkvset has a larger sequence number than the
kv-tuple in older c0kvms/LC/cnkvset.

More formally, given a kv-tuple *kv*, define **_container(kv)_** to be the
c0kvms or LC or cN kvset that contains *kv*.  Define **_newer than_** as a
total order an all containers:

*c1* is newer than *c2* if an only if:
- *c1* is a c0kvms and *c2* is LC, or
- *c1* is LC and *c2* is a cN kvset, or
- *c1* has a higher dgen number than *c2*.

The new seqno ordering invariant can now be stated as follows:

Given two kv-tuples *kv1* and *kv2*, if:
- *kv1* and *kv2* are in the same KVS, and
- *overlap(kv1, kv2) == true*, and
- *!is_txn(kv1) || is_committed(kv1)*, and
- *!is_txn(kv2) || is_committed(kv2)*, and
- *container(kv1)* is newer than *container(kv2)*,

Then:
- *kv1.seqno >= kv2.seqno*.

### Garbage Collection

An entry can be removed from LC when it no longer needed for LC
queries.  Let *kv* be an entry in LC.  Then *kv* can be removed
from LC if and only if *kv* has been ingested into cN
and *kv.seqno < LC.view_seqno_min*, or if *kv* has been aborted.

Garbage collection will be implemented as part of the ingest
operation.

### Ingest

The ingest operation must take care not to break transaction atomicity.  Prior
to the ingest pipeline work, atomicity was ensured because all kv-tuples for a
transaction were part of the same ingest operation and ingest operations
themselves are atomic.  In the new design, ingest operations are still atomic,
but (1) the kv-tuples in the c0kvms being ingested can flip to the committed state
while the ingest operation is in progress, and (2) a transaction’s kv-tuples
can be spread among multiple c0kvms objects.

The first issue is handled by ensuring all ingested transaction
kv-tuples are committed and have a seqno less or equal the seqno of
any recently committed transaction (see *c0kvms[d].txn_seqno_frozen*).

There are two ways to deal with the second issue depending on use of the WAL:
- *(USE_WAL==1)* Rely on knowledge that the WAL has persisted the
  transaction's commit and all of its kv-tuples, or
- *(USE_WAL==0)* Ensure all of a transaction's kv-tuples are in the
  input c0kvms and LC before ingesting any of the transaction's
  kv-tuples.

We use the second approach (*USE_WAL==0*) approach because it provides an easy way
to "assign" each transaction to an ingest operation, which is helpful for supporting
concurrent ingest operations.  We can revisit this when the WAL is available to more
efficiently support large transactions.

Algorithm for ingesting c0kvms with dgen d:

```
ingest(d) {

  Inputs:
    c0kvsm[d];
    LC;

  Outputs:
    cnkvset[d]
    LC;

  Algorithm:

    assert(c0kvms[d].txn_seqno_frozen >= c0kvms[d].txn_seqno_min);
    Create instance cnkvset[d], initally empty;
    Let new_lc_entries be a list of kv-tuples, initally empty;
    Let merge(A, B) represent the union of kv-tuples in A and B sorted by key;

    For each kv in merge(c0kvms[d], LC) {

        kv_from_kvms = (kv came from c0kvms[d]) ? true : false;
        ingest = false;

        if (kv_from_kvms) {
            if (!is_txn(kv)) {
                ingest = true;
            } else if (is_aborted(kv)) {
                ; // Do nothing.
            } else if (is_active(kv)) {
                add kv to new_lc_entries;
            } else {
                assert(is_committed(kv));
                assert(kv.seqno > c0kvms[d].txn_seqno_min);
                if (kv.seqno <= c0kvms[d].txn_seqno_frozen) {
                    // We have all kv-tuples for this txn, so it is safe to ingest.
                    ingest = true;
                } else {
                    // This will be handled by a future ingest.  Note this forces
                    // ingest operations to be serialized, but we have a fix to
                    // enable concurrent ingest.  We can split this for-loop into
                    // two parts: part 1 would be serialized and would include
                    // modifications to LC, part 2 would be concurrent and would
                    // include creating the output cnkvset.
                    add kv to new_lc_entries;
                }
            }
        }
        else {
            assert(is_txn(kv));
            if (is_aborted(kv)) {
                kv.gc = true;
            } else if (is_active(kv)) {
                ; // do nothing
            } else {
                assert(is_committed(kv));
                beyond_horizon = kv.seqno < LC.view_seqno_min;
                if (kv.seqno <= c0kvms[d].txn_seqno_min) {
                    // An old txn, might have been left in LC due to seqno horizon.
                    // If it has been ingested and beyond horizon, mark it for GC.
                    // Note the beyond_horizon check could be done in the GC task
                    // instead of here.
                    if (kv.ingested && beyond_horizon && !kv.gc)
                        kv.gc = true;
                } else if (kv.seqno <= c0kvms[d].txn_seqno_frozen) {
                    // This txn has been "assigned" to this ingest operation.
                    // In addition, we have all kv-tuples for this txn, so it
                    // is safe to ingest.
                    ingest = true;
                    kv.ingested = true;
                    kv.gc = beyond_horizon;
                } else {
                    ; // Do nothing.  This txn will be handled by a future ingest.
                }
            }
        }

        if (ingest) {
            add kv to cnkvset[d];
        }
    }

    // After the output cnkvset has been published:

    get LC write lock;
    for each (kv in new_lc_entries)
        lc_add(LC, kv);
    release LC write lock;
}

                -----------------------------
                Ingest and Garbage Collection
                -----------------------------
```

### LC Data Structure and API

LC will be implemented as an RCU bonsai tree backed by dynamic memory
allocation instead of by a *cheap*.

LC API:
```
lc_create();
lc_destroy();

// Point queries
lc_get();
lc_pfx_probe();

// Cursor support
// - These APIs mimic cursor APIs for c0 and cN and should plug right
//   into the existing KVS cursor implementation.  LOL.
lc_cursor_create();
lc_cursor_destroy();
lc_cursor_read();
lc_cursor_restore();
lc_cursor_seek();
lc_cursor_update();
```

## Failure and Recovery Handling

- *lc_create()* will be called during *hse_kvdb_open()*.  If *lc_create()* fails,
  then *hse_kvdb_open()* will fail.
- The ingest operation can fail due to problems with memory allocation or media
  writes.  In either case, the system must be shutdown.  This design does not
  provide a way to back out of partially completed ingestg operation.

## Testing Guidance

A good test workload consists of long-lived transactions that insert
kv-tuples along with a moderate to high rate of non-txn put
operations.  The non-txn put operations will force ingests, and the
long-lived transactions will force use of LC.
