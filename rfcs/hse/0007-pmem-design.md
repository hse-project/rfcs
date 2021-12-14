# PMEM via DAX support

## Problem

Add support for using persistent memory in HSE

## Requirements

1. Support persistent memory as a separate media class
2. Support persistent memory configured in fsdax mode
3. Support persistent memory configured in sector mode
4. Support storing HSE's storage resident structures on pmem mclass
5. Support placing the entire KVDB on pmem mclass
6. The design must not prevent storing HSE's memory resident structures on pmem
   mclass

## Non-Requirements

1. Support for storing memory resident structures on the pmem mclass
2. Support persistent memory configured in devdax mode

## Solution

### Terminology

- **_pmem_** -- Byte-addressable non-volatile persistent memory

- **_DAX_** -- Direct load/store access to persistent memory via mmap(2)

- **_BTT_** -- Block Translation Table, the driver that provides sector
  atomicity when configuring a persistent memory in sector mode

- **_mmap_** -- System call to memory-map a file or device into memory

- **_WAL_** -- Write-Ahead Log to provide durability guarantees to c0 data in
  the event of a crash

- **_mblock_** -- Media blocks written exactly once, are immutable after
  writing, and can be read in whole or in part until deleted

- **_OID_** -- Object Identifier

- **_MDC_** -- A metadata container implemented on top of a pair of files
  providing crash safe record logging functionality

- **_mcache_** -- Provides an abstraction to glue an arbitrary collection of
  memory-mapped mblocks

- **_kblock_** -- An mblock that stores keys, key and value metadata, bloom
  filter, hlog etc.

- **_vblock_** -- An mblock that stores values

- **_kvset_** -- A sorted container in cN storing keys and values from an ingest
  or compaction. A kvset comprises of one or more kblocks and zero or more
  vblocks.

- **_cNDB_** -- An MDC enabling transactional creation/deletion of cN kvsets

### Background

Persistent memory or storage class memory describes media that provides
non-volatile, byte-addressable, cache-coherent load-store access. A pmem device
can be connected to the system memory bus today. With CXL 2.0, a pmem device can
be connected to the CXL bus and accessed via the CXL.mem protocol.

In Linux, persistent memory can be configured in one of the following three
modes:

1. Sector mode
2. Filesystem-DAX (fsdax) mode
3. Device-DAX (devdax) mode

In sector mode, pmem is exposed as a traditional block device on which an FS can
be created as on any other block device. Sector mode does provide power-fail
safe atomic 4K-writes with the help of the BTT driver, so any application that
relies upon the sector atomicity can run unchanged. Please note that the
page-cache is _not_ skipped in this mode, so the app is just presented with a
faster block store.

In fsdax mode, pmem is exposed via a persistent memory aware filesystem. fsdax
removes the page-cache from the I/O path. The read(2)/write(2) syscalls incur
the cost of a kernel mode switch, but the DAX mode enables the kernel to
implement I/O using memcpy() from/to the pmem device. The mmap(2) syscall
establishes direct mapping to the pmem device, so any subsequent memory-mapped
accesses get direct load/store semantics to the pmem device completely bypassing
the kernel. For user data persistence, the app can either use the standard
msync()/fsync() syscalls or issue mmap(2) with MAP_SYNC | MAP_SHARED_VALIDATE
and perform explicit cache flushes followed by SFENCE from the userland.

In the devdax mode, pmem is exposed as a raw character device. The only way for
an app to access pmem in this mode is through load-store access by
memory-mapping the pmem device. The user cannot create a filesystem on the pmem
device, so this mode doesn't provide support for FS read(2)/write(2) operations.
The app must perform explicit cache flushes and SFENCE from the userland to
persist user data.

On Intel architectures, the power fail-safe atomicity for persistent memory is
guaranteed for naturally aligned load/stores up to 64-bits that do not cross an
8-byte boundary. The flush unit is one cache-line, however, it is possible to
have torn cache-line updates to persistent memory (fsdax/devdax mode) in the
event of a crash.

### HSE storage-resident structures

The following is a list of storage resident structures in HSE, their I/O access
modes, and its reliance on power fail-safe atomic 4K-writes.

