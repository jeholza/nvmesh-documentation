# NORDDA Channel Documentation

## Overview

The NORDDA (No-RDDA) Channel is a specialized RDMA-based communication channel used for disk I/O operations in NVMesh. 

### NORDDA vs RDDA

- **RDDA (Remote Disk Direct Access)**: A method where the NVMe submission and completion queues are controlled remotely by the client over RDMA. The client has direct access to the target's NVMe queues and can post commands directly to the disk's doorbell registers.

- **NORDDA (No-RDDA)**: Does NOT use Remote Disk Direct Access. Instead, it uses a traditional command/response model over RDMA with bounce buffers on the target side. The client sends I/O commands via RDMA messages, and the target processes them through its local NVMe queues.

Despite the "No-RDDA" name, NORDDA extensively uses RDMA for data transfer - it just doesn't give clients direct access to the NVMe queues.

## Key Characteristics

- **Data Path**: Client ↔ RDMA ↔ Target Bounce Buffer ↔ Target NVMe Queue ↔ Disk
- **Queue Control**: Target-side (not client-controlled like RDDA)
- **Operations Supported**: 
  - I/O Operations (read, write, discard, write_uncor, metadata operations)
  - Generic Commands (journal operations, EC operations, lock commands)
  - Lock Operations (via bypass mechanism)
- **Request Management**: Version-tagged requests with double-completion handling
- **Reuse Support**: Optimized journal write → data write reuse path
- **Completion Modes**: Deferred receive completions, completion offload, per-CPU channels

## Architecture

### Channel Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                   NORDDA Channel System                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐         ┌──────────────────────────────┐  │
│  │   Lionic     │────────>│   Per-LIONIC NORDDA Channels │  │
│  │  (Local NIC) │         │   - One per target RNIC      │  │
│  │              │         │   - Multiple channels per    │  │
│  │              │         │     connection (QPs)         │  │
│  └──────────────┘         └──────────────────────────────┘  │
│         │                                                     │
│         ▼                                                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │           Request Pool                                │   │
│  │  - Fixed number of requests per channel              │   │
│  │  - Version tagging for double-comp prevention        │   │
│  │  - Bitmap tracking in-use requests                   │   │
│  └──────────────────────────────────────────────────────┘   │
│         │                                                     │
│         ▼                                                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │           Data Path (I/O Operations)                  │   │
│  │                                                        │   │
│  │  READ:                                                │   │
│  │    1. SEND command with target buffer descriptors    │   │
│  │    2. Target RDMA_WRITE data to client               │   │
│  │    3. Target SEND response                           │   │
│  │                                                        │   │
│  │  WRITE:                                               │   │
│  │    1. RDMA_WRITE data to target bounce buffer        │   │
│  │    2. SEND command                                    │   │
│  │    3. Target SEND response                           │   │
│  └──────────────────────────────────────────────────────┘   │
│         │                                                     │
│         ▼                                                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │           Completion & Reuse                          │   │
│  │  - Send completion handling                          │   │
│  │  - Receive completion handling                       │   │
│  │  - Release counter (2: send + recv)                  │   │
│  │  - Request reuse for journal → data writes           │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow

#### Read Operation

```
Client                                          Target
  │                                               │
  ├─> 1. SEND IO_READ command                   │
  │      - Remote buffer descriptors             │
  │      - Lock piggyback (optional)             │
  │                                               │
  │                                          2. Process
  │                                          3. Read disk
  │                                               │
  │   <──── 4. RDMA_WRITE data ───────────────────┤
  │         (to client's buffer)                  │
  │                                               │
  │   <──── 5. SEND response ─────────────────────┤
  │         - Completion code                     │
  │         - Lock piggyback result               │
  │                                               │
  └─> 6. Complete to block layer                 │
```

#### Write Operation

```
Client                                          Target
  │                                               │
  ├──── 1. RDMA_WRITE data ───────────────────>  │
  │      (to target bounce buffer)                │
  │                                               │
  ├─> 2. RDMA_WRITE JMDC piggyback (optional)   │
  │                                               │
  ├─> 3. SEND IO_WRITE command                  │
  │      - Lock piggyback (optional)             │
  │                                               │
  │                                          4. Process
  │                                          5. Write disk
  │                                               │
  │   <──── 6. SEND response ─────────────────────┤
  │         - Completion code                     │
  │         - Lock piggyback result               │
  │                                               │
  └─> 7. Complete to block layer                 │
```

## Data Structures

### struct nvmeibc_ib_nordda_channel

The main nordda channel structure:

```c
struct nvmeibc_ib_nordda_channel {
    struct nvmeibc_channel base;             // Base channel
    struct nvmeibc_ib_net_nordda net;        // Network connection
    
    // Request management
    struct nvmeibc_volume_req_info *reqs;    // Request array
    struct list_head free_reqs;              // Free request list
    DECLARE_BITMAP(req_in_use, NVMEIB_MAX_NORDDA_IO_REQ);
    
    spinlock_t guard;                        // Channel lock
    int n_used_reqs;                         // Requests in use
    int n_uses_ever;                         // Total uses
    unsigned long n_reuse_bb_returned;       // Reused buffers returned
    
    // Availability tracking
    struct list_head available_link;         // Link in disk's available list
    bool inuse;                              // Channel in use
    
    // Completion handling
    struct completion init_comp;             // Initialization complete
    
    // SRQ support
    struct nvmeib_srq_info *priv_srq;        // Private SRQ (optional)
    struct nvmeib_recv_q *recv_q;            // Receive queue (if no SRQ)
    
    // Deferred receive completions
    struct workqueue_struct *rc_wq;          // Recv completion work queue
    
    // Release work queue
    struct workqueue_struct *release_wq;     // Channel release work queue
    
    // Remote resources
    struct nvmeib_rai ka_rai;                // Keep-alive remote info
    
    // Connection info
    struct nvmeibc_io_lnic *lionic;          // Local NIC
    int max_io_sz;                           // Max I/O size
    
    // Priority
    union nvmeibc_rionic_priority priority;  // Channel priority
    
    // Watchdog
    atomic64_t last_received;                // Last receive time
    atomic64_t last_watchdog_warning;        // Last watchdog warning
    
    // Per-CPU support
    int pcpu_cpu;                            // Per-CPU channel CPU (-1 if not per-CPU)
    struct work_struct pcpu_connect_work;    // Per-CPU connect work
    struct completion pcpu_connect_comp;     // Per-CPU connect completion
    
    // Latency measurement
    struct nvmeibc_nr_lat_meas_nrch_per_cpu_lat_data __percpu *per_cpu_lat_data;
    
    // Configuration
    bool wait_release_zero_before_cb;        // Wait for both send+recv before callback
};
```

