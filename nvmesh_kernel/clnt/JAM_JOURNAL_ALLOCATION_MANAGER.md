# JAM - Journal Allocation Manager

## Overview

The Journal Allocation Manager (JAM) is a critical component that manages journal entry allocation for **Erasure Coded (EC) volumes**. It sits between the block layer and the transport/disk layer, providing transactional semantics for journal operations.

### Purpose

- **Transaction Management**: Ensures atomic allocation of journal entries across multiple disks in an EC stripe
- **Ambiguity Prevention**: Prevents multiple journal entries from pointing to the same data block
- **Pending Request Handling**: Queues requests when journal entries are temporarily unavailable
- **Cache Preservation**: Maintains journal state across disk rediscovery cycles

### Architecture

```
Block Layer (EC Datapath)
         |
         | (alloc/free/abandon)
         v
   JAM (nvmeibc_jam.c)
         |
         | (execute I/O)
         v
   Pausable Layer (PD)
         |
         v
   Disk/Transport Layer
```

## Key Data Structures

### Global JAM Context

```c
struct nvmeibc_jam {
    spinlock_t lock;                    // Global JAM lock
    struct list_head disks;             // List of jam_disk instances
    int n_disks;                        // Number of disks
    struct nvmeibc_jam_percpu_cnts __percpu *pcpu_cnts;  // Per-CPU statistics
};
```

One `nvmeibc_jam` instance per NVMesh client instance.

### Per-Disk JAM Context

```c
struct nvmeibc_jam_disk {
    int jidx_pool_size;                     // Number of journal entries
    int binje_shift;                        // Blocks-in-journal-entry shift
    struct nvmeibc_jam_jidx *jidx_pool;    // Array of journal indices
    spinlock_t lock;                        // Per-disk lock
    struct list_head free_list;             // Free journal entries
    bool alloc_enb;                         // Allocation enabled flag
    
    // Counts
    int n_alloced;                          // Currently allocated
    int n_free;                             // Currently free
    int n_abandoned;                        // Abandoned (waiting for cleanup)
    int n_erasing;                          // Being erased
    
    // Hash table for uniqueness (TxID, J2B) -> prevents ambiguity
    DECLARE_HASHTABLE(uniqj2b_htable, 8);   // 256 buckets
    struct list_head uniqj2b_trans_list;    // Transient entries
    
    // Pending requests (priority-ordered red-black tree)
    struct rb_root pending_root;
    int n_pending;
    
    // Workqueue for async operations
    struct workq_struct *jwq;
    
    struct nvmeibc_disk *disk;              // Back-pointer to disk
    struct nvmeib_public_procfs_ent *proc_ent_status;  // Debug interface
};
```

### Journal Index (Entry)

```c
struct nvmeibc_jam_jidx {
    int idx;                                    // Index in pool
    enum nvmeibc_jam_jidx_state state;          // Current state
    struct list_head pool_link;                 // Link in free_list
    u8 gen_id;                                  // Generation ID
    
    // Hash keys for uniqueness check
    struct jidx_hkeys hkeys;                    // committed + transient
    struct hlist_node uniqj2b_hlink;           // Hash table link (committed)
    struct list_head uniqj2b_trans_link;       // List link (transient)
    
    // Pending requests bound to this entry
    struct nvmeibc_jam_pending_req *jreq_bound_via_committed;
    struct nvmeibc_jam_pending_req *jreq_bound_via_transient;
    
    // Timing and retry info
    u64 alloc_jif, abnd_jif, erase_jif;
    u8 n_erase_attempts;
    bool skipped;                               // For no-locks mode
};
```

#### Journal Entry States

```c
enum nvmeibc_jam_jidx_state {
    NVMEIBC_JIDX_STS_FREE,          // Available for allocation
    NVMEIBC_JIDX_STS_ALLOCED,       // Allocated to a transaction
    NVMEIBC_JIDX_STS_ERASING,       // Being erased remotely
    NVMEIBC_JIDX_STS_ABANDONED      // Failed write, awaiting cleanup
};
```

### Hash Key (Uniqueness Check)

```c
typedef union jidx_hkey__t {
    struct {
        u32 j2b      : NVMEIB_EC_JMDC_BITS_J2B;      // Journal-to-blockset ptr
        u32 tx_id    : NVMEIB_EC_JMDC_BITS_TX_ID;    // Transaction ID
        u32 reserved : (32 - NVMEIB_EC_JMDC_BITS_TX_ID);
    } __attribute__((packed));
    u64 raw;
} __attribute__((packed)) jidx_hkey_t;
```

**Purpose**: Ensures only one journal entry points to a given data blockset within a transaction.

- `j2b`: Data blockset LBA >> `LOCKSET_SLICES_SHIFT`
- `tx_id`: Transaction identifier
- Used in hash table to detect conflicts

### Pending Request

