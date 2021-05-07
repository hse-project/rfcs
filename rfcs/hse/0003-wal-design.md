# !!!! DRAFT - PLEASE DO NOT REVIEW !!!!

# WAL Logical Design

## Problem

The data in c0 is volatile and is lost in the event of a crash or power loss.

## Requirements

- Provide durability to c0 data, i.e., honor both time and space-based durability
  guarantees
- Provide support for hse_kvdb_flush() and hse_kvdb_sync() API invoked by the app
- Maintain atomicity, consistency and isolation (ACI) semantics for transactional
  mutations during wal replay
- The performance of HSE with WAL must match or exceed 1.9.0 performance

## Non-Requirements

- Need not support synchronous (durable) transaction commits. The application can
  follow transaction commits with an explicit hse_kvdb_sync() for durability.
- WAL is not queried on a live KVDB and hence its in-memory representation need
  not be search friendly

## Solution

### Overview

### Terminology

- **_tx-write_** -- put, del and pfx-del in the context of a transaction

- **_nontx-write_** -- put, del and pfx-del without any transactional context

- **_tx-KVS_** -- A key-value store that accepts only transactional writes

- **_nontx-KVS_** -- A key-value store that accepts only non-transactional writes

- **_timestamped KVS_** -- A key-value store that orders kv-tuples based on an
  application provided timestamp

- **_non-timestamped KVS_** -- A key-value store that orders kv-tuples based on
  HSE minted logical sequence number

- **_dgen_** -- A monotonically increasing generation number assigned to each
  c0_kvmultiset

- **_durability-interval_** -- All kv-mutations that occur before the configured
  durability interval (in ms) must not be lost in the event of a crash

- **_durability-size_** -- All kv-mutations that got ingested prior to the last
  ingested 'durability-size' worth of contents must not be lost in the event
  of a crash

- **_WAL buffer_** -- A circular buffer used to stage c0 data before persisting it
  into WAL

- **_durable-boundary_** -- A WAL buffer offset below which all data has been
  persisted into WAL. The offset range (next_woff, _durable-boundary_) can be
  safely reclaimed.

- **_flush-boundary_** -- A sequential WAL buffer offset range in which all data
  has been filled by the client threads, i.e., there are no holes/gaps. This offset
  range (_durable-boundary_, _flush-boundary_) is ready to be persisted into WAL.


### WAL Components


#### WAL MDC

There is one WAL MDC per kvdb to track WAL metadata.

Here are few things that will be recorded in the WAL MDC:

- A version record to track WAL version
- An ingest record for each KVS to track ingested dgen
- WAL config record to track WAL capacity, durability parameters etc.
- A close record to indicate graceful kvdb shutdown


#### WAL Data Buffers

The kv-mutations from the client threads are first staged in the WAL buffers
before it is persisted on media.

The following are the desirable properties for the WAL buffers:

- WAL buffers must not be a central point of contention for the client threads.
- The total WAL buffer capacity must have an upper bound. If the WAL buffer usage
  reaches certain high watermarks, the application threads should be throttled.
- WAL buffers and c0 can share the same in-memory buffers which cuts down an
  additional memory copy of the DB data.
- WAL buffers should be NUMA aware to reduce access latencies


#### WAL files

The WAL buffers are flushed into the WAL files under the following circumstances:

- Invocation of hse_kvdb_sync()/flush() by the client threads
- On meeting either the durability interval or durability size based thresholds
  defined for a kvdb.

*How many WAL files?*

It has to be certainly more than one WAL file for parallelism. Also, having
multiple files will simplify the reclamation of WAL disk space.

A proposal is to have one or more WAL files exclusive to a dgen. An advantage
with this approach is that it is easier to determine which files to discard
post cN ingest.  If a dgen is backed by more than one WAL file, the wal files
can be named as *wal_\<dgen\>_\<id\>*.


#### WAL Threads

At a high level, the WAL implementation will contain the following thread types:

- A timer thread which keep tracks of both durability interval and size
- One or more log flusher threads, ideally one per active WAL buffer, to update
  the flush boundary in its WAL buffer and to queue IO requests