### struct nvmeibc_volume_req_info

Represents a single I/O request:

```c
struct nvmeibc_volume_req_info {
    struct nvmeibc_volume_request req;       // Base request
    struct nvmeibc_ib_nordda_channel *nrch;  // Parent channel
    
    int idx;                                 // Index in channel array
    u16 version;                             // Request version (for double-comp check)
    
    // Remote bounce buffer
    u64 raddr;                               // Remote address
    u32 rkey;                                // Remote key
    int rn_pages;                            // Number of pages
    
    // Completion tracking
    int release_counter;                     // 2 = send + recv pending
    u64 send_counter;                        // Total sends
    u64 send_comp_counter;                   // Send completions
    bool recv_comp_arrived;                  // Receive completion arrived
    u64 n_send_comp;                         // Send completions (after recv)
    u64 n_recv_comp;                         // Receive completions
    
    // Watchdog
    struct wd_info_common wdc;               // Watchdog context
    int n_wd_events;                         // Watchdog events
    bool wd_timeout_occurred;                // Watchdog timeout
    
    // Reuse support
    struct nvmeibc_volume_req_reuse_orig reuse_orig;  // Original SG/FR for reuse
    bool reused_bb_wait_send_comp;           // Wait for journal send comp
    unsigned long reused_bb_wait_send_comp_start_cnt;
    unsigned long reused_bb_wait_send_comp_finish_cnt;
    
    // JMDC piggyback
    struct nvmeibc_ib_net_jmdc_piggyback jmdc_pb;
    
    // Work requests
    struct nvmeib_send_wr wr[NVMEIBC_NR_CH_RDMA_MAX_WRS];
    struct ib_sge md_sg;                     // Metadata SGE
    
    // Latency measurement
    struct nvmeibc_nr_lat_meas_req lat_meas;
    
    // Per-CPU pending I/O
    struct nvmeibc_smp_call_data pcpu_pending_io_smp_call;
    
    // Completion code
    int comp_code;                           // Final completion code
    
    // Locking (for per-CPU lockless channels)
    spinlock_t lock;
    int locking_pid;                         // PID holding lock (-1 if none)
    
    // Statistics
    struct nvmeib_stats send;
};
```

### struct nvmeibc_ib_net_nordda

Network connection for nordda channel:

```c
struct nvmeibc_ib_net_nordda {
    struct nvmeibc_ib_net base;              // Base network
    struct nvmeibc_ib_nordda_channel *nrch;  // Parent channel
    
    // JMDC (Journal Metadata Cache) piggyback
    struct nvmeib_rai jmdc_rai;              // JMDC remote info
    
    // VEX operations
    const struct vex_ops *vex_nrio_ops[vex_nrch_max];
};
```

## Request Versioning and Double-Completion Prevention

### Version Tagging

Each request has a version number that increments on each use:

```c
#define NVMEIB_INC_TAG_VERSION(version) \
    do { \
        (version)++; \
        if ((version) == NVMEIB_TAG_VERSION_RESERVED) \
            (version)++; \
    } while (0)
```

- Version starts at 0
- Increments before each use
- Skips `NVMEIB_TAG_VERSION_RESERVED` (0xFFFF)
- Encoded in WR ID and command tag

### Tag Encoding

```c
u64 nordda_tag_encode(
    const struct nvmeibc_disk_last_target_ver *tgt_ver,
    u32 ch_version,
    u16 req_version,
    u16 req_index
);
```

Tag format (64-bit):
```
┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐
│  Target Ver(16) │  Ch Ver (16)    │  Req Ver (16)   │  Req Index (16) │
└─────────────────┴─────────────────┴─────────────────┴─────────────────┘
```

### Double-Completion Check

Completion handler verifies:

1. **Request index** matches
2. **Request version** matches (or is RESERVED)
3. **Channel version** matches (or is RESERVED)
4. **Release counter** is valid

If version mismatch, completion is for a previous use and is ignored.

## Request Release Counter

Each request uses a release counter to track completions:

```c
info->release_counter = 2;  // send + recv
```

### Release Counter States

| Value | Meaning |
|-------|---------|
| 2 | Send and receive both pending |
| 1 | One completion arrived, one pending |
| 0 | Both completions arrived, ready for callback/reuse |

### Decrement Flow

```
Initial: release_counter = 2
   │
   ├─> Send completion: --release_counter (now 1)
   │   └─> If 0: free request, finalize
   │
   └─> Recv completion: --release_counter (now 1 or 0)
       └─> If 0: free request, finalize
```

## Request Reuse Optimization

### Journal Write → Data Write Reuse

For journal writes followed by data writes, requests can be reused:

```
1. Journal Write:
   │
   ├─> Map data, map metadata
   ├─> RDMA_WRITE journal data
   ├─> SEND command (reused_bb=0)
   │
   └─> Recv completion arrives
       │
       ├─> do_reuse_request() == true
       │   │
       │   ├─> Store aside SG list: REUSE_SG_FR_STORE()
       │   ├─> Free RDMA resources
       │   ├─> Complete to ULP with STATS_DONE_LLP_COMPLETE_IO_RESPONSE_BUF_SAVE
       │   ├─> Increment version
       │   └─> Add to reuse LRU: init_reuse_request()
       │
       └─> (Send completion may arrive after recv)
           └─> If reused_bb_wait_send_comp:
               - Wait for journal send comp
               - Then process pending I/O

2. Data Write (reuses request):
   │
   ├─> Get reused request: do_reuse_request()
   ├─> Restore SG list: REUSE_SG_RESTORE()
   ├─> Restore FR descs: REUSE_FR_RESTORE()
   ├─> RDMA_WRITE data (use existing mapping)
   ├─> SEND command (reused_bb=1, use req->reuse_cmd)
   │
   └─> Recv completion arrives
       │
       ├─> del_reuse_request()
       ├─> Complete to ULP with STATS_DONE_LLP_COMPLETE_IO_RESPONSE_BUF_REUSE_DEL
       └─> Normal release flow
```

### Reuse Conditions

Request is reusable if:

1. Journal write operation (determined by ULP)
2. No errors in journal write completion
3. `do_reuse_request()` returns true
4. Disk supports BB reuse

