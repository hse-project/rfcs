# LC: A data structure to ensure uncommitted kv-tuples are not ingested into cN

## Problem

One of the invariants in cN is that newer kvsets contain newer data.
Since cN kvsets are searched in order from newest to oldest, this
invariant allows cN queries to terminate on the first kvset with a
suitable match.

Recent changes in c0[1] allow transactions to store uncommitted
kv-tuples in c0 kvsets.  Uncommitted kv-tuples do not yet have
sequence numbers. If they were ingested into cN and later assigned a
sequence number, the assigned number would be higher than sequence
numbers for other unrelated kv-tuples that have not yet been ingested
into cN.  This would violate the invariant.

## Requirements

- Design must not prevent concurrent ingest operations

## Non-Requirements

## Solution

### Overview

First, some terminology:

- "Ingest" refers the process of migrating kv-tuples from the oldest
  c0 kvset into cN, at which point it becomes the newest cN kvset.
- A "txn kv-tuple" is a a key-value pair created by a transaction.
- A txn kv-tuple can be active, committed or aborted (i.e., in the
  same state as the transaction that created it).

This RFC proposes a new data structure which will be referred to as
the "Late Commit" structure, or simply "LC".

There will be one LC per KVDB.

During ingest, active txn kv-tuples will be pulled aside and stored in
LC instead of being ingested into cN.  Over time, as transactions are
aborted and committed, related kv-tuples in LC will become aborted and
committed.  Thus, ingest will also consume from LC to move these
committed kv-tuples into cN.  LC kv-tuples that have been aborted or
ingested into cN will need to be garbage collected.

General operation:
- Ingest will store active txn kv-tuples in LC.
- Ingest will scan LC entries in key-order so that committed kv-tuples
  can be copied into cN.  After copying, the LC kv-tuple will be
  marked so that it will not be ingested again.
- Txn "get" requests and txn cursors will search LC for kv-tuples
  belonging to the txn.
- Non-txn "get" requests and non-txn cursors will search LC for
  kv-tuples in the committed state.

### Details

TBD.

## Failure and Recovery Handling

<!--How your RFC intends to handle failures-->

## Testing Guidance

<!--
  How would someone be able to confirm your RFC works assuming it is implemented
-->

## Footnotes

[1] After release 1.9.0, transaction-private kvsets were eliminated
and transactions now store kv-tuples directly in the active c0 kvset.
This was part of the "ingest pipeline" work that also eliminated c1
and replaced it a write-ahead log.