#### Mblock metadata file: mblock-meta-\<mclassid\>

- One pre-allocated metadata file (~50MiB) per media class to track mblock
  metadata
- mmap(2) is the only mode of access for this file and all pages are mlocked on
  fault
- The mblock meta file comprises the following on-media structs:
  1.  A 4K mblock meta header, formatted at mpool creation and never updated
  2.  A 4K mblock file header, one per mblock data file
  3.  A fixed set of mblock OID meta structs, one for each mblock, following the
      4K file header
  4.  All mblock meta file structs rely on the sector atomicity, i.e., none are
      checksummed

#### Mblock data file: mblock-data-\<mclassid\>-\<fileid\>

- An mblock data file hosts several fixed size (default is 32MiB) mblocks
- There are one or more mblock data files (default is 32) in each media class
- Mpool writes mblock data files with O_DIRECT synchronous writes
- Mpool reads mblock data files with both O_DIRECT synchronous reads and
  memory-mmaped reads
- An mblock data file is created sparse without any space pre-allocation
- Mblock delete punches holes in the mblock data file at the affected file
  offset and range

#### Metadata Container (MDC) - cNDB and WAL MDC

- MDC comprises a pair of sparse files whose size is determined at allocation
  time
- An MDC file comprises a 4K-header followed by zero or more user records. Both
  the header and user records are checksummed.
- MDC sees frequent tiny synchronous appends that are a few hundred bytes in
  size and large asynchronous appends during a compaction event which is rare
- An MDC file is written via mmap(2) for records < 4K and via cached write(2)
  for >= 4K. For synchronous appends, msync(MS_SYNC) and fsync() follows the MDC
  write.
- An MDC file is read via mmap(2) and the read is issued only at MDC open

#### cN Kblocks

- Compaction writes:
  - A cN kblock is fully prepared in memory and then written in 1MiB chunks via
    direct mblock write
- Compaction reads:
  - A cN kblock's wbtree and kmd regions are asynchronously read via direct
    mblock read using 2 x 512KiB buffers. Bloom lookup is done via mcache by
    default but there's facility to use direct reads. Hyperloglog is accessed
    via mcache.
- Point queries and cursors use mcache map for reading kblocks

#### cN Vblocks

- Compaction writes:
  - A cN vblock is prepared in memory in 1MiB chunks and written via direct
    mblock write
- Compaction reads:
  - A cN vblock is read via direct mblock read using 2 x 1MiB buffers in the
    root node and 2 x 256K buffers in the non-root nodes
- Point queries:
  - A cN vblock is read via direct mblock read for values >= 4K and via mcache
    map for values < 4K
- Cursors:
  - Cursors use mcache map for reading vblocks

#### WAL data file

- WAL logging:

  - There are two WAL buffers per NUMA node and one WAL file per buffer
  - The WAL buffers are persisted every durability interval via O_DIRECT +
    O_SYNC writes to the WAL files
  - WAL file header and key-value records are all checksummed

- WAL replay:
  - The candiate WAL files chosen for replay are read via mmap(2)
  - Replay uses managed buffers, so the key and value pointers in c0 point to
    the memory mapped region from the WAL files

Below is a high level summary of the current data access methods in HSE:

| HSE Component                       | Operations                            | Direct Read | Direct Write | mmap Read | mmap Write |  Atomicity  |
| :---------------------------------- | :------------------------------------ | :---------: | :----------: | :-------: | :--------: | :---------: |
| mblock-meta-\<mclassid\>            | mblock alloc, commit, del; mpool open |     No      |      No      |    Yes    |    Yes     |   Sector    |
| mblock-data-\<mclassid\>-\<fileid\> | mblock read, write; mcache read       |     Yes     |     Yes      |    Yes    |     No     |  via cNDB   |
| cNDB and WAL MDC                    | MDC append, read                      |     No      |     Yes      |    Yes    |    Yes     | Checksummed |
| cN kblock                           | Compaction                            |     Yes     |     Yes      |    Yes    |     No     |  via cNDB   |
| cN kblock                           | Point queries                         |     No      |      NA      |    Yes    |     NA     |     NA      |
| cN kblock                           | Cursors                               |     No      |      NA      |    Yes    |     NA     |     NA      |
| cN vblock                           | Compaction                            |     Yes     |     Yes      |    Yes    |     No     |  via cNDB   |
| cN vblock                           | Point queries                         |     Yes     |      NA      |    Yes    |     NA     |     NA      |
| cN vblock                           | Cursors                               |     No      |      NA      |    Yes    |     NA     |     NA      |
| WAL data file                       | WAL put, del, pdel, replay            |     No      |     Yes      |    Yes    |     No     | Checksummed |