### Wait for Send Completion

When journal recv arrives before send:

```c
if (req->send_comp_counter < req->send_counter) {
    req->reused_bb_wait_send_comp = true;
    // Will process pending when send comp arrives
}
```

This prevents LOC_PROT errors from reusing FR descriptors before send completes.

## I/O Operations

### Read Operation

#### Request Encoding

VEX operations for encoding:
- **Base**: `vex_nrch_io_read_clnt_base_encode`
  - Disk name
  - Start LBA, data length
  - Buffer descriptors (direct or indirect)
  - Metadata descriptors
  - Lock piggyback (optional)
- **Ext1**: `vex_nrch_io_read_clnt_ext1_encode`
  - Sub-block operations
- **Ext2**: `vex_nrch_io_read_clnt_ext2_encode`
  - Recovery flag

#### Buffer Descriptors

**Direct Buffer** (single SG entry):
```c
struct nvmeib_direct_buf {
    u64 va;      // Virtual address
    u32 len;     // Length
    u32 key;     // RDMA key
};
```

**Indirect Buffer** (multiple SG entries):
```c
struct nvmeib_indirect_buf {
    u32 len;                              // Total length
    u16 table_count;                      // Number of entries
    struct nvmeib_direct_buf table_desc;  // Table descriptor
    struct nvmeib_direct_buf desc_list[]; // SG list
};
```

#### Metadata Mapping

For reads with metadata:
```c
if (req->md.has) {
    nvmeibc_ib_net_map_md(&net->base, req, io_len,
        NVMEIBC_SECTOR_SHIFT, disk->sector_shift);
    
    io_req->md_desc.raddr = cpu_to_be64(req->md.local.addr);
    io_req->md_desc.size = cpu_to_be32(req->md.local.size);
    io_req->md_desc.rkey = cpu_to_be32(req->md.local.rkey);
}
```

### Write Operation

#### Data Transfer

Writes use RDMA_WRITE to transfer data to target bounce buffer:

```c
for (i = 0; i < iu->n_rdma_iu; ++i) {
    nvmeib_send_wr_common(wrh[i]).opcode = IB_WR_RDMA_WRITE;
    nvmeib_send_wr_rdma(wrh[i]).remote_addr = info->raddr + offset;
    nvmeib_send_wr_rdma(wrh[i]).rkey = info->rkey;
    // ... setup SGE
}
```

#### Metadata Handling

**Extended Metadata** (inline with data):
- Included in RDMA_WRITE of data
- No separate operation needed

**Separated Metadata**:
```c
if (wr_op_sep_md) {
    // Additional RDMA_WRITE for metadata
    nvmeib_send_wr_common(wr).opcode = IB_WR_RDMA_WRITE;
    nvmeib_send_wr_rdma(wr).remote_addr = req->md.remote.addr;
    nvmeib_send_wr_rdma(wr).rkey = req->md.remote.rkey;
}
```

#### JMDC Piggyback

For journal writes, JMDC (Journal Metadata Cache) can be piggybacked:

```c
if (NVMEIBC_NR_CH_RDMA_WRITE_JMDC_PB) {
    nvmeibc_ib_net_jmdc_pb_fill_wrs(&net->base, req->bcmd,
        &net->jmdc_rai, &info->jmdc_pb, &wrh[i]);
}
```

This RDMA_WRITEs journal metadata to target's JMDC area.

### Generic Commands

Generic commands support various operations:

#### GET_UUID_JOUR (Get Journal for UUID)

Request:
```c
struct volume_client_gen_req_uuid_jour_base {
    uuid_be client_uuid;                  // Client UUID
    uuid_be sgmnt_uuid[...];              // Segment UUIDs
    struct nvmeib_rai jmdc_rai;           // JMDC destination
    struct nvmeib_rai ent_md_rai;         // Entry metadata destination
};
```

Response:
```c
struct volume_server_gen_rsp_uuid_jour_base {
    uuid_be uuid;                         // Client UUID
    u32 rng_id;                           // Range ID
    u64 rng_slba;                         // Range start LBA
    u32 rng_nlba;                         // Range size
    u64 rng_gen_id;                       // Range generation ID
    u8 dirty_ents_bitmap[...];            // Dirty entries
    u8 abnd_ents_bitmap[...];             // Abandoned entries
    u32 jmdc_len;                         // JMDC length
    u32 ent_md_len;                       // Entry metadata length
};
```

#### BLKSET_RECOVERED

Notify target that blockset recovery completed:

```c
struct volume_client_gen_req_blkset_recovered_base {
    uuid_be uuid;                         // Client UUID
    uuid_be ds_uuid;                      // Disk set UUID
    u64 blkset_num;                       // Blockset number
    u64 blkset_slba;                      // Blockset start LBA
    u64 lock_ent;                         // Lock entry
    u32 rng_id;                           // Range ID
    u32 ent_id;                           // Entry ID
    bool pass2toma;                       // Pass to TOMA
};
```

#### GET_EC_DB (Get EC Dirty Bits)

Request dirty bit information for EC volumes:

```c
struct disk_req_get_ec_dirty_bits_base {
    u64 lba;                              // Start LBA
    u32 sectors;                          // Number of sectors
    bool get_dbits;                       // Get dirty bits
    bool get_stales;                      // Get stale info
    bool get_full_val;                    // Get full values
    struct nvmeib_rai rai;                // Destination
};
```

#### FREE_JRNL_ENTS (Free Journal Entries)

Free journal entries after commit:

```c
struct volume_client_gen_req_free_ents_base {
    u8 seg_uuid[NVMEIB_GID_STR_MAX];      // Segment UUID
    u8 src;                               // Source
    u16 num_ents;                         // Number of entries
    bool pass2toma;                       // Pass to TOMA
    u64 blkset_num;                       // Blockset number
    u64 blkset_slba;                      // Blockset start LBA
    u64 lock_ent;                         // Lock entry
    u8 serjio_boot_id[NVMEIB_GID_STR_MAX];
    // Followed by array of wire_free_ents_entry
};
```

#### JENTRY_ERASE (Erase Journal Entry)

Erase a journal entry:

```c
struct volume_client_gen_req_jentry_erase_base {
    u64 rng_gen_id;                       // Range generation ID
    u32 rng_id;                           // Range ID
    u32 ent_id;                           // Entry ID
    struct jblock_entry_md ent_md;        // Entry metadata
    u64 sw_jlba;                          // Software journal LBA
};
```

