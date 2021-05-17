# !!!! DRAFT - PLEASE DO NOT REVIEW !!!!

# WAL Logical Design

## Problem
The data in c0 is volatile and is lost in the event of a crash or power loss.

## Requirements
- Provide durability to c0 data, i.e., honor both time and space-based durability
  constraints
- Maintain atomicity, consistency and isolation (ACI) semantics for transactional
  mutations during WAL replay
- Provide support for hse_kvdb_flush() and hse_kvdb_sync() HSE API
- The performance of HSE with WAL must match or exceed 1.9.0 with c1
- Provide an option to disable WAL for the entire KVDB

## Non-Requirements
- Support for synchronous (durable) transaction commits. The application can
  follow transaction commits with an explicit hse_kvdb_sync() for durability.
- Support for timestampled-KVS, although hooks are added wherever possible to
  accommodate it

## Solution

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

- **_LSN_** -- Log Sequence Number, a monotonically increasing timestamp attached
  to each kv-mutation

- **_durability-interval_** -- All kv-mutations that occur before the configured
  durability interval (in ms) must not be lost in the event of a crash

- **_durability-size_** -- All kv-mutations that got ingested prior to the last
  ingested 'durability-size' worth of contents must not be lost in the event
  of a crash

- **_WAL buffer_** -- A circular buffer used to stage c0 data before persisting it
  into WAL

- **_durable-boundary_** -- A WAL buffer offset below which all data has been
  persisted into WAL. The offset range (cur-pos, _durable-boundary_) can be
  safely reclaimed.

- **_flush-boundary_** -- A sequential WAL buffer offset range in which all data
  has been filled by the client threads, i.e., there are no holes/gaps. This offset
  range (_durable-boundary_, _flush-boundary_) is ready to be persisted into WAL.


### WAL Invariants

#### Non-transaction writes

A non-transactional write can be issued *only* to a non-transactional KVS.
There can be be more than one non-transactional KVS in a KVDB.

_Invariants:_
- **I1:** The ordering for non-transactional writes to the same key or different
  keys within a single thread must be preserved during WAL replay.
- **I2:**  In the event of a crash, all non-tx writes that occur within a single
  thread must be replayed *only* up to some checkpoint and nothing beyond that.
- **I3:** The ordering for non-tx writes that are carefully ordered by cooperating
  threads, each thread modifying either unique or overlapping set of keys, must be
  preserved during WAL replay.
- **I4:** The ordering for non-tx writes issued concurrently by non-cooperating
  threads, each thread modifying either unique or overlapping set of keys, may not
  be preserved during WAL replay.
- **I5:** The ordering for non-tx writes with respect to the other simultaneously
  occurring non-tx/tx writes to one or more non-tx/tx KVSes respectively cannot
  be preserved during WAL replay.
- **I6:** Issuing WAL sync prior to cN ingest ensures that all non-tx kv-tuples
  are persisted in WAL

#### Transaction writes

A transactional write can be issued *only* to a transactional KVS.
There can be be more than one transactional KVS in a KVDB.

_Invariants:_
- **I7:** The atomicity of transactional writes issued to one or more txn-KVSes
  must be preserved during WAL replay.
- **I8:** The transactional writes must be ordered by their commit seqno during
  WAL replay.
- **I9:** In the event of a crash, all tx writes must be replayed *only* up to
  some checkpoint and nothing beyond that.
- **I10:** The ordering for tx writes with respect to the other simultaneously
  occurring non-tx writes to one or more non-tx KVSes cannot be preserved during
  WAL replay.
- **I11:** Issuing WAL sync prior to cN ingest ensures that all committed tx
  kv-tuples are persisted in WAL


### WAL MDC

There is one WAL MDC per KVDB to track WAL metadata.