### Applicability of Persistent Memory in HSE

**_Which pmem mode is better for HSE - sector or fsdax or devdax?_**

HSE can run unchanged in sector mode, however, this mode doesn't fully utilize
the byte-addressable and persistent nature of the media.

The devdax mode does not have the FS overhead. However, supporting it will add
the following complexities in HSE:

1. Add support to manage the persistent memory address space to store both
   storage and memory resident structures
2. Add support for all IO accesses to be via memory-mapped IO
3. Add support in mpool to run on a raw character device
4. May need to add some architecture specific code to issue CPU cache flush +
   fence instructions from userland, not sure if msync() can be used in devdax
   mode
5. Creates an administrative burden in that the devdax device must be
   partitioned via LVM (or other mechanism) in order for KVDBs to share a pmem
   device

The fsdax mode is a natural fit for HSE for the following reasons:

1. HSE is already equipped to run on a filesystem
2. The fsdax mode supports all the current data access methods in HSE
3. The fsdax mode provides load-store semantics to pmem via mmap(2)
4. Adding future support to store memory-resident structures on pmem is simpler
   as the filesystem provides address-space, namespace and access control
   management
5. The fsdax mode enables KVDBs to easily share a pmem device

For the above reasons, fsdax mode is better suited for HSE.

**_Which workloads benefit from persistent memory?_**

1. Workloads whose working sets fit entirely in DRAM are better off continuing
   to use DRAM due to its low access latency compared to pmem which is approx.
   4x-10x slower. This is especially because HSE relies on the page-cache for
   database caching.

   The above point is based on the latency characteristics of the pmem media
   which is available in the market today.

2. Workloads whose working sets are several times the DRAM size and those which
   run in memory constrained environments can take huge advantage of persistent
   memory. The access latency of a pmem device is approx. 10-100x better than a
   NAND SSD.

**_Persistent memory - separate media class vs property of a media class?_**

Having persistent memory as a separate tier/mclass allows HSE to pick and choose
the memory and storage resident structures to place on persistent memory using
appropriate mclass policies. The 10-100x better access latency of pmem media
compared to the NAND SSDs enables pmem to fit perfectly as a new tier in the HSE
mclass hierarchy, above staging and capacity.

Using a separate mclass for pmem isolates all pmem related optimizations to a
single media class vs having is_pmem() check all over the place when we have DAX
as a property of an media class.

Finally, it provides a simpler and cleaner management model.

So, the recommended approach is to configure pmem media as a separate media
class.

_What if a user still configures a DAX filesystem in the capacity/staging tier?_

For 2.1.0, HSE must enforce in code and also document that a DAX filesystem can
be configured only in the pmem media class. HSE cannot guarantee correctness if
the staging or capacity media class are in a DAX filesystem.

**_Is capacity media class still mandatory for a KVDB?_**

The capacity media class will continue to be mandatory except the case where a
KVDB is created with a home directory in a DAX filesystem.

So we will have two flavors of KVDB: standard and pmem-only.

1. A standard KVDB is unchanged from what we have today, except that one can now
   configure the optional pmem media class.

2. A pmem-only KVDB is indicated by creating a KVDB home directory in a DAX file
   system, in which case it only has the pmem media class, with the default
   location being KVDB_HOME/pmem (analogous to the default capacity path for a
   standard KVDB). A pmem-only KVDB supports only the "pmem_only" mclass policy.

   A pmem-only KVDB becomes a standard KVDB by adding a capacity media class,
   which by definition cannot exist in a DAX file system. A standard KVDB cannot
   be converted to a pmem-only KVDB since it will by definition have cN data in
   at least the capacity media class.