- One or more log writer threads which dequeues the IO requests and writes data
  to the WAL files. The writer threads also update the durable boundary of the
  WAL buffer that it worked on.
- A write-notifier thread which wakes up the client threads that issued sync and
  are waiting for the durable boundary to reach a sync boundary.
- A log reclamation thread that reclaims wal files post cN ingest
- One or more WAL replay threads employed for WAL replay


#### WAL Invariants


##### Non-transaction writes

A non-transactional write can be issued *only* to a non-transactional KVS.
There can be be more than one non-transactional KVS in a KVDB.

_Invariants:_
- **I1:** The ordering for non-transactional writes within a single thread must be
  preserved during WAL replay.
- **I2:**  In the event of a crash, all non-tx writes that occur within a single
  thread must be replayed *only* up to some checkpoint and nothing beyond that.
- **I3:** The ordering for non-tx writes that are carefully ordered by cooperating
  threads, each thread modifying either unique or overlapping set of keys, must be
  preserved during WAL replay.
- **I4:** The ordering for non-tx writes issued concurrently by non-cooperating
  threads, each thread modifying either unique or overlapping set of keys, may not
  be preserved during WAL replay.
- **I5:** The ordering for non-tx writes with respect to the other simultaneously
  occuring non-tx/tx writes to one or more non-tx/tx KVSes respectively cannot
  be preserved during WAL replay.


##### Transaction writes

A transactional write can be issued *only* to a transactional KVS.
There can be be more than one transactional KVS in a KVDB.

_Invariants:_
- **I6:** The ordering for transactional writes to one or more txn-KVSes must be
  preserved during WAL replay, be it from a single thread or from multiple
  threads.
- **I7:** In the event of a crash, all tx writes must be replayed *only* up to
  some checkpoint and nothing beyond that.
- **I8:** The ordering for tx writes with respect to the other simultaneously
  occuring non-tx writes to one or more non-tx KVSes cannot be preserved during
  WAL replay.


#### WAL Logging Path


#### WAL Replay Path


### WAL Interfaces

#### WAL Control Path

```c

/**
 * wal_create() - create WAL storage files
 *
 * - 'capacity' is an upper bound on WAL size
 * - 'mclass' specifies the media class from which to allocate WAL storage files
 * - 'walid' is an identifier returned by wal_create()
 */
merr_t wal_create(struct ikvdb *kvdb, u64 capacity, enum mpool_mclass mclass, u64 *walid);


/**
 * wal_destroy() - destroy the specified walid
 */
merr_t wal_destroy(uint64_t walid);


/**
 * wal_open() - open the specified walid
 *
 * - A dirty WAL is replayed at open time if rdonly == false
 */
merr_t wal_open(struct ikvdb *kvdb, u64 walid, bool rdonly, struct wal **wal);


/**
 * wal_props_get() - get WAL props
 */
merr_t wal_props_get(struct wal *wal, struct wal_props *props);


/**
 * wal_discard() - invoked post cN ingest by c0 to discard kv-tuples persisted by cN
 */
merr_t wal_discard(struct wal *wal, u64 ingest_dgen);


/**
 * wal_sync() - sync WAL buffers to media
 */
merr_t wal_sync(struct wal *wal);


/**
 * wal_flush() - asynchronously write WAL buffers to media
 */
merr_t wal_flush(struct wal *wal);


/**
 * wal_close() - close the specified WAL handle
 */
merr_t wal_close(struct wal *wal);

```
#### WAL Data Path

#### Non-Transaction writes