Here are some record types present in the WAL MDC:
```
WAL_MDC_VERS:    version record to track WAL version
WAL_MDC_INGEST:  ingest record to track ingested dgen
WAL_MDC_RECLAIM: reclaim record to track the last WAL file dgen that was reclaimed
WAL_MDC_CONFIG:  config record to track WAL capacity, durability parameters etc.
WAL_MDC_CLOSE:   close record to indicate graceful KVDB shutdown
```


### WAL Data Buffers

The kv-mutations from the client threads are first staged in an WAL buffer
before persisted on media.

The following are the desirable properties for WAL buffers:

- WAL buffers must not be a central point of contention for the client threads.
- The total WAL buffer capacity must have an upper bound. If the WAL buffer usage
  reaches certain high watermarks, the application threads should be throttled.
- WAL buffers and c0 can share the same in-memory buffers which cuts down an
  additional memory copy of the DB data.
- WAL buffers should be NUMA aware to reduce access latencies


### WAL Threads

At a high level, the WAL implementation will contain the following thread types:

- A timer thread which keep tracks of both durability interval and size
- One or more log flusher threads, ideally one per active WAL buffer, to update
  the flush boundary in its WAL buffer and to queue IO requests
- One or more log writer threads which dequeues the IO requests and writes data
  to the WAL files. The writer threads also update the durable boundary of the
  WAL buffer that it worked on.
- A sync-notifier thread which wakes up the client threads that issued sync and
  are waiting for the durable boundary to reach a sync boundary.
- A log reclamation thread that reclaims WAL files post cN ingest
- One or more WAL replay threads employed for WAL replay

#### WAL Threads Workflow

```
Ttimer: wal_timer()
{
    while (!timer_stop) {
        if (timer_exipred || kv-size(active_buflist) > durability-size)
            for each buf in active_buflist
                wakeup Tlog-flusher(buf);
    }
}

Tlog-flusher: log_flush(buf)
{
    while (!flusher_stop) {
        while (not notified for flush)
            wait to be signaled;

        /* Determine record boundary for the next flush */
        flush_start = current flush_boundary(buf);
        flush_end = cur_write_pos(buf);

        while (flush_boundary hasn't reached flush_end) {
            while (next_op is a hole)
                wait for next_op to finish c0 update
            advance flush_boundary by size(next_op);
        }

        rg_payload = buf[flush_start, flush_end]
        alloc and format rg_header in a separate buf
	record_group = sg list [rg_header, rg_payload]
        enqueue(log_writer_queue(buf), record_group)

        wakeup Tlog-writer(buf);
    }
}

Tlog-writer: log_write(buf)
{
    while (!writer_stop) {
        while (request queue is empty)
            wait for work;

        fd = wal_fd(buf);
        rg = dequeue(log_writer_queue(buf));
        write(fd, rg) and sync(fd);
        set durable_boundary(buf) to flush_boundary(buf);

        wakuep Tsync-notifier(buf);
    }
}

Tsync-notifier: sync_notify(buf)
{
    while (sync waiters_list is not empty) {
        For each waiter in sync_waiters_list(buf)
            if (sync_boundary(waiter) <= durable_boundary(buf))
                sem_post(waiter);
    }
}

```


### WAL files

The WAL buffers are flushed to the WAL files under the following circumstances:

- Invocation of hse_kvdb_sync()/flush() by the client threads
- On meeting either the durability interval or durability size constraints
  defined for a KVDB.

*How many WAL files?*

It has to be certainly more than one WAL file for parallelism. Also, having
multiple files will simplify the reclamation of WAL disk space.

A proposal is to have one or more WAL files exclusive to a dgen. An advantage
with this approach is that it is easier to determine which files to discard
post cN ingest.  If a dgen is backed by more than one WAL file, the WAL files
can be named as *wal_\<dgen\>_\<id\>*.

#### WAL file format

- An WAL file is a collection of record groups. Each record group is a collection
  of one or more records (kv-mutations) accumulated in an WAL buffer in one
  durability/sync interval.

- Each WAL file contains a 4K-header followed by zero or more record groups.
  The txpinfo section is optional.