### Lock Operations (via Bypass)

Lock operations can be sent via nordda channel using RPC bypass:

```c
struct volume_client_lock_req_base {
    enum nvmeib_lock_op op;               // Operation type
    u64 offset;                           // Lock offset
    struct {
        u64 compare_add;                  // Compare/Add value
        u64 swap;                         // Swap value
        u64 compare_add_mask;             // Compare/Add mask
        u64 swap_mask;                    // Swap mask
    } atomic;
    u32 lmi;                              // Lock memory index (base)
};

struct volume_client_lock_req_ext1 {
    u64 seg_id;                           // Segment ID (ext1)
    struct {
        u32 len;                          // RDMA data length
        u8 data[...];                     // RDMA data
        u8 table_type;                    // Table type
    } rdma;
};
```

Response:
```c
struct volume_server_lock_rsp_base {
    u64 cmp_swap_val;                     // Compare/Swap result
    u32 comp_code;                        // Completion code
};

struct volume_server_lock_rsp_ext1 {
    u8 read_data[...];                    // Read data
    u32 read_len;                         // Read length
};
```

## Completion Handling

### Send Completion

```c
static int nordda_send_completion(
    struct nvmeibc_ib_net *net,
    struct ib_wc *wc,
    bool last_in_series
)
```

Send completion handles:

1. **Decode WR ID**:
   ```c
   opcode = nordda_wr_id_decode_opcode(wc->wr_id);
   index = nordda_wr_id_decode_index(wc->wr_id);
   version = nordda_wr_id_decode_version(wc->wr_id);
   reused = nordda_wr_id_decode_reused(wc->wr_id);
   ```

2. **Update timestamps**:
   ```c
   this_cpu_write(*ch->lionic->last_send_success_jif, jiffies);
   ```

3. **Handle different opcodes**:
   - `NVMEIB_SEND_IO`: I/O operation send
   - `NVMEIB_RDMA_IO_KA`: Keep-alive operation
   - `NVMEIB_RDMA_LAST`, `NVMEIB_RDMA_MID_*`: Data transfer
   - `NVMEIB_RDMA_METADATA`: Metadata transfer
   - Gen cmd opcodes: Generic commands

4. **Process completion**:
   ```c
   if (--req->release_counter == 0) {
       // Both send and recv completed
       nvmeibc_ib_nordda_channel_req_stop_wd(req);
       nvmeibc_ib_net_free_req(&ch->net.base, &req->req);
       process_rsp_finalize(ch, req);
   }
   ```

### Receive Completion

```c
static int nordda_recv_completion(
    struct nvmeibc_ib_net *net,
    struct ib_wc *wc
)
```

Receive completion:

1. **Update receive timestamp**:
   ```c
   atomic64_set(&ch->last_received, jiffies);
   this_cpu_write(*ch->lionic->last_recv_success_jif, jiffies);
   ```

2. **Process response**:
   ```c
   put_back = process_rsp(ch, iu, wc);
   ```

3. **Repost receive buffer**:
   ```c
   if (put_back)
       nvmeibc_ib_nordda_channel_put_rx_iu(ch, iu);
   ```

### Response Processing

```c
static int process_rsp(
    struct nvmeibc_ib_nordda_channel *ch,
    struct nvmeib_iu *iu,
    struct ib_wc *wc
)
```

1. **Decode tag**:
   ```c
   rsp_tag = be64_to_cpu(rsp->hdr.tag);
   req_index = nordda_tag_decode_index(rsp_tag);
   req_version = nordda_tag_decode_version(rsp_tag);
   req_ch_version = nordda_tag_decode_ch_version(rsp_tag);
   ```

2. **Validate**:
   - Request index in range
   - Version matches (or RESERVED)
   - Channel version matches (or RESERVED)

3. **Extract completion code**:
   ```c
   if (rsp->opcode == NVMEIBS_RSP_IO_OPCODE_OK ||
       rsp->opcode == NVMEIBS_RSP_IO_OPCODE_ERR) {
       req->comp_code = be32_to_cpu(rsp->io_rsp.base.comp_code);
   }
   ```

4. **Process by command type**:
   - **I/O**: `process_io_rsp()`
   - **Gen**: `process_gen_rsp()`
   - **Lock**: `process_lock_rsp()`

5. **Update completion tracking**:
   ```c
   req->n_recv_comp++;
   req->recv_comp_arrived = true;
   ```

6. **Handle completion**:
   ```c
   if (--req->release_counter == 0) {
       // Both send and recv completed
       nvmeibc_ib_nordda_channel_req_stop_wd(req);
       nvmeibc_ib_net_free_req(&ch->net.base, &req->req);
       process_rsp_finalize(ch, req);
   }
   ```

### Finalization

```c
static void process_rsp_finalize(
    struct nvmeibc_ib_nordda_channel *ch,
    struct nvmeibc_volume_req_info *req
)
```

1. **Handle completion mode**:
   - If `wait_release_zero_before_cb`: Complete now
   - Else: Already completed in receive handler

2. **Check for reuse**:
   ```c
   if (comp_code == 0 && do_reuse_request(&req->req)) {
       init_reuse_request(...);
       // Don't free yet, save for reuse
       return;
   }
   ```

3. **Update latency stats**:
   ```c
   nvmeibc_nr_lat_meas_update_nrch_pcpu_data(...);
   ```

4. **Try pending I/O**:
   ```c
   if (nvmeibc_channel_try_use_req_info(&ch->base)) {
       nordda_pending_io(ch->base.disk, &ch->base, req, false, 0);
   }
   ```

## Deferred Completion Mode

### Configuration

```c
bool nr_defer_recv_comps;        // Defer recv comps (IB)
bool nr_defer_recv_comps_tcp;    // Defer recv comps (TCP)
```

### Deferred vs. Inline

**Inline Mode** (`nr_defer_recv_comps=false`):
- Completion handled in interrupt context
- Callback executed immediately
- Lower latency
- Limited processing in interrupt

**Deferred Mode** (`nr_defer_recv_comps=true`):
- Completion handled in work queue thread
- Allows blocking operations in callbacks
- Slightly higher latency
- More flexible processing

### Work Queue Creation

```c
if (params->nr_defer_recv_comps) {
    ch->rc_wq = wq_create_on(pname, params->comp_cpu);
    params->defer_recv_intr_wq = ch->rc_wq;
}
```

Receive completions are scheduled on `rc_wq` thread.