```c
/**
 * wal_put/del/del_pfx() - log non-tx mutations
 *
 * This interface is called post value compression and a successful c0 update.
 * Handles non-tx mutations in both a timestamped and a non-timestamped KVS.
 *
 * 'cnid': KVS identifier
 *
 * 'dgen': c0kvms identifier
 *   - dgen temporally orders non-tx mutations across c0kvmses during WAL replay. Useful for
 *     aggregating non-tx mutations that belong to a c0kvms and spread across multiple WAL files.
 *   - Used to establish c0kvms boundary during WAL replay
 *
 * 'seqno':
 *   - For a timestamped KVS, seqno is the application provided timestamp
 *   - For a non-timestamped KVS, seqno is the logical seqno assigned by HSE
 *
 */
merr_t
wal_put(struct wal *wal, u64 cnid, u64 dgen, u64 seqno, const struct kvs_ktuple *kt,
        const struct kvs_vtuple *vt);

merr_t
wal_del(struct wal *wal, u64 cnid, u64 dgen, u64 seqno, const struct kvs_ktuple *kt);

merr_t
wal_del_pfx(struct wal *wal, u64 cnid, u64 dgen, u64 seqno, const struct kvs_ktuple *kt);
```

#### Transaction writes

```c
/**
 * wal_txn_begin() - log WAL begin record for a kvdb txn begin
 *
 * 'txid':
 *   - For a non-timestamped KVS, txid corresponds to the view_seqno of this txn
 *     view_seqo = atomic64_fetch_add(1, kvdb_seqno);
 *   - For a timestamped KVS, HSE should mint a unique txid for each txn.
 *     The view_seqno cannot serve as a txid as an application can request a older read
 *     snapshot for a txn.
 *   - The logging of begin record can be delayed until the first txn write
 */
merr_t wal_txn_begin(struct wal *wal, u64 txid, u64 cnid);


/**
 * wal_txn_put/del/del_pfx() - log tx mutations into wal
 *
 * This interface handles both timestamped and non-timestamped KVS.
 *
 * 'txid': transaction id established at txn begin
 *
 * 'dgen':
 *   - For transactions that span multiple c0kvmses, WAL could determine the c0kvms boundary
 *     at replay time using either (1) or (2):
 *       (1) highest dgen in all the kv tuples for a given txn
 *       (2) span a txn replay across multiple kvmses determined by each kv-tuple's dgen
 *
 * 'ts':
 *   - For a timestamped KVS, ts is the commit timestamp established prior to a subset of tx puts
 *   - For a non-timestamped KVS, ts is 0. The seqno is established at txn commit time.
 */
merr_t
wal_txn_put(struct wal *wal, u64 txid, u64 dgen, u64 ts, const struct kvs_ktuple *kt,
            const struct kvs_vtuple *vt);

merr_t
wal_txn_del(struct wal *wal, u64 txid, u64 dgen, u64 ts, const struct kvs_ktuple *kt);

merr_t
wal_txn_del_pfx(struct wal *wal, u64 txid, u64 dgen, u64 ts, const struct kvs_ktuple *kt);

/**
 * wal_txn_commit() - log wal commit record for a kvdb txn commit
 *
 * 'txid': transaction id established at begin
 *
 * 'seqno':
 *   - For a timestamped KVS, seqno = default timestamp, i.e., those txn mutations not
 *     associated with a commit timestamp is assigned with 'seqno' at replay time
 *   - For a non-timestamped KVS, this interface binds 'txid' with 'seqno'
 *
 */
merr_t wal_txn_commit(struct wal *wal, u64 txid, u64 seqno);


/**
 * wal_txn_abort() - log WAL abort record for a kvdb txn abort
 *
 * At replay time, WAL can safely discard all mutations associated with 'txid'
 *
 * 'txid': transaction id established at begin
 *
 * [TODO]: Evaluate if abort API is needed, as the absence of commit record implies an abort.
 *
 * Transactions can be aborted from two places - (1) Client threads (2) LC
 *
 * Does it simplify WAL buffer and/or disk space reclamation if WAL knows about txn aborts?
 *
 * Seems not. By the time an active txn kv-pair is populated in LC, the kv-pair is already
 * in the WAL buffer. Invoking wal_sync() prior to cN ingest ensures that this kv-pair is
 * also persisted in WAL. So, the WAL-buffer and disk space consumed by an ingested dgen
 * can be reclaimed safely without an txn abort hint from either the client or LC.
 */
merr_t wal_txn_abort(struct wal *wal, u64 txid);

```

## Failure and Recovery Handling


## Testing Guidance