```c
struct nvmeibc_jam_pending_req {
    int n_disks;                            // Number of disks in stripe
    struct jalloc *sorted;                  // Disks sorted for allocation
    u32 txid;                               // Transaction ID
    bool wait_bound_abnd;                   // Wait for abandoned entries?
    
    u64 *res_jlbas;                         // Result journal LBAs
    void *ctx;                              // Callback context
    
    int curr;                               // Current disk being allocated
    jidx_hkey_t hkey;                       // Hash key for current disk
    int bound_idx;                          // Index blocking this request
    
    struct rb_node pending_node;            // RB-tree node (ordered by priority)
    
    // Timeout handling
    TIMER_LIST_INSTANCE(timeout_timer);
    bool timeout_expired;
    unsigned long pend_jif;
    unsigned long deadline_jif;
    unsigned long priority;
    
    // Deferred resume
    atomic_t resume_wip;
    struct workqe_struct resume_work;
    struct nvmeibc_jam_disk *jam_disk;
    
    const struct nvmeib_cpu_mask_info *cpu_mask_info;  // CPU affinity
};
```

## JAM Cache (Rediscovery Preservation)

### The Problem

During disk rediscovery:
1. Disk is released (disconnected from network)
2. JAM disk is destroyed (`nvmeibc_jam_disk_del`)
3. Disk is rediscovered (reconnected)
4. JAM disk is recreated (`nvmeibc_jam_disk_add`)

**Challenge**: Journal state must survive this cycle to avoid:
- Re-allocating in-use entries
- Losing track of abandoned entries
- Breaking ongoing transactions

### The Solution: JAM Cache in `c_disk`

```c
struct nvmeibc_jam_range_cache {
    struct nvmeibc_disk_client_journal jour;     // Journal range info
    DECLARE_BITMAP(free_bitmap, NVMEIB_EC_JOURNAL_MAX_ENTRIES_PER_RANGE);
    struct nvmeib_jrnl_ent_md ent_md[NVMEIB_EC_JOURNAL_MAX_ENTRIES_PER_RANGE];
    int n_jents;                                 // Number of entries
};

struct nvmeibc_disk {
    ...
    struct nvmeibc_jam_range_cache jrc;          // JAM cache
    struct nvmeibc_jam_disk *jam_disk;           // Active JAM (NULL when released)
    ...
};
```

### Cache Lifecycle

#### 1. Initialization (First Discovery)

```c
void nvmeibc_jam_disk_cache_init(struct nvmeibc_disk *disk)
{
    nvmeibc_disk_client_journal_mark_as_no_journal(&disk->jrc.jour);
    disk->jrc.n_jents = 0;
    bitmap_zero(disk->jrc.free_bitmap, NVMEIB_EC_JOURNAL_MAX_ENTRIES_PER_RANGE);
    memset(disk->jrc.ent_md, NVMEIB_EC_INVALID_JOURNAL_ENT_GEN_ID,
           sizeof(disk->jrc.ent_md));
}
```

Called once when disk is created. Initializes empty cache.

#### 2. Cache Fill (Before Destruction)

```c
static void jam_disk_cache_fill(struct nvmeibc_disk *disk)
{
    struct nvmeibc_jam_jidx *jidx_pool = disk->jam_disk->jidx_pool;
    int jidx_pool_size = disk->jam_disk->jidx_pool_size;
    int i;
    
    // Copy journal range info
    disk->jrc.jour = disk->jour;
    disk->jrc.n_jents = jidx_pool_size;
    
    // Build free bitmap and copy generation IDs
    bitmap_zero(disk->jrc.free_bitmap, NVMEIB_EC_JOURNAL_MAX_ENTRIES_PER_RANGE);
    for (i = 0; i < jidx_pool_size; i++) {
        if (jidx_pool[i].state == NVMEIBC_JIDX_STS_FREE) {
            set_bit(i, disk->jrc.free_bitmap);
        }
        disk->jrc.ent_md[i].ent_gen_id = jidx_pool[i].gen_id;
    }
}
```

**Called from**: `nvmeibc_jam_disk_del` before destroying `jam_disk`.

**What's preserved**:
- Journal range metadata (range ID, generation ID, SERJIO boot ID)
- Free/allocated state of each entry (bitmap)
- Generation IDs for all entries

**What's NOT preserved**:
- Hash table contents (rebuilt on next allocation)
- Pending requests (canceled and failed back to block layer)
- Entries in ERASING state (treated as non-free)
- Abandoned entries (marked as non-free, will be cleaned by SERJIO)

#### 3. Cache Restore (During Rediscovery)

```c
int nvmeibc_jam_disk_add(struct nvmeibc_disk *disk,
                         unsigned long *free_ents_bmp,
                         union jblock_md *jmdc_tbl,
                         struct nvmeib_jrnl_ent_md *ent_md)
{
    // Allocate new jam_disk
    jam_disk = kzalloc(...);
    jidx_pool = kzalloc(jidx_pool_size * sizeof(*jidx_pool), ...);
    
    // Initialize each entry
    for (i = 0; i < jidx_pool_size; i++) {
        jidx_pool[i].idx = i;
        jidx_pool[i].gen_id = ent_md[i].ent_gen_id;
        
        // Restore state from free bitmap
        if (test_bit(i, free_ents_bitmap)) {
            jidx_pool[i].state = NVMEIBC_JIDX_STS_FREE;
            list_add_tail(&jidx_pool[i].pool_link, &jam_disk->free_list);
            jam_disk->n_free++;
        } else {
            // Non-free: could be ALLOCATED or ABANDONED
            // Determine from JMDC table
            ...
        }
    }
    
    // Add to global JAM
    disk->jam_disk = jam_disk;
    jam_disk->disk = disk;
}
```