## Wait Release Zero Before Callback Mode

### Configuration

```c
ch->wait_release_zero_before_cb = nvmeibc_iommu_enabled ||
    (NVMEIB_SIW_NRCH_WAIT_RLS_ZERO_BEFORE_CB && is_siw);
```

### Completion Timing

**wait_release_zero_before_cb = false** (default):
```
Recv arrives first:
  │
  ├─> Stop WD
  ├─> Process response, extract completion code
  ├─> Handle reuse
  ├─> Complete to ULP
  ├─> Decrement release_counter (now 1)
  │
  └─> Send arrives later:
      ├─> Decrement release_counter (now 0)
      ├─> Free RDMA resources
      └─> Try pending I/O

Send arrives first:
  │
  ├─> Decrement release_counter (now 1)
  │
  └─> Recv arrives later:
      ├─> Stop WD
      ├─> Process response
      ├─> Complete to ULP
      ├─> Decrement release_counter (now 0)
      ├─> Free RDMA resources
      └─> Try pending I/O
```

**wait_release_zero_before_cb = true** (IOMMU/SIW):
```
Both completions must arrive before ULP callback:

Recv arrives first:
  │
  ├─> Process response, extract completion code
  ├─> Decrement release_counter (now 1)
  │
  └─> Send arrives later:
      ├─> Stop WD
      ├─> Decrement release_counter (now 0)
      ├─> Free RDMA resources
      ├─> Complete to ULP (now)
      └─> Try pending I/O

Send arrives first:
  │
  ├─> Decrement release_counter (now 1)
  │
  └─> Recv arrives later:
      ├─> Stop WD
      ├─> Process response
      ├─> Decrement release_counter (now 0)
      ├─> Free RDMA resources
      ├─> Complete to ULP (now)
      └─> Try pending I/O
```

This ensures RDMA resources are freed before ULP sees completion, preventing use-after-free with IOMMU.

## Watchdog

### Watchdog Initialization

```c
static void init_req_wd(
    struct nvmeibc_ib_nordda_channel *ch,
    struct nvmeibc_volume_req_info *req
)
{
    req->wdc.wd = ch->base.wd;
    req->wdc.cntx = req;
    req->wdc.on_start = on_start_wd_event;
    req->wdc.process = handle_watchdog_event_nordda;
    req->wdc.on_end = on_end_wd_event;
    nvmeib_wd_init_wdc(&req->wdc);
    nvmeib_wd_add_wdc(&req->wdc);
}
```

### Watchdog Event Types

```c
enum nr_wd_type {
    NR_WD_NONE,
    NR_WD_QP_TIMEOUT_SINCE_SEND,          // Time since send > QP timeout
    NR_WD_QP_TIMEOUT_SINCE_LAST_RECV,     // Time since last recv > QP timeout
    NR_WD_CHANNEL_TIMEOUT_SINCE_SEND,     // Time since send > channel timeout
};
```

### Timeout Logic

```c
static int handle_watchdog_event_nordda(
    void *cntx,
    unsigned long time_passed
)
```

1. **Check QP timeout** (short timeout):
   ```
   If (send_time > last_recv_time):
       If (time_since_send > qp_timeout):
           → Network issue, disconnect
   Else:
       If (time_since_last_recv > qp_timeout):
           → Network issue, disconnect
   ```

2. **Check channel timeout** (long timeout):
   ```
   If (network OK but time_since_send > channel_timeout):
       → Disk issue, disconnect
   ```

3. **Rescue timeout**:
   ```c
   if (time_passed > nvmeibc_nr_wd_rescue_timeout) {
       nveibc_ib_net_defer_recv_interrupts_external(...);
       // Trigger polling to catch missed events
   }
   ```

### Configuration

```bash
# Long timeout (considers last received)
/sys/module/nvmesh_ib_client/parameters/nr_wd_long_timeout
Default: 0 (uses NVMEIBC_IO_LONG_TIMEOUT)

# Rescue timeout (detect missing events)
/sys/module/nvmesh_ib_client/parameters/nr_wd_rescue_timeout
Default: 0 (disabled)
```

## Per-CPU Channels

### Configuration

Per-CPU channels can be enabled for lockless operation:

```bash
# Use per-CPU channels
/sys/module/nvmesh_ib_client/parameters/use_pcpu_cq
Default: false
```

### Per-CPU Channel Structure

```c
struct nvmeibc_ib_nordda_channel {
    int pcpu_cpu;                         // CPU this channel belongs to (-1 if not per-CPU)
    struct work_struct pcpu_connect_work; // Per-CPU connect work
    struct completion pcpu_connect_comp;  // Connect completion
};
```

### Lockless Operation

When per-CPU channels are enabled:

1. **No spinlock** needed for request submission
2. **IRQs must be disabled** during operation
3. **Request must be submitted from channel's CPU**

Verification:
```c
if (is_ll_pcpu_nrch(ch) && smp_processor_id() != pcpu_nrch_cpu_get(ch)) {
    // Wrong CPU, schedule on correct CPU
    nvmeib_public_smp_call_function_single_async(
        pcpu_nrch_cpu_get(ch), &info->pcpu_pending_io_smp_call.call_data);
}
```

### Connection

```c
int nvmeibc_disk_connect_nrch_pcpu(
    struct nvmeibc_disk *disk,
    struct nvmeibc_io_lnic *lionic,
    int qpn,
    int cpu
);
```

Connects per-CPU channel:
1. Schedules connect work on target CPU
2. Waits for completion
3. Assigns `pcpu_cpu = cpu`

## Keep-Alive

### Keep-Alive Remote Info

```c
struct nvmeib_rai ka_rai;
// Filled from login response:
nrch->ka_rai.raddr = be64_to_cpu(lrsp->base.nr_rsp.io_ka_raddr);
nrch->ka_rai.rkey = be32_to_cpu(lrsp->base.nr_rsp.io_ka_rkey);
nrch->ka_rai.len = sizeof(u64);
```

### Keep-Alive Execution

```c
static int nordda_channel_execute_ka(struct nvmeibc_channel *ch)
{
    struct nvmeibc_ib_nordda_channel *nrch = c_to_inrc(ch);
    u64 val = jiffies;
    return nvmeibc_ib_net_execute_ka(&nrch->net.base, &nrch->ka_rai, &val, sizeof(val));
}
```

RDMA_WRITE jiffies value to target's keep-alive area.

### Per-CPU Tracking