```
  ------------------------------------------------------------------------
  |       |       |      |        |       |      |      |      |         |
  | vers  | magic | dgen | txpoff | cksum | rg 1 | .... | rg n | txpinfo |
  |       |       |      |        |       |      |      |      |         |
  ------------------------------------------------------------------------
                                    WAL File

  dgen:     dgen of the c0kvms instance
  txpoff:   txpinfo section start offset
  cksum:    checksum of the preceding fields
  rg[1..n]: record groups
  txpinfo:  pending txids written after an WAL file is frozen

```

- A record group is a collection of records demarcated by the record type
  WAL_TYPE_EORG. The records in an record group are variably sized. The size
  of a record depends on the size of the kv-mutation (put, del, pfx-del)
  logged by this record. All record header fields are 8B in length.

```
  --------------------------------------------------------------------------------------
  |       |       |      |       |       |       |      |        |     |       |       |
  | rlen  | rtype | LSN  | rec 1 | cksum | ...   | rlen | rtype  | LSN | rec m | cksum |
  |       |  (RG) |      |       |       |       |      | (EORG) |     |       |       |
  --------------------------------------------------------------------------------------
                                     WAL Records

  rlen: record size (pointer to the next record)
  rtype:
       WAL_TYPE_RG:      start/middle of a record group
       WAL_TYPE_EORG:    end of a record group
       WAL_OP_NONTX:     non-tx put/del/pfx-del record
       WAL_OP_TX:        tx put/del/pfx-del record
       WAL_OP_TX_BEGIN:  tx begin record
       WAL_OP_TX_COMMIT: tx commit record
       WAL_OP_TX_ABORT:  tx abort record
  LSN:   Log sequence number of this record
  rec m: record payload
  cksum: checksum for the whole record
```

*Key Points:*

- Both record group and records are variably sized. So, a record is never
  broken down for framing which has the following benefits -
  (1) Avoids framing related memory copies
  (2) Enables c0 to share the key and value data from the WAL buffer


### WAL Logging Path

With kv-mutations spread across multiple WAL files, the WAL component needs an
monotonically increasing log sequence number (LSN) to order kv-mutations.

The LSN must be unique across multiple WAL buffers and across KVSes in a KVDB.
The LSN captures the arrival order of kv-mutations.

```
Tclient: wal_op()/wal_txn_op()
{
    buf  = NUMA local wal buffer;
    len  = sizeof(wal_record) + klen + vlen;
    woff = atomic64_fetch_add(len, buf->off);
    lsn = atomic64_inc_return(&kvdb_lsn);

    initialize record header and WAL record at buf[woff];
    copy key and value data into the WAL record;
}

```

#### Non-transaction writes

The HSE minted logical sequence number is not sufficient to order non-tx
writes as it is not bumped for every non-tx op. WAL replay fully relies
on LSN to order non-tx kv-mutations.

For non-tx writes, the seqno is not established prior to calling ikvs_put.
The seqno can be filled in the WAL buffer by c0 once it's available, i.e.,
before returning from ikvs_put().

```
NONTX-OP <op> <cnid> <dgen> <seqno> <klen> <vlen> <kvdata>
```

#### Tansaction writes

A KVDB client txn is uniquely identified by its transaction id (TID).
For a non-timestamped KVS, the view seqno bumped at tx begin is used
as the transaction identifier.

```
TX-BEGIN  <txid>
TX-ABORT  <txid>
TX-COMMIT <txid> <commit-seqno>
TX-OP     <txid> <op> <cnid> <dgen> <seqno> <klen> <vlen> <kvdata>
```

The HSE minted commit sequence number is sufficient to order client
transactions at replay time. The LSN can be used further to order
operations within a transaction. If maintaining a global atomic for
LSN proves to be a bottleneck, then the LSN could be a per-transaction
entity for tx-mutations and the global atomic can be used purely for
non-tx mutations.

The ingest pipeline work imposes the following behavioral change on how
transactions are implemented:

(1) Transaction mutations can span across multiple c0kvmses/dgens
(2) Transactions can remain active until the client decides to commit
    it or the client/LC decides to abort it

As a result of (1) and (2), the WAL files for an ingested dgen *d* cannot
be reclaimed as it can contain active transaction kv-tuples that are
required during WAL replay in the event of a crash.

An approach to solve the above problem is as follows:

During the ingest of a c0kvms with dgen *d*, the LC is populated with
those txn kv-tuples that are either (1) active, i.e., uncommitted or
(2) committed after c0kvms(d) is frozen(**TODO:** Check this with Alex).
These kv-tuples will be resolved(committed/aborted) in a future
c0kvms(dgen >= *d+1*).

As LC is already aware of the list of active transactions in an ingest
operation, it can communicate a list of active TXIDs to WAL during
or post cN ingest. This interaction of LC with WAL is warranted, otherwise,
tracking pending txn ids in WAL adds unnecessary complexity and duplicates
state that LC already tracks. This information can be piggybacked in the
cN ingest callback (*struct kvdb_callback*) which WAL can subscribe to.

Using the callbacks from LC, WAL maintains a pending txid list in-core
for each ingested dgen. As soon as WAL realizes that all pending
txids for a dgen is resolved, the associated WAL file can be reclaimed
as this file contents has been fully ingested into cN.

WAL also persists the txn pending list of dgen *d* in the *txpinfo*
section of its associated WAL file.

During replay, the txn pending list is reconstituted from the *txpinfo*
section for all dirty WAL files whose *dgen < ingest_dgen*. If the
*txpinfo* section is not available due to a crash before this info
got completely written into an WAL file, WAL replay can reconstitute this
information by parsing all the WAL records in the WAL file of interest.

The WAL buffer space reclamation is simpler as the tx mutations are written
into WAL as they occur and WAL doesn't need to wait for the txs to resolve
prior to reclaiming it. The WAL replay path becomes a bit complicated,
however, it's worth the additional time/space complexity for a rare
crash/replay scenario.


#### cN ingest

c0 notifies WAL the dgen that was successfully ingested into cN.
WAL writes an INGEST record to its MDC recording this dgen.


### WAL Replay Path

#### Non-transaction writes

```
wal_nontx_replay(wal_file_list) {
    ingest_dgen = dgen of the last successful cN ingest;

    For each wal_file in wal_file_list {  // by dgen order
        if (dgen(wal_file) <= ingest_dgen)
            reclaim wal_file; // nothing to replay

        while ((wal_record = next_record(wal_file)) != EOL)
            add wal_record to an in-core non-tx list

        For each nontx-op in non-tx list: // sorted LSN order
            insert nontx-op into active c0kvms;

	ikvdb_flush(); // queue active c0kvms for ingest
    }
}

```

#### Transaction writes

The key difference in replaying tx-writes compared to non-tx writes is that
an WAL file whose dgen <= ingest_dgen can still contain pending transactional
mutations and hence cannot be reclaimed/skipped until those are resolved.


```
wal_tx_write(op) {
    static u64 curdgen = 0;

    if (curdgen == 0)
        curdgen = dgen(op);

    if (dgen(op) != curdgen) {
        flush(active c0kvms);
        curdgen = dgen(op);
    }

    insert op into active c0kvms;
}

wal_tx_replay(wal_file_list) {
    ingest_dgen = dgen of the last successful cN ingest;

    For each wal_file in wal_file_list {  // by dgen order
        d = dgen(wal_file);
        if (d <= ingest_dgen) {
            read tx_pending list from wal_file[txpoff];
            add txid to the in-core tx_pending list against 'd';

            while ((wal_record = next_record(wal_file)) != EOL) {
	        txid = txid(wal_record);
                if (txid in tx_pending(d))
                    add tx-op the in-core txn_list against txid;
            }
        }
    }

    For each wal_file in wal_file_list {  // by dgen order
        if (dgen(wal_file) > ingest_dgen) {
            while ((wal_record = next_record(wal_file)) != EOL)
                add tx-op to the in-core txn_list against txid;
        }
    }

    For each committed txid in txn_list { // by commit-seqno order
        For each tx-op in txid: // sorted LSN order
            wal_tx_write(tx-op);
    }

    reclaim WAL files that have been replayed successfully
}

```