**Called from**: Disk discovery after receiving journal range from server.

**Inputs**:
- `free_ents_bmp`: Fresh free bitmap from server (via SERJIO)
- `jmdc_tbl`: Journal metadata table from server
- `ent_md`: Entry metadata (generation IDs) from server

**Logic**:
1. Server's free bitmap is authoritative
2. Client's cached state provides hints
3. Merge both to determine final state
4. Abandoned entries remain abandoned until SERJIO cleans them

#### 4. Cache Communication with Server

When requesting journal range from server:

```c
void nvmeibc_jam_disk_cache_hton(struct nvmeibc_disk *disk,
                                 struct volume_client_config_jrange_cache *clnt_jrc)
{
    clnt_jrc->rng_id = cpu_to_be32(disk->jrc.jour.rng_id);
    clnt_jrc->rng_genid = cpu_to_be64(disk->jrc.jour.rng_gen_id);
    nvmeib_bitmap_to_be32(clnt_jrc->free_bitmap, disk->jrc.free_bitmap, n_jents);
    // ... copy generation IDs and SERJIO boot ID
}
```

**Purpose**: Tell server what client thinks is free so server can correct inconsistencies.

For local disks (bypass):

```c
void nvmeibc_jam_disk_cache_local(struct nvmeibc_disk *disk,
                                  struct nvmeib_jrange_cache *clnt_jrc)
{
    // Copy without byte-order conversion
    clnt_jrc->rng_id = disk->jrc.jour.rng_id;
    clnt_jrc->gen_id = disk->jrc.jour.rng_gen_id;
    bitmap_copy(clnt_jrc->free_bmp, disk->jrc.free_bitmap, n_jents);
    // ...
}
```

### Cache Benefits

1. **Fast Rediscovery**: Server can quickly validate client's cached state
2. **Consistency**: Prevents accidental reuse of non-free entries
3. **SERJIO Integration**: Works with server-side journal garbage collection
4. **Stateful Resume**: Client can resume after temporary disconnections

### Cache Limitations

- **Pending requests are lost**: Upper layer must retry failed allocations
- **Erasing entries may leak**: If erase was in-flight during disconnect
  - Mitigated by SERJIO eventually cleaning these
- **Abandoned entries persist**: Until SERJIO's free-abandoned message arrives
  - Client won't allocate over them, so safe

## Journal Allocation (`nvmeibc_jam_lbas_alloc`)

### High-Level Flow

```
Block Layer Request
        |
        v
Sort Disks (by address)
        |
        v
Acquire PD Approval (pause protection)
        |
        v
For Each Disk:
        |
        +---> Compute Hash Key (TxID, J2B)
        |
        +---> Check Uniqueness (hash table)
        |
        +---> Allocate Journal Index
        |       |
        |       +---> SUCCESS -> Continue to next disk
        |       |
        |       +---> BUSY -> Add to Pending Queue
        |       |
        |       +---> ERROR -> Rollback and fail
        |
        v
All Allocated -> Callback Success
Release PD Approval
```

### Allocation Steps

#### 1. Sort Disks

```c
struct jalloc *sorted = sort_disks(n_disks, disks);
```

Ensures consistent allocation order across all clients to prevent deadlocks.

#### 2. Per-Disk Allocation Loop

```c
for (i = 0; i < n_disks; i++) {
    struct nvmeibc_disk *disk = sorted[i].disk;
    struct nvmeibc_jam_disk *jam_disk = disk->jam_disk;
    u64 dlba = dlbas[sorted[i].orig_pos];
    
    // Compute hash key
    hkey.j2b = __j2d_to_j2b(dlba);  // Blockset index
    hkey.tx_id = txid;
    hkey.reserved = 0;
    
    // Allocate journal index
    rv = jidx_alloc_(jam_disk, hkey, &idx, wait_bound_abnd, false);
    
    if (rv == 0) {
        // Success: convert idx to LBA
        res_jlbas[orig_pos] = idx_2_lba(disk, idx);
    }
    else if (rv == -EBUSY) {
        // Pend request
        jreq = pending_req_alloc(...);
        jreq->curr = i;
        jreq->hkey = hkey;
        pending_req_list_add_(jreq);
        return -EINPROGRESS;
    }
    else {
        // Error: rollback and fail
        rollback_partial_allocation(sorted, res_jlbas, i);
        return rv;
    }
}
```

#### 3. Journal Index Allocation (`jidx_alloc_`)

