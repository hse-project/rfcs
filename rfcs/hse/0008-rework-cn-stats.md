# Rework Internal CN Tree Statistics

## Problem

Internal CN tree statistics are confusing, redundant, misleading and inefficient.

1. There is much duplication caused by the same statistic appearing in different
   structures.  For example, `key_stats.tot_vlen`, `kblk_metrics.tot_val_bytes`,
   and `kvset_metrics.tot_val_bytes` all track the same statistic.
1. `kvset_metric` and `kvset_stats` are redundant. We should pick one.
1. Many metrics don't measure what their name implies.  See *Misleading statistics*.
1. `kvset_metrics` is computed on demand, requires a node lock, and has to
   iterate over all kblocks in the kvset -- despite the fact that they never
   change for the life of a kvset.
1. Some kvset level statistics are stored in CNDB, others are stored in kblock
   headers, and others are stored in the kvset hblock.

### Misleading statistics

1. `key_stats.nvals` measures number of values associated with key.  It includes
   tombstones but not prefix tombstones.
1. `key_stats.tot_vlen`, `kblk_metrics.tot_val_bytes`,
   `kvset_metrics.tot_val_bytes`: These measure the total on-media length of all
   values associated with key/kblock/kvset.  It would be nice to have more
   detail such as: length of vals in kblocks (VTYPE_IVAL) or maybe just lengths
   of values by type (enum kmd_type).
1. `kvset_metrics.num_keys` measures number of keys. It includes keys with
   tombstones but not keys with prefix tombstones.
1. `kvset_metrics.tot_key_bytes` measures sum of key lengths. In includes keys
   with tombstones but not keys with prefix tombstones. It is also a wildly
   inaccurate measure of on-media key length since it doesn't account for
   longest common prefix elimination.  Nonetheless, it is used as if it were a
   measure of on-media key length.

## Solution

### Overview

1. Reduce the number of stat struct types.
1. Simplify on-media storage of stats.
1. Eliminate misleading stats.
1. Eliminate the need to compute kvset level stats by iterating over all kblocks in kvset.

### Details

1. Eliminate `struct kblk_metrics` and `struct kvset_metrics` use `struct kvset_stats` more widely.
   - Advantages:
     - Eliminate the overhead of computing `struct kvset_metrics`.

1. In `struct kvset_stats`, store `vlen` and `vcount` stats for each `enum
   kmd_vtype` and eliminate special named fields such as `nptombs`.
   - Advantages:
     - Eliminates all problems identified in "Misleading Statistics".

1. Store kvset-specific instances of the `one_stat_struct` in kvset hblocks
   and remove related data them from CNDB.
   - Advantages:
     - Would not have to walk all kblocks to compute kvset level stats.
     - Simplify CNDB.
     - Simplify kvset open process.

1. Store kblock-specific members of `struct kvset_stats` in kblock headers for
   use during node split.  But in the provide access to it in memory as an
   instance of `struct kvset_stats`.
   - Advantages:
       - Eliminate need for `struct kblk_metrics`
     - Dead simple to compute kvset stats from kblock stats.

1. Use `struct kvset_stats` with kvset builders instead of `struct key_stats`.
   - Advantages:
     - Eliminate need for `struct key_stats`.

1. Replace `vused` with its inverse, `vgarbage`
   - Advantages:
     - A kv-compacted kvset will have `vgarbage == 0`, which is much easier to
       work with than `vused` which has an unspecified non-zero value for nicely
       compacted kvset (it equals the vblock size minus overhead such as headers
       and padding).

## Current stat struct definitions

### struct key_stats
Used by kvset builder to track statistics about a key's values during kvset
construction.
```
struct key_stats {
    uint nvals;
    uint ntombs;
    uint nptombs;
    u64  tot_vlen;
    u64  tot_vused;
    u64  seqno_prev;
    u64  seqno_prev_ptomb;
    u64  c0_vlen;
};
```

### struct kblk_metrics
Read from kblock header during kvset_open().
```
struct kblk_metrics {
    u32 num_keys;
    u32 num_tombstones;
    u64 tot_key_bytes;
    u64 tot_val_bytes;
    u64 tot_vused_bytes;
    u32 tot_wbt_pages;
    u32 tot_blm_pages;
};
```

### struct kvset_stats
Designed to be "rolled up" to provide useful info about kvsets, nodes, KVS's
and KVDBs.  The primary struct for managing space amp.
```
struct kvset_stats {
    uint64_t kst_keys;      // number of keys
    uint64_t kst_tombs;     // number of tombtones
    uint64_t kst_halen;     // sum of mpr_alloc_cap for all hblocks
    uint64_t kst_hwlen;     // sum of mpr_write_len for all hblocks
    uint64_t kst_kalen;     // sum of mpr_alloc_cap for all kblocks
    uint64_t kst_kwlen;     // sum of mpr_write_len for all kblocks
    uint64_t kst_valen;     // sum of mpr_alloc_cap for all vblocks
    uint64_t kst_vwlen;     // sum of mpr_write_len for all vblocks
    uint64_t kst_vulen;     // total referenced data in all vblocks
    uint32_t kst_kvsets;    // number of kvsets (for node-level)
    uint32_t kst_hblks;     // number of hblocks
    uint32_t kst_kblks;     // number of kblocks
    uint32_t kst_vblks;     // number of vblocks
};
```

### struct kvset_metrics
Originally implemented to support offline tree statistics (cn_metrics.c).
```
struct kvset_metrics {
    uint64_t num_keys;
    uint64_t num_tombstones;
    uint64_t nptombs;
    uint32_t num_hblocks;
    uint32_t num_kblocks;
    uint32_t num_vblocks;
    uint64_t header_bytes;
    uint64_t tot_key_bytes;
    uint64_t tot_val_bytes;
    uint64_t tot_vused_bytes;
    uint32_t tot_wbt_pages;
    uint32_t tot_blm_pages;
    uint32_t compc;
    uint16_t rule;
    uint16_t vgroups;
};
```

### struct kvset_meta
Used by CN and CNDB to store metadata about a KVSET in CNDB.  Not really a
"stats" struct, but it contains some statistics such as "vused" and "compc".
```
struct kvset_meta {
    uint64_t km_dgen;
    uint64_t km_vused;
    uint64_t km_nodeid;
    uint16_t km_compc;
    uint16_t km_rule;
    bool     km_capped;
    bool     km_restored;
};
```

### struct cn_node_stats
Used for primarily for compaction.
```
struct cn_node_stats {
    struct kvset_stats ns_kst; // Sum of kvset stats for all kvsts in node
    uint64_t ns_keys_uniq;     // number of unique keys (estimated from HyperLogLog stats)
    uint64_t ns_hclen;         // estimated total hblock capacity (mpr_alloc_cap) after compaction
    uint64_t ns_kclen;         // estimated total kblock capacity (mpr_alloc_cap) after compaction
    uint64_t ns_vclen;         // estimated total vblock capacity (mpr_alloc_cap) after compaction
    uint16_t ns_pcap;          // current size / max size as a percentage (0 <= ns_pcap <= 100)
};
```