### WAL Interfaces - Control Path

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

/**
 * wal_ingest_cb() - ikvdb cb registered by WAL to be invoked post cN ingest
 */
merr_t wal_ingest_cb(struct ikvdb *ikvdb, uint64_t cnid, uint64_t seqno, uint64_t status,
                     int txidc, uint64_t *txidv);
```


### WAL Interfaces - Data Path

#### Non-Transaction writes

```c
/**
 * wal_put/del/del_pfx() - log non-tx mutations
 *
 * Handles non-tx mutations in both a timestamped and a non-timestamped KVS.
 *
 * 'cnid': KVS identifier
 * 'dgen': c0kvms identifier
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
 * wal_txn_begin() - log WAL begin record for a KVDB txn begin
 *
 * 'txid':
 *   - For a timestamped KVS, HSE should mint a unique txid for each txn.
 *   - The logging of begin record can be delayed until the first txn write
 */
merr_t wal_txn_begin(struct wal *wal, u64 txid, u64 cnid);

/**
 * wal_txn_put/del/del_pfx() - log tx mutations into wal
 *
 * This interface handles both timestamped and non-timestamped KVS.
 *
 * 'txid': transaction id established at txn begin
 * 'dgen': c0kvms generation
 * 'seqno':
 *   - For a timestamped KVS, seqno is the commit timestamp established prior to a
       subset of tx puts
 *   - For a non-timestamped KVS, seqno is 0. The seqno is established at txn commit time.
 */
merr_t
wal_txn_put(struct wal *wal, u64 txid, u64 dgen, u64 seqno, const struct kvs_ktuple *kt,
            const struct kvs_vtuple *vt);

merr_t
wal_txn_del(struct wal *wal, u64 txid, u64 dgen, u64 seqno, const struct kvs_ktuple *kt);

merr_t
wal_txn_del_pfx(struct wal *wal, u64 txid, u64 dgen, u64 seqno, const struct kvs_ktuple *kt);

/**
 * wal_txn_commit() - log wal commit record for a KVDB txn commit
 *
 * 'txid': transaction id established at begin
 * 'seqno':
 *   - For a timestamped KVS, seqno = default timestamp, i.e., those txn mutations not
 *     associated with a commit timestamp is assigned with 'seqno' at replay time
 *   - For a non-timestamped KVS, this interface binds 'txid' with 'seqno'
 *
 */
merr_t wal_txn_commit(struct wal *wal, u64 txid, u64 seqno);

/**
 * wal_txn_abort() - log WAL abort record for a KVDB txn abort
 *
 * At replay time, WAL can safely discard all mutations associated with 'txid'.
 * The abort record helps in timely cleanup of per-txn resources.
 *
 * 'txid': transaction id established at begin
 */
merr_t wal_txn_abort(struct wal *wal, u64 txid);

```

## Failure and Recovery Handling

Discussed in the WAL replay path section.

## Testing Guidance

All c1 tests written to exercise various crash/replay scenarios can be used
for testing WAL.

## References

- [mysql-8-0-new-lock-free-scalable-wal-design](https://mysqlserverteam.com/mysql-8-0-new-lock-free-scalable-wal-design/)
- [Scalability of write-ahead logging on multicore and multisocket hardware](https://dl.acm.org/doi/10.1007/s00778-011-0260-8)
- [Scalable Database Logging for Multicores](http://www.vldb.org/pvldb/vol11/p135-jung.pdf)
- [Rethinking Logging, Checkpoints, and Recovery for High-Performance Storage Engines](https://db.in.tum.de/~leis/papers/rethinkingLogging.pdf)