### HSE Persistent Memory Optimizations

The optimizations below are categorized into three priorities.

- The _Must-Haves_ are needed for 2.1.0 for ensuring data integrity on a pmem
  device.
- The _Nice-to-Haves_ can be considered for 2.1.0, if time permits.
- The _Future Opportunities_ are strictly for a release later than 2.1.0.

#### Mblock metadata file: mblock-meta-\<mclassid\>

_Must-Haves:_

1.  Bump the mblock meta file version
2.  Checksum the following OMF structs in the mblock meta file: 4K mblock meta
    header, 4K mblock file header and mblock OID meta.
3.  Ensure that the checksum field is at a naturally aligned offset, less than
    64-bits wide, do not cross an 8-byte boundary, and is the last thing that's
    updated at mblock_alloc()/commit()/delete(). Also, add a valid bit to the
    checksum field to catch a checksum collision following a crash.
4.  Employ user-space cacheline flushing using libpmem1 interfaces
5.  Gracefully handle checksum validation failures in the mpool meta OMF structs
    during crash recovery
6.  Add upgrade support from 2.0

#### Mblock data file: mblock-data-\<mclassid\>-\<fileid\>

_Nice-to-Haves:_

1.  Memory-mapped mblock writes:

    - Add support for writing mblocks via memory-map. This must be enabled only
      on the pmem mclass with DAX access, otherwise it will pollute the
      page-cache.

    - One way to implement this is to modify mpool_mblock_alloc() to optionally
      return a memory-mapped address through which this allocated mblock can be
      written.

      An mblock data file is memory-mapped today in 32G chunks, each chunk
      encompassing 1024 x 32MiB mblocks. A chunk is created with only PROT_READ
      protection by default. Before returning the mapped address for the newly
      allocated mblock, change the protection for this mblock offset range to
      PROT_READ + PROT_WRITE.

      At mblock_commit() or mblock_abort(), reset the protection on the mblock
      range back to PROT_READ.

      cN could then prepare its kblocks/vblocks directly onto the persistent
      memory through the mapped address instead of allocating an intermediate
      DRAM buffer. This eliminates the memcpy from DRAM to pmem and the syscall
      overhead.

    - The other alternative is to add a pmem backend to mpool that implements
      mpool_mblock_read()/write() using memcpy when running on a DAX filesystem.
      This is transparent to cN and eliminates the syscall overhead but doesn't
      get rid of the memcpy overhead from DRAM to pmem.

_Future Opportunities:_

1.  Separating out certain kblock regions like the cache-aligned blocked bloom
    into an mblock of its own, allocating it from the pmem tier and accessing it
    via load/store operations may prove to be beneficial.

#### MDC - cNDB and WAL

_Must-Haves:_

1.  Write MDC via mmap(2) irrespective of the record size
2.  Ensure that the checksum field is at a naturally aligned offset, less than
    64-bits wide, do not cross an 8-byte boundary, and is the last thing that's
    updated. Also, add a valid bit to the checksum field to catch a checksum
    collision following a crash.
3.  Employ user-space cacheline flushing using libpmem1 interfaces
4.  Validate that the failure scenarios are handled correctly during MDC reopen
5.  Add upgrade support from 2.0

#### cN kblocks and vblocks

_Must-Haves:_

1.  Enable compaction logic to use mcache map for compaction reads. The code
    path already exists to use mcache for compaction reads, it just needs to be
    enabled and tested. It will be interesting to evaluate whether using mcache
    for compaction reads increase the compaction run time as there's no
    readahead in this case.

2.  For point queries, read vblocks via mcache maps irrespective of the value
    size. This eliminates the read() system call overhead.

_Nice-to-Haves:_

1.  With memory-mapped mblock write support, the kblock and vblock builder need
    not require an intermediate write buffer when operating on a pmem device.
    The builder can prepare the k/vblock data in the memory-mapped address
    returned by the mblock alloc call, and this would translate into direct
    stores to the persistent memory. This a) saves DRAM b) eliminates the
    syscall overhead and c) eliminates a copy from DRAM to pmem.

#### WAL data file

It is recommended to leave WAL buffers in DRAM for now to achieve low latency
puts.