```c
this_cpu_write(*ch->lionic->last_io_ka_jif, jiffies);
```

Track last keep-alive per-CPU to avoid false timeouts.

## Connection Management

### Connection

```c
int nvmeibc_ib_nordda_channel_connect(
    struct nvmeibc_ib_nordda_channel *ch
)
```

1. **Find path**:
   ```c
   if (!ch->lionic->path_valid) {
       nvmeibc_disk_lionic_rionic_find_path(ch->lionic);
   }
   ```

2. **Setup network params**:
   ```c
   params->max_send_q = calc_nr_sq_size(ch, ...);
   params->max_send_sg = NVMEIBC_CHANNEL_MAX_MAIN_NORDDA_SG;
   params->use_srq = true;
   params->shared_cq = nr_shared_cq;
   params->nr_defer_recv_comps = nr_defer_recv_comps;
   ```

3. **Setup SRQ or RQ**:
   - If SRQ supported: Use device SRQ or create private SRQ
   - Else: Create receive queue

4. **Setup completion handling**:
   - If deferred: Create `rc_wq` work queue
   - Else: Use completion offload

5. **Fill login request**:
   ```c
   nvmeibc_login_req_init(&req, ...);
   nvmeibc_login_req_set_ioch(&req, sgid, dgid, ch->base.index, 0);
   ```

6. **Allocate network**:
   ```c
   nvmeibc_ib_net_nordda_alloc(&ch->net, params, &req);
   ```

7. **Complete initialization**:
   ```c
   complete(&ch->init_comp);
   ```

### Disconnection

```c
bool nvmeibc_ib_nordda_channel_try_disconnect(
    struct nvmeibc_ib_nordda_channel *ch
)
```

Triggers disconnect and schedules remove work:
```c
on_disconnect_nordda_ch(net) {
    net->remove_work.defer_release_wq_create = true;
    WQ_INIT_WORK(&net->remove_work.work, nordda_channel_remove_work);
    nvmeibc_admin_channel_add_work(net->admin_ch, &net->remove_work.work);
}
```

### Remove Work

```c
static void nordda_channel_remove_work(struct workqe_struct *work)
```

1. **Try create release WQ**:
   ```c
   if (defer_release_wq_create)
       nordda_channel_try_launch_release_wq(ch);
   ```

2. **Invalidate channel version**:
   ```c
   nvmeibc_disk_channel_version_invalidate(&ch->base);
   ```

3. **Wait for init completion**:
   ```c
   wait_for_completion(&ch->init_comp);
   ```

4. **Remove from available list**:
   ```c
   nvmeibc_disk_available_norddas_del(ch->base.disk, ch);
   ```

5. **Wait for used requests**:
   ```c
   nvmeibc_channel_wait_used_req_infos(&ch->base);
   ```

6. **Remove watchdog**:
   ```c
   remove_wdc(ch);
   ```

7. **Break QP**:
   ```c
   nvmeibc_ib_net_break_qp(net);
   ```

8. **Drain recv comps**:
   ```c
   rc_wq_destroy(ch);
   ```

9. **Free volume requests**:
   ```c
   nordda_channel_free_volume_reqs(ch);
   ```

10. **Free network**:
    ```c
    nvmeibc_ib_net_nordda_free(&ch->net);
    ```

11. **Mark unused**:
    ```c
    nvmeibc_ib_nordda_channel_end_use(ch);
    ```

12. **Speedup reconnect** (if not dying):
    ```c
    nvmeibc_disk_start_io_channels(disk, false);
    ```

## SRQ Support

### SRQ Selection

1. **Private SRQ** (if `max_nic_srqs > 1` and `USE_PRIV_SRQ`):
   ```c
   ch->priv_srq = nvmeib_srq_info_create(P2NV(lionic->port), &params, ch, c_nordda_srq_info);
   params->srq_priv = ch->priv_srq;
   ```

2. **Device SRQ** (default):
   ```c
   params->use_srq = true;
   params->srq_type = NVMEIB_SRQ_TYPE_SECONDARY;
   ```

3. **No SRQ** (fallback):
   ```c
   ch->recv_q = kzalloc(sizeof(*ch->recv_q), GFP_KERNEL);
   nvmeib_init_recvq(ch->recv_q, ...);
   params->recv_q = ch->recv_q;
   ```

### RX IU Management

```c
struct nvmeib_iu* nvmeibc_ib_nordda_channel_get_rx_iu(
    struct nvmeibc_ib_nordda_channel *ch,
    int index
);

int nvmeibc_ib_nordda_channel_put_rx_iu(
    struct nvmeibc_ib_nordda_channel *ch,
    struct nvmeib_iu *iu
);
```

Routes to SRQ or RQ based on configuration.

## VEX (Version Extension) Support

### VEX Operations

VEX allows protocol versioning and extension:

```c
const struct vex_ops *vex_nrio_ops[vex_nrch_max];
```

Registered operations:
- `vex_nrch_io_read`: Read I/O encoding/decoding
- `vex_nrch_io_other`: Write/discard/etc. encoding/decoding
- `vex_nrch_gen_uj_req`: GET_UUID_JOUR encoding
- `vex_nrch_gen_uj_rsp`: GET_UUID_JOUR decoding
- `vex_nrch_gen_br_req`: BLKSET_RECOVERED encoding
- `vex_nrch_gen_db_req`: GET_EC_DB encoding
- `vex_nrch_gen_fje`: FREE_JRNL_ENTS encoding
- `vex_nrch_gen_fje_ent`: Free entry encoding
- `vex_nrch_gen_je`: JENTRY_ERASE encoding
- `vex_nrch_io_srv_lock_req`: Lock request encoding
- `vex_nrch_io_srv_lock_rsp`: Lock response decoding

### VEX Encoding Flow

```c
rv = CALL_VEX_OP(encode,
    vex_nrch_io_read_clnt_ops, TWO_EXT,
        base, vex_nrch_io_read_clnt_base_encode,
        ext1, vex_nrch_io_read_clnt_ext1_encode,
        ext2, vex_nrch_io_read_clnt_ext2_encode,
        vex_ops, wire_buf, wire_buf_end, &cmd_ctx);
```

1. Encodes base fields (always present)
2. Encodes ext1 fields (if link version >= ext1)
3. Encodes ext2 fields (if link version >= ext2)
4. Returns total encoded size

### VEX Decoding Flow