```c
static int jidx_alloc_(struct nvmeibc_jam_disk *jam_disk, jidx_hkey_t hkey,
                      int *idx, bool wait_bound_abnd, bool dry_run)
{
    // Check if free list is empty
    if (list_empty(&jam_disk->free_list)) {
        // Check for abandoned conflict
        if (!wait_bound_abnd) {
            bind_rv = uniqj2b_bind_check(jam_disk, hkey, &bound_idx, ...);
            if (bind_rv == HKEY_BOUND_ABANDONED) {
                *idx = bound_idx;
                return -EDEADLK;  // Deadlock with abandoned entry
            }
        }
        *idx = NVMEIBC_JAM_INVALID_BOUND_IDX;
        return -EBUSY;  // No free entries
    }
    
    // Check max used entries limit
    if (nvmeibc_jam_max_used_entries &&
        jam_disk->n_alloced >= nvmeibc_jam_max_used_entries) {
        *idx = NVMEIBC_JAM_INVALID_BOUND_IDX;
        return -EBUSY;
    }
    
    // Check for hash conflict (ambiguity prevention)
    bind_rv = uniqj2b_bind_check(jam_disk, hkey, &bound_idx, ...);
    if (bind_rv != HKEY_UNBOUND) {
        // Conflict exists
        if (bind_rv == HKEY_BOUND_ABANDONED && !wait_bound_abnd) {
            *idx = bound_idx;
            return -EDEADLK;
        }
        *idx = bound_idx;
        return -EBUSY;  // Wait for conflicting entry to free
    }
    
    // Allocate from free list
    jidx = list_first_entry(&jam_disk->free_list, struct nvmeibc_jam_jidx, pool_link);
    
    if (!dry_run) {
        jidx_get_(jam_disk, jidx);  // Remove from free list, add to hash
        *idx = jidx->idx;
    }
    
    return 0;  // Success
}
```

#### 4. Uniqueness Check (`uniqj2b_bind_check`)

```c
enum {
    HKEY_UNBOUND = 0,               // No conflict
    HKEY_BOUND_ALLOCED,             // Conflict with allocated entry
    HKEY_BOUND_ERASING,             // Conflict with erasing entry
    HKEY_BOUND_ABANDONED,           // Conflict with abandoned entry
};

static int uniqj2b_bind_check(struct nvmeibc_jam_disk *jam_disk,
                             jidx_hkey_t hkey, int *bound_idx, int flags)
{
    struct nvmeibc_jam_jidx *jidx;
    
    // Check committed hash table
    hash_for_each_possible(jam_disk->uniqj2b_htable, jidx, uniqj2b_hlink, hkey.raw) {
        if (jidx->hkeys.committed.raw == hkey.raw) {
            *bound_idx = jidx->idx;
            switch (jidx->state) {
            case NVMEIBC_JIDX_STS_ALLOCED:   return HKEY_BOUND_ALLOCED;
            case NVMEIBC_JIDX_STS_ABANDONED: return HKEY_BOUND_ABANDONED;
            default: BUG();
            }
        }
    }
    
    // Check transient list (erasing entries)
    if (!(flags & HKEY_BIND_CHECK_ONLY_COMMITTED)) {
        list_for_each_entry(jidx, &jam_disk->uniqj2b_trans_list, uniqj2b_trans_link) {
            if (jidx->hkeys.transient.raw == hkey.raw) {
                *bound_idx = jidx->idx;
                return HKEY_BOUND_ERASING;
            }
        }
    }
    
    return HKEY_UNBOUND;
}
```

### Pending Request Handling

When allocation returns `-EBUSY`, request is queued:

```c
// Add to priority-ordered RB-tree
static void pending_req_list_add_(struct nvmeibc_jam_pending_req *jreq)
{
    struct nvmeibc_jam_disk *jam_disk = jreq->sorted[jreq->curr].disk->jam_disk;
    struct rb_root *root = &jam_disk->pending_root;
    struct rb_node **new = &root->rb_node, *parent = NULL;
    
    // Find insertion point (ordered by priority, then deadline)
    while (*new) {
        struct nvmeibc_jam_pending_req *this = 
            container_of(*new, struct nvmeibc_jam_pending_req, pending_node);
        parent = *new;
        
        // Higher priority goes first
        if (jreq->priority > this->priority)
            new = &parent->rb_left;
        else if (jreq->priority < this->priority)
            new = &parent->rb_right;
        // Same priority: earlier deadline goes first
        else if (time_before(jreq->deadline_jif, this->deadline_jif))
            new = &parent->rb_left;
        else
            new = &parent->rb_right;
    }
    
    rb_link_node(&jreq->pending_node, parent, new);
    rb_insert_color(&jreq->pending_node, root);
    jam_disk->n_pending++;
    
    // Start timeout timer
    mod_timer(&jreq->timeout_timer, jreq->deadline_jif);
}
```

### Pending Request Resume

When a journal entry becomes free, pending requests are resumed:

```c
static void pending_req_resume_(struct nvmeibc_jam_pending_req *jreq, int idx)
{
    struct jalloc *sorted = jreq->sorted;
    int curr = jreq->curr;
    u64 lba;
    
    // Allocate this disk
    lba = idx_2_lba(sorted[curr].disk, idx);
    jreq->res_jlbas[sorted[curr].orig_pos] = lba;
    jreq->curr++;  // Move to next disk
}

static void pending_req_resume_bh(struct nvmeibc_jam_pending_req *jreq)
{
    // Continue allocation from curr to n_disks
    for (i = jreq->curr; i < jreq->n_disks; i++) {
        rv = jidx_alloc_(...);
        if (rv != 0) break;
    }
    
    if (rv == 0) {
        // All allocated - success callback
        nvmeibc_block_dp_ec_journal_alloc_cb(0, jreq->res_jlbas, jreq->ctx);
        pending_req_free(jreq);
    }
    else if (rv == -EBUSY) {
        // Still busy - re-queue
        jreq->curr = i;
        pending_req_list_add_(jreq);
    }
    else {
        // Error - fail callback
        rollback_partial_allocation(...);
        nvmeibc_block_dp_ec_journal_alloc_cb(rv, NULL, jreq->ctx);
        pending_req_free(jreq);
    }
}
```

### Timeout Handling

```c
TIMER_CALLBACK_DECL(pending_req_timeout_timer_fn)
{
    struct nvmeibc_jam_pending_req *jreq = TIMER_GET_DATA(data);
    
    // Remove from pending list
    pending_req_list_del_(jreq, jam_disk);
    
    // Rollback partial allocation
    rollback_partial_allocation(jreq->sorted, jreq->res_jlbas, jreq->curr);
    
    // Fail callback with timeout
    nvmeibc_block_dp_ec_journal_alloc_cb(-ETIMEDOUT, NULL, jreq->ctx);
    
    pending_req_free(jreq);
}
```

## Journal Free (`nvmeibc_jam_lbas_free`)

### Free with Write Status

```c
void nvmeibc_jam_lbas_free(int n_disks, struct nvmeibc_disk *disks[],
                           u64 jlbas[], u32 wr_sts_bm)
{
    for (i = 0; i < n_disks; i++) {
        int idx = lba_2_idx(disks[i], jlbas[i]);
        
        if (wr_sts_bm & (1 << i)) {
            // Write FAILED or not issued - erase remotely
            jidx_event(disks[i], idx, NVMEIBC_JIDX_EVT_ERASE);
        } else {
            // Write OK - just mark free
            jidx_event(disks[i], idx, NVMEIBC_JIDX_EVT_END_USE);
        }
    }
}
```

**Write status bitmap**:
- Bit `i` = 0: Journal write completed successfully -> simple free
- Bit `i` = 1: Journal write failed or not issued -> remote erase required

### Journal Entry State Machine

```c
static int jidx_event_(struct nvmeibc_disk *disk, int idx, u8 *idx_gen_id,
                      enum nvmeibc_jam_jidx_event evt,
                      bool *put_back, int *n_jreqs)
{
    struct nvmeibc_jam_disk *jam_disk = disk->jam_disk;
    struct nvmeibc_jam_jidx *jidx = &jam_disk->jidx_pool[idx];
    
    switch (evt) {
    case NVMEIBC_JIDX_EVT_END_USE:
        if (jidx->state == NVMEIBC_JIDX_STS_ALLOCED) {
            jam_disk->n_alloced--;
            *n_jreqs = unlink_jidx_from_any_jreq(jidx);
            uniqj2b_commit_transient_(jam_disk, jidx);  // Merge transient->committed
            
            jidx->state = NVMEIBC_JIDX_STS_FREE;
            *put_back = true;  // Return to free list
            rv = 0;
        }
        break;
        
    case NVMEIBC_JIDX_EVT_ERASE:
        if (jidx->state == NVMEIBC_JIDX_STS_ALLOCED) {
            jam_disk->n_alloced--;
            uniqj2b_hadd_transient_(jam_disk, jidx);  // Add transient to list
            
            jidx->state = NVMEIBC_JIDX_STS_ERASING;
            jam_disk->n_erasing++;
            jidx->n_erase_attempts = 0;
            
            rv = jentry_erase(disk, jidx);  // Issue remote erase
        }
        break;
        
    case NVMEIBC_JIDX_EVT_ERASE_COMP:
        if (jidx->state == NVMEIBC_JIDX_STS_ERASING) {
            jam_disk->n_erasing--;
            *n_jreqs = unlink_jidx_from_any_jreq(jidx);
            uniqj2b_hdel_both(jam_disk, jidx);  // Remove from hash
            
            jidx->state = NVMEIBC_JIDX_STS_FREE;
            *put_back = true;
            rv = 0;
        }
        break;
        
    case NVMEIBC_JIDX_EVT_ERASE_COMP_ERR:
        if (jidx->state == NVMEIBC_JIDX_STS_ERASING) {
            // Retry erase
            rv = jentry_erase(disk, jidx);
        }
        break;
        
    case NVMEIBC_JIDX_EVT_ABANDON:
        if (jidx->state == NVMEIBC_JIDX_STS_ALLOCED) {
            jam_disk->n_alloced--;
            *n_jreqs = unlink_jidx_from_any_jreq(jidx);
            uniqj2b_commit_transient_(jam_disk, jidx);
            
            jidx->state = NVMEIBC_JIDX_STS_ABANDONED;
            jidx->abnd_jif = jiffies;
            jam_disk->n_abandoned++;
            *idx_gen_id = jidx->gen_id;  // Return for SERJIO
            rv = 0;
        }
        break;
        
    case NVMEIBC_JIDX_EVT_FREE_ABND:
        if (jidx->state == NVMEIBC_JIDX_STS_ABANDONED) {
            jam_disk->n_abandoned--;
            *n_jreqs = unlink_jidx_from_any_jreq(jidx);
            uniqj2b_hdel_committed_(jam_disk, jidx);
            
            jidx->state = NVMEIBC_JIDX_STS_FREE;
            jidx->gen_id = *idx_gen_id;  // Update from SERJIO
            *put_back = true;
            rv = 0;
        }
        break;
    }
    
    if (*put_back) {
        list_add_tail(&jidx->pool_link, &jam_disk->free_list);
        jam_disk->n_free++;
        
        // Resume any pending requests
        pending_req_resume(jam_disk, idx);
    }
    
    return rv;
}
```