_Must-Haves:_

1.  Ensure that the checksum field is at a naturally aligned offset, less than
    64-bits wide, do not cross an 8-byte boundary, and is the last thing that's
    updated. Also, add a valid bit to the checksum field to catch a checksum
    collision following a crash.

2.  Add upgrade support from 2.0, if needed.

_Nice-to-Haves:_

1.  Evaluate the benefit of writing to WAL files using memory-mapped IO vs using
    the default O_DIRECT + O_SYNC write() IO. If the application issues frequent
    kvdb syncs, then the memory-mapped IO might perform better as there's no
    syscall overhead.

_Future Opportunities:_

1.  Having c0 entirely on pmem and potentially removing the need for WAL when c0
    lives on pmem have a tremendous value in low-memory environments.

2.  There is a hybrid option to explore for low-memory environments which will
    use a combination of DRAM and pmem to store bonsai trees. This requires some
    minor redesign of WAL buffer management. The idea is to allocate the
    structural elements of a bonsai tree like tree nodes from DRAM and to
    allocate the values (and optionally keys) from pmem. This has an advantage
    of traversing the bonsai tree at DRAM speed to land at a target bonsai node
    and then access values directly from pmem.

    We already have a buffer managed mode in WAL, where a c0 bonsai tree stores
    pointers to WAL buffers for keys and values. If a WAL buffer is nothing but
    a memory map of a WAL file that lives on the pmem mclass, then inserting
    keys and values into c0 bonsai trees will also persist them after the
    required calls for ensuring persistence is issued. Of-course, WAL is still
    needed in this hybrid mode to store other metadata for a kv-pair and to
    rebuild the bonsai tree during replay.

### HSE Media Class Policies

This section lists the existing media class policies and the new additions with
the introduction of the pmem media class.

1. WAL Media Class

   The media class for WAL is a per-KVDB property and is defined at KVDB open
   time using the _durability.mclass_ rparam.

   It is recommended to always configure WAL in the best available mclass tier.

```
    durability.mclass = pmem | staging | capacity
```

If the durability.mclass refers to a media class that isn't configured, then
fallback to the next best available media class.

2. Media Class Policies

   The media class policies are defined at a per-KVS granularity using the KVS
   rparam _mclass.policy_. The mclass policy affects the placement of kblocks
   and vblocks allocated by cN during ingest and compactions.

   The new pmem_max_capacity is similar to staging_max_capacity in that the cN
   root and all keys live on pmem and the values are tiered from pmem to
   capacity.

   There's no fallback support for media class policies, i.e., if a media class
   policy is configured for a KVS and the KVDB doesn't contain one or more of
   the media as required by the configured policy, then KVS open fails with an
   error.

   The media class policy for a KVS defaults to "auto" which lets HSE choose a
   best policy based on the media classes configured for a KVDB.

   | Mclass Policy     | cN Root Keys / Values | cN Internal Keys / Values | cN Leaf Keys / Values |
   | :---------------- | :-------------------: | :-----------------------: | :-------------------: |
   | pmem_only         |      pmem / pmem      |        pmem / pmem        |      pmem / pmem      |
   | pmem_max_capacity |      pmem / pmem      |        pmem / cap         |      pmem / cap       |

## Failure and Recovery Handling

Mpool, cNDB and WAL combined must handle partial pmem updates gracefully during
crash recovery with the help of the checksummed on-media structures.

## Testing Guidance

1. Run standard media class workload testing with pmem in the mix and different
   mclass policies

2. Run standard crash recovery tests with WAL on the pmem tier

3. Run standard CLI testing to ensure that the new media class addition is
   working fine

4. Run upgrade tests from 2.0.0/2.0.1 to 2.1.0

## References

- [Persistent Memory Programming](https://pmem.io/)
- [FSDAX](https://www.kernel.org/doc/Documentation/filesystems/dax.txt)
- [USENIX article on pmem](https://www.usenix.org/system/files/login/articles/login_summer17_07_rudoff.pdf)
- [PMEM google group](https://groups.google.com/g/pmem)
- [Block Translation Table](https://www.kernel.org/doc/html/latest/driver-api/nvdimm/btt.html)