```c
rv = CALL_VEX_OP(decode,
    vex_nrch_gen_uj_rsp_clnt_ops, TWO_EXT,
        base, vex_nrch_gen_uj_rsp_clnt_base_decode,
        ext1, vex_nrch_gen_uj_rsp_clnt_ext1_decode,
        ext2, vex_nrch_gen_uj_rsp_clnt_ext2_decode,
        vex_ops, wire_buf, wire_buf_end, gen_cmd);
```

1. Decodes base fields (always present)
2. Decodes ext1 fields (if present in response)
3. Decodes ext2 fields (if present in response)
4. Returns total decoded size

### VEX Registration

```c
static void register_nrio_vex_ops(struct nvmeibc_ib_net *net)
{
    struct nvmeibc_ib_nordda_channel *nrch = c_to_inrc(net->ioch);
    memcpy(nrch->net.vex_nrio_ops, net->admin_ch->vex_nrio_ops,
           sizeof(nrch->net.vex_nrio_ops));
}
```

Called after successful login, copies VEX operations from admin channel.

## Bypass Mechanisms

### RPC Locks (Server-Side)

When atomic operations are not available (e.g., TCP), lock operations use RPC bypass:

```c
int nvmeibc_disk_execute_lock(
    struct nvmeibc_disk *disk,
    struct nvmeibc_disk_lock_cmd *disk_lock_cmd,
    struct nvmeibc_channel *ch
);
```

Flow:
1. Convert lock operation to disk command
2. Execute via `execute_lock()` on nordda channel
3. Target performs lock operation server-side
4. Response via `process_lock_rsp()`

Supported operations:
- `NVMEIB_LOCK_CMP_AND_SWAP`
- `NVMEIB_LOCK_FORCE_WRITE`
- `NVMEIB_LOCK_READ`
- `NVMEIB_LOCK_RDMA_READ` (ext1+)
- `NVMEIB_LOCK_RDMA_WRITE` (ext1+)

### Piggyback Operations

Lock operations can be piggybacked on I/O commands:

#### Read with Lock Piggyback

```c
if (dp_cmds_pigbck_has_any(block_cmd)) {
    io_req->ind_pg_op = NVMEIB_IND_PG_READ_LOCK;
    io_req->lmi = lsi->lmi;
    io_req->offset = cpu_to_be64(offset);
}
```

Response includes lock value:
```c
if (dc->opr == NVMEIBC_LOCK_READ) {
    union nvmeib_lock_blkset_entry *lock_entry =
        (void *)&rsp->io_rsp.base.piggyback_read;
    nvmeibc_set_lock_read(bcmd, dc, rsp->io_rsp.base.piggyback_read,
                          lock_entry->lock_id.all, lock_entry->blkset_info.all);
}
```

#### Write with Lock Piggyback

```c
if (dp_cmds_pigbck_has_any(block_cmd)) {
    io_req->ind_pg_op = NVMEIB_IND_PG_WRITE_BLKSET_INFO;
    io_req->lmi = lsi->lmi;
    io_req->offset = cpu_to_be64(offset);
    io_req->blkset_info = cpu_to_be32(binfo);
}
```

Updates blockset info atomically with write.

## Latency Measurement

### Per-Request Measurement

```c
struct nvmeibc_nr_lat_meas_req {
    ktime_t sq_post_time;                 // SQ post time
    ktime_t send_comp_time;               // Send completion time
    ktime_t recv_comp_time;               // Recv completion time
    enum nvmeibc_disk_command_type cmd_type;
};
```

### Measurement Points

1. **SQ Post**:
   ```c
   nvmeibc_nr_lat_meas_init_req(&info->lat_meas, info->req.dcmd->cmd_type);
   nvmeibc_nr_lat_meas_sq_post(&info->lat_meas);
   ```

2. **Send Completion**:
   ```c
   nvmeibc_nr_lat_meas_send_comp(&req->lat_meas, wc);
   ```

3. **Recv Completion**:
   ```c
   nvmeibc_nr_lat_meas_recv_comp(&req->lat_meas, wc);
   nvmeibc_nr_lat_meas_recv_comp_process(&req->lat_meas);
   ```

4. **Update Per-CPU Stats**:
   ```c
   nvmeibc_nr_lat_meas_update_nrch_pcpu_data(
       ch->per_cpu_lat_data, &req->lat_meas,
       both_comps_arrived, is_siw);
   ```

### SIW Timestamping

For SIW (Software iWARP), TX timestamps are captured:

```c
#if ENABLE_SIW
nvmeib_send_wr_common(*wrt).send_flags |= SIW_IB_SEND_TX_TIMESTAMP;
#endif
```

Allows accurate measurement of network transit time.

## Debugging

### Trace Points

Key trace events:
- `trace_nvmeibc_post_io`: I/O command post
- `trace_ib_nordda_channel_nvmeibc_ib_nordda_channel_use`: Channel use
- `trace_ib_nordda_channel_send_completion_has_rsp`: Send completion
- `trace_ib_nordda_channel_process_rsp`: Receive response
- `trace_ib_nordda_channel_nordda_pending_io`: Pending I/O
- `trace_ib_nordda_channel_handle_watchdog_event`: Watchdog event

### Debug Parameters

```bash
# Skip RDMA write (UNSAFE, for debugging)
/sys/module/nvmesh_ib_client/parameters/nr_skip_rdma_write
Default: false

# Store FR descriptors for reuse workaround
/sys/module/nvmesh_ib_client/parameters/nr_store_fr
Default: true

# Fail UUID_JOUR encoding (testing)
/sys/module/nvmesh_ib_client/parameters/debug_fail_uj_encode
Default: false
```

### Request In-Use Bitmap

```c
DECLARE_BITMAP(req_in_use, NVMEIB_MAX_NORDDA_IO_REQ);

BUG_ON(test_and_set_bit(info->idx, info->nrch->req_in_use));
BUG_ON(!test_and_clear_bit(req->idx, req->nrch->req_in_use));
```

Tracks which requests are actively in use, catches double-use bugs.

## Configuration Parameters

### Channel Configuration

```bash
# Maximum requests per nordda channel
/sys/module/nvmesh_ib_client/parameters/nr_max_used_reqs_per_channel
Default: 64

# Use per-CPU CQ for nordda channels
/sys/module/nvmesh_ib_client/parameters/use_pcpu_cq
Default: false
```

### Completion Configuration