### State Diagram

```
         +-> ALLOCED ----+
         |      |         |
     alloc     free      erase
         |      |         |
         |      v         v
      FREE <--  +      ERASING
         ^              |     |
         |    erase_ok  |     | erase_err
         +------+-------+     | (retry)
                              +--+
         +-> ABANDONED
         |      |
      abandon  |
         |      | free_abnd (SERJIO)
         +------+
                |
                v
              FREE
```

### Remote Erase

```c
static int jentry_erase(struct nvmeibc_disk *disk, struct nvmeibc_jam_jidx *jidx)
{
    struct nvmeibc_disk_gen_cmd *gen_cmd = kzalloc(...);
    
    gen_cmd->disk_cmd.cmd_type = NVMEIBC_DISK_CMD_GEN;
    gen_cmd->opcode = NVMEIB_GEN_OP_JENTRY_ERASE;
    gen_cmd->param.je.rng_gen_id = disk->jour.rng_gen_id;
    gen_cmd->param.je.rng_idx = disk->jour.rng_id;
    gen_cmd->param.je.ent_erase.ent_idx = jidx->idx;
    gen_cmd->param.je.ent_erase.ent_md.ent_gen_id = jidx->gen_id;
    gen_cmd->param.je.ent_erase.ent_swlba = 
        nvmeibc_jam_decode_lba(idx_2_lba(disk, jidx->idx));
    gen_cmd->timeout = NVMEIBC_JAM_CMD_TIMEOUT;
    gen_cmd->disk = disk;
    
    // Execute generic command (async)
    rv = nvmeibc_pd_execute_gen(disk, gen_cmd);
    
    if (rv) {
        // Failed to issue - trigger disk release
        nvmeibc_disk_start_release(disk, 
            NVMEIBC_DISK_RELEASE_JOURNAL_ENTRY_ERASE_FAILURE);
    }
    
    return rv;
}
```

**Server-side**: Erases journal data, metadata, and JMDC entry.

### Erase Completion

```c
int nvmeibc_jam_jentry_erase_comp(struct nvmeibc_disk_gen_cmd *gen_cmd)
{
    struct nvmeibc_disk *disk = gen_cmd->disk;
    int idx = gen_cmd->param.je.ent_erase.ent_idx;
    
    if (gen_cmd->comp_code) {
        // Error
        jidx_event(disk, idx, NVMEIBC_JIDX_EVT_ERASE_COMP_ERR);
    } else {
        // Success
        jidx_event(disk, idx, NVMEIBC_JIDX_EVT_ERASE_COMP);
    }
    
    kfree(gen_cmd);
    return 0;
}
```

## Abandon (`nvmeibc_jam_abandon_lba`)

### When to Abandon

Called by block layer when:
- Transaction is aborted
- Write issued but failed
- Need to revert a partial write

### Abandon Flow

```c
int nvmeibc_jam_abandon_lba(struct nvmeibc_disk *disk, u64 jlba, u8 *gen_id)
{
    int idx = lba_2_idx(disk, jlba);
    
    // Transition to ABANDONED state
    rv = jidx_event(disk, idx, NVMEIBC_JIDX_EVT_ABANDON);
    
    // gen_id is returned for SERJIO notification
    // Block layer will notify SERJIO about abandoned entry
    
    return rv;
}
```

### SERJIO Integration

1. **Client abandons**: Marks entry as ABANDONED, returns gen_id
2. **Block layer notifies SERJIO**: Sends abandoned entry info to server
3. **SERJIO cleans**: Journal garbage collector erases abandoned entries
4. **SERJIO responds**: Sends free-abandoned message back to client
5. **Client processes**: `nvmeibc_jam_process_recv_comp` -> `NVMEIBC_JIDX_EVT_FREE_ABND`

```c
int nvmeibc_jam_process_recv_comp(struct nvmeibc_disk *disk,
                                 struct volume_server_req *req)
{
    struct volume_server_cmd_jmd_free_abnd *free_abnd = &req->cmd.jmd_free_abnd;
    
    // Decode free-abandoned message
    for (i = 0; i < free_abnd->n_ents; i++) {
        int idx = free_abnd->ent_idx[i];
        u8 gen_id = free_abnd->ent_gen_id[i];
        
        // Free abandoned entry
        jidx_event_(disk, idx, &gen_id, NVMEIBC_JIDX_EVT_FREE_ABND, ...);
    }
    
    return 0;
}
```

## JAM Disk Lifecycle

### Creation (`nvmeibc_jam_disk_add`)

Called during disk discovery after receiving journal range from server.

```c
int nvmeibc_jam_disk_add(struct nvmeibc_disk *disk,
                         unsigned long *free_ents_bmp,
                         union jblock_md *jmdc_tbl,
                         struct nvmeib_jrnl_ent_md *ent_md)
{
    struct nvmeibc_jam *c_jam = cdisk2cj(disk);
    struct nvmeibc_jam_disk *jam_disk;
    int jidx_pool_size = NUM_JENTS_JAM_USES_IN_JRI(disk);
    
    // Allocate jam_disk
    jam_disk = kzalloc(sizeof(*jam_disk), GFP_KERNEL);
    jam_disk->jidx_pool = kzalloc(jidx_pool_size * sizeof(*jidx_pool), ...);
    jam_disk->jidx_pool_size = jidx_pool_size;
    jam_disk->binje_shift = ilog2(disk->jour.rng_binje);
    
    spin_lock_init(&jam_disk->lock);
    INIT_LIST_HEAD(&jam_disk->free_list);
    hash_init(jam_disk->uniqj2b_htable);
    INIT_LIST_HEAD(&jam_disk->uniqj2b_trans_list);
    jam_disk->pending_root = RB_ROOT;
    
    // Create workqueue
    jam_disk->jwq = wq_create(...);
    
    // Initialize journal indices
    for (i = 0; i < jidx_pool_size; i++) {
        jidx_pool[i].idx = i;
        jidx_pool[i].gen_id = ent_md[i].ent_gen_id;
        INIT_LIST_HEAD(&jidx_pool[i].pool_link);
        jidx_pool[i].jreq_bound_via_committed = NULL;
        jidx_pool[i].jreq_bound_via_transient = NULL;
        jidx_hkeys_reset_both(&jidx_pool[i]);
        
        // Determine state from free bitmap
        if (test_bit(i, free_ents_bitmap)) {
            jidx_pool[i].state = NVMEIBC_JIDX_STS_FREE;
            list_add_tail(&jidx_pool[i].pool_link, &jam_disk->free_list);
            jam_disk->n_free++;
        } else {
            // Check JMDC table to distinguish ALLOCATED vs ABANDONED
            if (is_jmdc_abandoned(&jmdc_tbl[i])) {
                jidx_pool[i].state = NVMEIBC_JIDX_STS_ABANDONED;
                jam_disk->n_abandoned++;
            } else {
                // Assume allocated (will be freed by block layer)
                jidx_pool[i].state = NVMEIBC_JIDX_STS_ALLOCED;
                jam_disk->n_alloced++;
            }
            // Add to hash table
            uniqj2b_add(...);
        }
    }
    
    // Check if binje matches requested
    if (disk->binje_ulp != disk->jour.rng_binje) {
        if (jam_disk->n_abandoned == 0) {
            // Mismatch without abandoned entries is error
            goto err;
        }
        // Disable allocation until abandoned entries are cleaned
        jam_disk->alloc_enb = false;
    } else {
        jam_disk->alloc_enb = true;
    }
    
    // Add to global JAM
    disk->jam_disk = jam_disk;
    jam_disk->disk = disk;
    list_add_sorted(&jam_disk->jam_link, &c_jam->disks);
    c_jam->n_disks++;
    
    return 0;
}
```

### Destruction (`nvmeibc_jam_disk_del`)

Called during disk release before destroying disk.

```c
void nvmeibc_jam_disk_del(struct nvmeibc_disk *disk)
{
    struct nvmeibc_jam *c_jam = cdisk2cj(disk);
    struct nvmeibc_jam_disk *jam_disk = disk->jam_disk;
    LIST_HEAD(cancel_list);
    
    // Cancel all pending requests that involve this disk
    list_for_each_entry(jdisk, &c_jam->disks, jam_link) {
        for (node = rb_first(&jdisk->pending_root); node; node = rb_next(node)) {
            jreq = container_of(node, struct nvmeibc_jam_pending_req, pending_node);
            for (i = 0; i < jreq->n_disks; i++) {
                if (jreq->sorted[i].disk == disk) {
                    list_add_tail(&jreq->tmp_link, &cancel_list);
                    break;
                }
            }
        }
    }
    
    // Remove from global JAM
    c_jam->n_disks--;
    list_del(&jam_disk->jam_link);
    
    // Cancel pending requests
    list_for_each_entry_safe(jreq, tmp, &cancel_list, tmp_link) {
        del_timer_sync(&jreq->timeout_timer);
        pending_req_cancel(jreq);  // Rollback, callback error, free
    }
    
    // Wait for deferred work to complete
    wait_event(jam_disk->deferred_jreqs, 
              atomic_read(&jam_disk->deferred_jreqs_cnt) == 0);
    wq_drain(jam_disk->jwq);
    
    // Fill cache before destroying
    jam_disk_cache_fill(disk);
    
    // Cleanup
    nvmeib_public_free_percpu(jam_disk->pcpu_cnts);
    kfree(jam_disk->jidx_pool);
    wq_destroy(jam_disk->jwq);
    kfree(jam_disk);
    disk->jam_disk = NULL;
}
```