```bash
# Defer receive completions (IB)
/sys/module/nvmesh_ib_client/parameters/nr_defer_recv_comps
Default: false

# Defer receive completions (TCP)
/sys/module/nvmesh_ib_client/parameters/nr_defer_recv_comps_tcp
Default: false

# Use shared CQ (IB)
/sys/module/nvmesh_ib_client/parameters/nr_shared_cq
Default: false

# Use shared CQ (TCP)
/sys/module/nvmesh_ib_client/parameters/nr_shared_cq_tcp
Default: false

# Use SRQ (IB)
/sys/module/nvmesh_ib_client/parameters/nr_use_srq
Default: true

# Use SRQ (TCP)
/sys/module/nvmesh_ib_client/parameters/nr_use_srq_tcp
Default: true
```

### Watchdog Configuration

```bash
# Long timeout (considers last received)
/sys/module/nvmesh_ib_client/parameters/nr_wd_long_timeout
Default: 0 (uses NVMEIBC_IO_LONG_TIMEOUT)

# Rescue timeout (detect missing events)
/sys/module/nvmesh_ib_client/parameters/nr_wd_rescue_timeout
Default: 0 (disabled)
```

### Debug Configuration

```bash
# Skip RDMA write (UNSAFE)
/sys/module/nvmesh_ib_client/parameters/nr_skip_rdma_write
Default: false

# Store FR descriptors for reuse
/sys/module/nvmesh_ib_client/parameters/nr_store_fr
Default: true

# Fail UUID_JOUR encode (testing)
/sys/module/nvmesh_ib_client/parameters/debug_fail_uj_encode
Default: false
```

## Common Issues and Solutions

### Issue: LOC_PROT Errors on Send Completion

**Symptoms**: `IB_WC_LOC_PROT_ERR` on send completions

**Possible Causes**:
1. Reusing FR descriptor before previous send completes
2. Invalidating FR while still in use

**Solutions**:
- Ensure `nr_store_fr=true` (default)
- Wait for journal send completion before data write reuse
- Check `reused_bb_wait_send_comp` logic

### Issue: Double Completions

**Symptoms**: BUG() in completion handler, "Both send and recv already arrived"

**Possible Causes**:
1. Version wraparound bug
2. Request freed prematurely
3. Tag encoding/decoding mismatch

**Solutions**:
- Verify version increments correctly
- Check `NVMEIB_TAG_VERSION_RESERVED` handling
- Verify tag encoding matches decoding

### Issue: Request Starvation

**Symptoms**: No free requests available, operations queued

**Possible Causes**:
1. `nr_max_used_reqs_per_channel` too low
2. Requests leaked (not freed properly)
3. Long-running operations

**Solutions**:
- Increase `nr_max_used_reqs_per_channel`
- Check for request leaks (in-use bitmap)
- Review watchdog timeouts

### Issue: Watchdog Timeouts

**Symptoms**: Operations timeout, channels disconnect

**Possible Causes**:
1. Network issues
2. Target overload
3. Missing completions

**Solutions**:
- Check network connectivity
- Verify target health
- Enable `nr_wd_rescue_timeout` to detect missed events
- Increase `nr_wd_long_timeout` if needed

### Issue: Reuse Logic Failures

**Symptoms**: Errors after journal write, data write fails

**Possible Causes**:
1. Send completion arrives before recv
2. FR descriptors freed prematurely
3. SG list not restored correctly

**Solutions**:
- Verify `reused_bb_wait_send_comp` logic
- Check `REUSE_SG_FR_STORE/RESTORE` macros
- Ensure `wait_release_zero_before_cb` is set for IOMMU

### Issue: High Latency

**Symptoms**: Slow I/O performance

**Possible Causes**:
1. Deferred completions enabled
2. Single channel bottleneck
3. Completion offload overhead

**Solutions**:
- Disable `nr_defer_recv_comps` for lower latency
- Use per-CPU channels (`use_pcpu_cq=true`)
- Reduce completion offload

### Issue: Per-CPU Channel Wrong CPU

**Symptoms**: "wrong CPU" warnings, operations fail

**Possible Causes**:
1. Operation submitted from wrong CPU
2. IRQs not disabled
3. Preemption not disabled

**Solutions**:
- Ensure operations submitted from correct CPU
- Disable IRQs during submission
- Use `pcpu_pending_io_smp_call` for cross-CPU submission

## Performance Optimization

### Per-CPU Lockless Channels

Enable per-CPU channels for maximum performance:

```bash
echo 1 > /sys/module/nvmesh_ib_client/parameters/use_pcpu_cq
```

Benefits:
- No spinlock contention
- Cache-line exclusive to CPU
- Lower latency
- Higher throughput

Requirements:
- IRQs disabled during operation
- Operation on correct CPU

### Request Reuse

Journal write → data write reuse reduces overhead:

```
Without reuse:
  - Map data (DMA map, FR)
  - RDMA_WRITE journal
  - Unmap data
  - Map data again (DMA map, FR)
  - RDMA_WRITE data
  - Unmap data

With reuse:
  - Map data (DMA map, FR)
  - RDMA_WRITE journal
  - Keep mapping
  - RDMA_WRITE data (reuse mapping)
  - Unmap data
```

Saves ~50% of mapping overhead for journal writes.

### Deferred Completions

Optimize for latency vs. flexibility:

```bash
# Lower latency (inline completions)
echo 0 > /sys/module/nvmesh_ib_client/parameters/nr_defer_recv_comps

# More flexibility (deferred completions)
echo 1 > /sys/module/nvmesh_ib_client/parameters/nr_defer_recv_comps
```

### Shared CQ

Reduce resource usage:

```bash
echo 1 > /sys/module/nvmesh_ib_client/parameters/nr_shared_cq
```

Benefits:
- Fewer CQs
- Lower memory usage
- Better cache utilization

Trade-offs:
- Higher CQ contention
- More polling overhead

## See Also

- **LOCKS_CHANNEL.md**: Lock channel documentation
- **RDMA_COMPLETION_MODES.md**: RDMA completion handling
- **Admin Channel documentation**: Channel management

## Implementation Files

- `clnt/nvmeibc_ib_nordda_channel.c`: Main nordda channel implementation (~2600 lines)
- `clnt/nvmeibc_ib_nordda_channel.h`: Nordda channel header
- `clnt/nvmeibc_ib_net_nordda.c`: Nordda network implementation (~2000 lines)
- `clnt/nvmeibc_ib_net_nordda.h`: Nordda network header
- `common/nvmeib_completion_noise.c`: Completion noise measurement