**Critical**: `jam_disk_cache_fill(disk)` preserves state in `disk->jrc` for next rediscovery.

## Module Parameters

```bash
# Pending request timeout (jiffies)
jam_pending_req_timeout_jif=40    # Default: HZ/5 (200ms)

# Max used entries (0 = no limit)
jam_max_used_entries=0

# Timeout for non-free entries (seconds)
jam_non_free_entry_timeout=300

# Metrics logging period (seconds, 0 = disabled)
jam_log_metrics_period=0

# Enable/disable pending mode
jam_pending_enb=true

# Use system per-CPU workqueue for pending
jam_use_system_pcpu_wq=false
```

## Debugging

### Procfs Interface

```bash
# Per-disk JAM status
cat /proc/nvmesh/jam/<disk_name>

# Global JAM status
cat /proc/nvmesh/status | grep -A 50 "^JAM"
```

### Trace Events

```bash
# Enable JAM tracing
echo 1 > /sys/kernel/debug/tracing/events/nvmesh/*jam*/enable

# Watch JAM operations
cat /sys/kernel/debug/tracing/trace_pipe | grep jam
```

### Key Traces

- `trace_jam_lbas_alloc`: Allocation requests and results
- `trace_jam_jidx_alloc`: Per-entry allocation
- `trace_jam_jidx_event`: State transitions
- `trace_jam_pending_req_*`: Pending request lifecycle
- `trace_jam_jentry_erase`: Erase operations

### Statistics

Per-disk stats (in procfs output):
- `n_free`, `n_alloced`, `n_abandoned`, `n_erasing`: Current counts
- `n_pending`: Queued requests
- `tot_alloc`, `tot_free`, `tot_erase`, `tot_abnd`: Lifetime totals
- `n_pending_timeout`: Timed-out requests
- Averages: allocated/erasing durations

Global stats (`/proc/nvmesh/status`):
- Per-CPU counters for ULP operations
- Total journal entries allocated/freed
- Rollback and cancel counts

## Performance Considerations

### Hash Table Size

```c
DECLARE_HASHTABLE(uniqj2b_htable, 8);  // 256 buckets
```

- Good for typical journal sizes (128-512 entries)
- O(1) average lookup
- Prevents allocation collisions

### Pending Request Ordering

- Priority-based (higher first)
- Within same priority, deadline-based (earlier first)
- RB-tree for efficient insertion/removal

### Lock Granularity

- **Global JAM lock**: Only for disk list modifications
- **Per-disk locks**: For all journal operations
- Spinlocks (can be called from softirq)
- Short critical sections

### Per-CPU Counters

Avoid cache-line bouncing for statistics:

```c
struct nvmeibc_jam_percpu_cnts __percpu *pcpu_cnts;

#define jam_cnts_inc(__j, __name) do {
    unsigned long __f;
    local_irq_save(__f);
    __p = this_cpu_ptr(__j->pcpu_cnts);
    __p->__name++;
    local_irq_restore(__f);
} while (0)
```

### Workqueue for Deferred Operations

- Per-disk workqueue (`jwq`)
- Used for pending request resume
- Prevents blocking in completion context

## Error Handling

### Allocation Failures

- **-ENOMEM**: Memory allocation failed (fatal)
- **-EBUSY**: Temporary (no free entries or hash conflict) -> pend
- **-EDEADLK**: Deadlock with abandoned entry (can't wait) -> fail fast
- **-ETIMEDOUT**: Pending request timed out -> fail
- **-EINPROGRESS**: Request successfully queued

### Erase Failures

- Retry up to `NVMEIBC_JAM_MAX_ERASE_ATTEMPTS` (3 times)
- If all retries fail: trigger disk release
- Reason: `NVMEIBC_DISK_RELEASE_JOURNAL_ENTRY_ERASE_FAILURE`

### Disk Release During Operations

- Pending requests are canceled
- Allocated entries remain allocated (leaked in short term)
- Cache preserves state for rediscovery
- SERJIO will eventually clean up server-side

## Related Documentation

- `DISK_DISCOVERY_REDISCOVERY.md`: JAM creation during discovery (Stage 12)
- `DISK_UPDATES.md`: JAM preservation during updates
- EC datapath docs: JAM API usage
- SERJIO documentation: Server-side journal garbage collection

## Future Improvements (TODOs in Code)

1. Replace hash table with Bloom Filter for faster negative lookups
2. Keep `jam_disk` across rediscovery (don't destroy/recreate)
   - Would eliminate need for `jrc` cache
   - Just disable/enable allocation
3. Separate simulator code into dedicated file
4. Replace `BUG_ON` with error returns for better resilience
5. Reduce journal range information (JRI) size back to 128 entries

