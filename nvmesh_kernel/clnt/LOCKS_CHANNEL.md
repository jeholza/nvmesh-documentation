# Locks Channel Documentation

## Overview

The Locks Channel is a specialized RDMA-based communication channel used for distributed lock management in NVMesh. It provides high-performance, low-latency lock operations using RDMA atomic operations or RPC-based fallback mechanisms when hardware atomic support is unavailable.

## Architecture

### Channel Hierarchy

The Locks Channel consists of:

1. **Primary Lock Channel** - Main channel for lock operations
2. **Secondary Lock Channels** - Additional channels for scaling and performance
   - Per-CPU channels (lockless or locked)
   - Coremask-based channels for CPU affinity
   - Standard secondary channels with LRU/sharding selection

### Key Components

```
┌─────────────────────────────────────────────────────────────┐
│                    Locks Channel System                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐         ┌──────────────────────────────┐  │
│  │   Primary    │────────>│   Secondary Channels         │  │
│  │   Channel    │         │   - Per-CPU (lockless)       │  │
│  │              │         │   - Coremask-based           │  │
│  │ - Login/Auth │         │   - Standard (LRU/sharding)  │  │
│  │ - Keep-Alive │         └──────────────────────────────┘  │
│  │ - Watchdog   │                                            │
│  └──────────────┘                                            │
│         │                                                     │
│         ▼                                                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │           RDMA Operations                             │   │
│  │  - Atomic Compare-and-Swap (CAS)                     │   │
│  │  - Masked Atomic CAS (for locks w/ blkset info)     │   │
│  │  - RDMA Read/Write                                   │   │
│  │  - RPC Locks (fallback)                              │   │
│  └──────────────────────────────────────────────────────┘   │
│         │                                                     │
│         ▼                                                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │           Completion & Callback                       │   │
│  │  - Send completion handling                          │   │
│  │  - Deferred operation processing                     │   │
│  │  - Lock status updates                               │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## Data Structures

### struct nvmeibc_locks_channel

The main locks channel structure:

```c
struct nvmeibc_locks_channel {
    spinlock_t locks_spinlock;           // Channel lock
    struct nvmeibc_channel base;          // Base channel
    struct nvmeibc_ib_net net;            // IB network connection
    
    // Operation management
    struct list_head defered;             // Deferred operations
    struct list_head in_progress;         // In-progress operations
    struct list_head free_ip_pool;        // Free operation pool
    struct list_head aborted;             // Aborted operations
    
    // Operation buffers
    struct nvmeibc_lock_opr_in_progress *locks_ip_buffer;
    
    // Statistics
    int num_of_free;                      // Free operations
    int num_in_progress;                  // In-progress operations
    int num_defered;                      // Deferred operations
    int num_of_atom_read_ip;              // Atomic reads in progress
    int max_atom_read_ip;                 // Max atomic operations
    
    // Atomic capabilities
    enum ib_atomic_cap atomic_cap;        // Local atomic support
    enum ib_atomic_cap masked_atomic_cap; // Masked atomic support
    enum ib_atomic_cap tgt_atomic_cap;    // Target atomic support
    
    // Endianness handling
    bool atomic_req_endian_swap;          // Request endian swap needed
    bool atomic_reply_endian_swap;        // Reply endian swap needed
    
    // Secondary channels
    struct nvmeibc_locks_channel *_2nd_ch[NVMEIB_N_2ND_LOCK_CHS];
    int n_2nd_ch;                         // Number of secondary channels
    struct nvmeibc_locks_channel *primary_ch;  // Back pointer to primary
    
    // Per-CPU channel support
    struct nvmeibc_locks_channel *_2nd_ch_pcpu_map[NVMEIB_DFLT_MAX_CPUS];
    cpumask_var_t _2nd_ch_pcpu_mask;      // CPUs with per-CPU channels
    
    // Coremask channel support
    struct nvmeibc_locks_channel *_2nd_ch_coremask_map[NVMEIB_DFLT_MAX_CPUS];
    struct nvmeib_cpu_mask _2nd_ch_coremask_mask;
    
    // Channel selection method
    enum nvmeibc_lock_channel_choosing_method method;
    
    // Local bypass (for loopback)
    DECLARE_BITMAP(local_bypass_bmp, NUM_NVMEIBC_LOCK_OPR);
    
    // Keep-alive
    struct admin_periodic periodic_ka;
    u64 ka_post_cnt;                      // KA posted count
    u64 ka_comp_jif;                      // KA completion jiffies
    
    // Callback work queue
    struct workqueue_struct *callback_wq;
};
```

### struct nvmeibc_lock_opr_in_progress

Represents an in-flight lock operation:

```c
struct nvmeibc_lock_opr_in_progress {
    struct nvmeibc_disk_lock_cmd disk_lock_cmd;  // Disk command (for bypass)
    bool disk_piggyb;                     // Piggyback on NRDDA cmd
    struct nvmeibc_locks_channel *ch;     // Parent channel
    int index;                            // Index in buffer
    
    // RDMA data
    u64 val[NVMEIB_LOCK_DATA_BUFFERS];   // Value buffer
    dma_addr_t val_phys;                  // Physical address
    struct ib_sge list;                   // Scatter-gather entry
    struct nvmeib_send_wr wr;             // RDMA work request
    
    // Linked list management
    struct list_head link;                // Link in list
    struct nvmeibc_d_rdma_comp *comp;     // Completion descriptor
    
    // Watchdog
    struct wd_info_common wdc;            // Watchdog context
    unsigned long start_time;             // Start time
    unsigned long opr_timeout;            // Operation timeout
    
    // State tracking
    bool aborted;                         // Was aborted
    u16 version;                          // Version for double-comp check
    struct nvmeib_state_guard state;      // Operation state
    struct nvmeib_state_guard bypass_state;  // Bypass state
};
```

### struct nvmeibc_d_rdma_comp

Lock operation completion descriptor:

```c
struct nvmeibc_d_rdma_comp {
    enum nvmeibc_disk_locks_opr opr;      // Operation type
    u64 lockset_id;                       // Lock ID
    u64 compare;                          // CAS compare value
    u64 exchange;                         // CAS exchange value
    u64 val[2];                           // Result values
    
    union nvmeib_lock lock;               // Lock value
    const struct nvmeib_lock_constants *lock_cnsts;  // Lock constants
    
    enum nvmeibc_lock_completion_status lock_status;  // Completion status
    
    // Callback
    void (*callback)(struct nvmeibc_d_rdma_comp *, u64);
    
    // Memory info
    struct nvmeibc_disk_seg_locks_mem_info *mem_info;
    
    // Deferred operation management
    struct list_head link_deferred;       // Deferred list link
    unsigned long deferred_jif;           // Deferred jiffies
    
    // CPU affinity
    struct nvmeib_cpu_mask_info cpu_mask_info;
    
    // Statistics
    struct nvmeibc_disk_command_probes probes;
    int n_retries_cmpxcng;                // CAS retry count
    
    // Non-masked CAS with blkset info
    struct {
        u32 last_value;                   // Last blkset_info value
        u8 n_attempts;                    // Number of attempts
    } nmcs_bi;
};
```

## Lock Operations

### Operation Types

```c
enum nvmeibc_disk_locks_opr {
    NVMEIBC_LOCK_CMP_AND_SWAP,           // Atomic compare-and-swap
    NVMEIBC_LOCK_FORCE_WRITE,            // Force write lock value
    NVMEIBC_LOCK_READ,                   // Read lock value
    NVMEIBC_LOCK_BLKSET_INFO_WRITE,     // Write blockset info
    NVMEIBC_LOCK_BLKSET_INFO_READ,      // Read blockset info
};
```

### Lock Acquisition Flow

```
1. Client calls nvmeibc_disk_locks_interlocked_cmp_exchange()
   │
   ├─> Validate handle
   │
   ├─> Choose lock channel (primary or secondary)
   │   │
   │   ├─> Check if already locked by this CPU (prevent deadlock)
   │   ├─> For per-CPU channels: use channel for current CPU
   │   ├─> For coremask channels: lookup channel for CPU/coremask
   │   └─> For standard channels: LRU / CPU-based / Sharding
   │
   ├─> Add to deferred queue
   │
   ├─> lock_prepare_and_send()
   │   │
   │   ├─> Check free operations available
   │   ├─> Check atomic operation limit
   │   │
   │   ├─> For each deferred operation:
   │   │   ├─> Get free operation from pool
   │   │   ├─> Prepare RDMA work request
   │   │   │   │
   │   │   │   ├─> CAS: Setup atomic compare-and-swap
   │   │   │   │   ├─> Regular: IB_WR_ATOMIC_CMP_AND_SWP
   │   │   │   │   └─> With blkset_info: IB_WR_MASKED_ATOMIC_CMP_AND_SWP
   │   │   │   │
   │   │   │   ├─> READ: Setup RDMA read (IB_WR_RDMA_READ)
   │   │   │   └─> WRITE: Setup RDMA write (IB_WR_RDMA_WRITE)
   │   │   │
   │   │   └─> Link work requests for batching
   │   │
   │   ├─> Decide on bypass (local/RPC):
   │   │   ├─> Local bypass: execute_opr_local_bypass()
   │   │   │   └─> Call local server gen_cmd directly
   │   │   │
   │   │   └─> RPC bypass: nvmeibc_disk_execute_lock()
   │   │       └─> Piggyback on NRDDA disk command
   │   │
   │   └─> Post send (ib_post_send or post_send_atomic_fn)
   │
   └─> Start watchdog timer

2. RDMA completion arrives
   │
   ├─> lock_send_completion()
   │   │
   │   └─> nvmeibc_disk_locks_on_completion()
   │       │
   │       ├─> Validate version (prevent double completion)
   │       │
   │       ├─> disk_locks_on_completion()
   │       │   │
   │       │   ├─> Handle endianness swap if needed
   │       │   │
   │       │   ├─> Extract lock value
   │       │   │   ├─> Regular lock: val[0] = lock_id
   │       │   │   └─> Lock with blkset_info:
   │       │   │       ├─> val[0] = lock_id
   │       │   │       └─> val[1] = blkset_info
   │       │   │
   │       │   ├─> Handle non-masked CAS retry (if needed)
   │       │   │   └─> Resubmit with updated blkset_info
   │       │   │
   │       │   └─> Update completion status:
   │       │       ├─> CAS: TAKEN if compare==val[0], else CONTENDED
   │       │       ├─> READ: CONTENDED if val[0]!=0, else TAKEN
   │       │       └─> WRITE: TAKEN
   │       │
   │       ├─> Free operation back to pool
   │       │
   │       ├─> Process more deferred operations
   │       │
   │       └─> Execute callback
   │           ├─> Inline (if on correct CPU)
   │           ├─> On work queue
   │           └─> On system per-CPU work queue
   │
   └─> Client callback invoked with lock_status
```

### Lock Operations API

#### Initialize Lock Handle

```c
void *nvmeibc_disk_locks_seg_locks_mem_info(
    struct nvmeibc_disk *disk,
    int seg_id
);
```

Returns a handle for lock operations on a segment. Must be freed with `nvmeibc_disk_locks_free_mem_info()`.

#### Compare-and-Swap (Lock/Unlock)

```c
int nvmeibc_disk_locks_interlocked_cmp_exchange(
    void *handle,                         // Lock handle
    u64 addr,                             // Lock address
    struct nvmeibc_d_rdma_comp *comp      // Completion descriptor
);
```

Performs atomic compare-and-swap. Used for:
- **Lock acquisition**: compare=0, exchange=client_id
- **Lock release**: compare=client_id, exchange=0

**Returns**: 0 on success, negative error code on failure

**Completion statuses**:
- `NCL_STATUS_TAKEN`: Lock acquired/released successfully
- `NCL_STATUS_CONTENDED`: Lock held by another client
- `NCL_STATUS_FAIL_COMP`: Operation failed
- `NCL_STATUS_FAIL_NO_COMP`: No completion received

#### Read Lock

```c
int nvmeibc_disk_locks_read_lock(
    void *handle,
    u64 addr,
    struct nvmeibc_d_rdma_comp *comp
);
```

Reads current lock value without modification.

#### Write Blockset Info

```c
int nvmeibc_disk_locks_write_blkset_info(
    void *handle,
    u64 addr,
    struct nvmeibc_d_rdma_comp *comp
);
```

Writes blockset metadata to lock (for locks with `w_blkset_info` enabled).

#### Read Blockset Info

```c
int nvmeibc_disk_locks_read_blkset_info(
    void *handle,
    u64 addr,
    struct nvmeibc_d_rdma_comp *comp
);
```

Reads blockset metadata from lock.

#### Free Lock Handle

```c
void nvmeibc_disk_locks_free_mem_info(void *handle);
```

Frees lock handle and associated resources.

## Channel Management

### Connection

```c
struct nvmeibc_locks_channel *nvmeibc_locks_channel_connect_locks(
    struct nvmeibc_admin_channel *admin_ch,
    union ib_gid *gid,
    int max_tgt_atomic_ops,
    enum rdma_link_layer dest_link_layer,
    enum rdma_transport_type dest_transport_type,
    int rgid_idx,
    unsigned tcp_base_port,
    unsigned tcp_num_ports
);
```

Connects primary lock channel and creates secondary channels based on configuration.

**Secondary Channel Creation**:
1. Determines max secondary channels:
   - For TCP/RoCE: `nvmeibc_max_lock_channels_tcp` (default: varies)
   - For InfiniBand: `nvmeibc_max_lock_channels` (default: 5)
   - Limited by target CPU count

2. Creates per-CPU channels (if `lock_ch_2nd_ch_pcpu` > 0):
   - Lockless channels for each CPU
   - Direct submission without locking
   - Requires IRQs disabled for correctness

3. Creates coremask channels (if `lock_ch_2nd_ch_coremask` > 0):
   - Channels for specific CPU masks
   - Used for coremask-aware volumes

4. Creates standard secondary channels:
   - Selected by LRU, CPU-based, or sharding
   - Shared across CPUs with locking

### Atomic Capability Testing

After connection, the channel tests atomic capabilities:

```
test_atomics() for both regular and masked atomics:
│
├─> RDMA_WRITE test value
├─> ATOMIC_CMP_AND_SWAP
├─> RDMA_READ result
└─> Verify and detect endianness requirements
```

**Atomic Operation Support**:
- `IB_ATOMIC_NONE`: No atomic support → Use RPC locks
- `IB_ATOMIC_HCA`: Atomic support in HCA
- `IB_ATOMIC_GLOB`: Global atomic support

**Masked Atomic Support**:
- Required for locks with blockset info
- Falls back to retry logic if unsupported
- Retries up to `NMCS_BI_MAX_ATTEMPTS` (8)

### Keep-Alive

Primary channel sends periodic keep-alive operations:

```c
static int lock_ch_periodic_ka_fn(void *arg, unsigned long t);
```

- Posts RDMA_WRITE to test zone
- Detects channel timeouts
- Frequency controlled by `admin_periodic` mechanism

### Disconnection

```c
void nvmeibc_locks_channel_disconnect(
    struct nvmeibc_locks_channel *ch
);
```

Disconnection sequence:
1. Pause channel (no new operations)
2. Drain deferred operations
3. Abort in-progress operations with callbacks
4. Disconnect all secondary channels
5. Break QP and free resources
6. Trigger disk release if needed

## Channel Selection Methods

### 1. LRU with Sharding Tiebreak (Default)

```c
LOCK_CHANNEL_CHOOSING_METHOD_LRU
```

Selects channel with minimum total operations (in-progress + deferred), using address-based sharding for tiebreak.

### 2. By-CPU

```c
LOCK_CHANNEL_CHOOSING_METHOD_BY_CPU
```

Selects channel based on `cpu_id % num_channels`.

### 3. Sharding by Address

```c
LOCK_CHANNEL_CHOOSING_METHOD_SHARDING
```

Selects channel based on lock address: `(addr % num_channels)`.

### 4. Per-CPU Lockless (Configurable)

Dedicated channel per CPU, no spinlock needed. Requires:
- IRQs disabled during operation
- Preemption disabled
- Operation submitted from channel's CPU

Configuration:
```bash
# Enable with 1 channel per CPU
echo 1 > /sys/module/nvmesh_ib_client/parameters/lock_ch_2nd_ch_pcpu

# Enable for specific CPUs only
echo "0xff,0xff" > /sys/module/nvmesh_ib_client/parameters/lock_ch_pcpu_cpus
```

### 5. Coremask-based

For coremask-aware volumes, channels are associated with specific CPU masks:

```c
int nvmeibc_locks_channel_connect_coremask_chs(
    struct nvmeibc_locks_channel *ch,
    struct nvmeibc_admin_channel *admin_ch,
    u64 coremask_uid,
    const struct nvmeib_cpu_mask *cpumask,
    void *coremask_cookie,
    const struct nvmeibc_locks_channel_coremask_ops *coremask_ops,
    struct nvmeib_cpu_mask *lch_cpumask
);
```

## Bypass Mechanisms

### Local Bypass

For loopback connections (client and target on same host):

```c
static int execute_opr_local_bypass(
    struct nvmeibc_locks_channel *locks_channel,
    struct nvmeibc_d_rdma_comp *comp,
    struct nvmeibc_d_rdma_comp **bad_comp
);
```

- Calls `nvmeibs_handle_gen_cmd()` directly
- Avoids RDMA round-trip
- Only for operations in `local_bypass_bmp`
- Can use PCIe atomics for CAS operations

**Local Bypass Operations**:
- `NVMEIBC_LOCK_READ`
- `NVMEIBC_LOCK_BLKSET_INFO_WRITE`
- `NVMEIBC_LOCK_BLKSET_INFO_READ`
- `NVMEIBC_LOCK_CMP_AND_SWAP` (if PCIe atomics supported)

### RPC Locks (Server-side)

When atomic operations are not supported:

```c
int nvmeibc_disk_locks_server_side_post_send_atomic(
    struct ib_qp *qp,
    struct nvmeib_send_wr *send_wr,
    struct nvmeib_send_wr **bad_send_wr
);
```

- Converts atomic WRs to disk commands
- Piggybacks on NRDDA (Non-RDMA) disk operations
- Target performs atomic operations server-side
- Returns via disk command completion

**RPC Lock Flow**:
```
Client                    Target
  │                         │
  ├─> nvmeibc_disk_execute_lock()
  │   (converts atomic to disk cmd)
  │                         │
  ├────── Disk Command ────>│
  │                         │
  │                    Execute Lock
  │                    (atomic on server)
  │                         │
  │<──── Completion ────────┤
  │                         │
  └─> nvmeibc_locks_channel_lock_cmd_completion()
```

## Error Handling

### Watchdog

Each lock operation has a watchdog timer:

```c
static void init_lock_watchdog(
    struct nvmeibc_locks_channel *chl,
    struct nvmeibc_lock_opr_in_progress *opr
);
```

- Timeout: `net->wd_timeout_jif`
- On timeout: triggers `locks_handle_qp_error()`
- Disconnects channel and fails all operations
- Triggers disk release

### Operation Abort

```c
void nvmeibc_disk_locks_abort_all_oprs(
    struct nvmeibc_locks_channel *ch
);
```

Abort sequence:
1. Pause all channels (primary + secondary)
2. Drain deferred operations with failure callbacks
3. Abort in-progress operations
4. Call completion callbacks with `NCL_STATUS_FAIL_COMP`

### Retry Logic

**Non-masked CAS with Blockset Info**:
When target doesn't support masked atomics:
1. First attempt with guessed blkset_info
2. On failure, extract actual blkset_info from returned value
3. Retry with correct blkset_info
4. Repeat up to `NMCS_BI_MAX_ATTEMPTS` (8 times)

**Lock Contention**:
- Client retries on `NCL_STATUS_CONTENDED`
- Warning after 512 retries
- Triggers `nvmeibc_disk_report_long_contended_lock()`

## Performance Optimization

### Batching

Operations are batched for efficiency:

```c
#define NVMEIBC_CHANNEL_AV_NUM_OF_WR_PER_MSG 8
```

- Posts multiple work requests in one `ib_post_send()`
- Reduces context switches
- Improves PCIe utilization

### Fast Reuse

```c
module_param_named(disk_locks_fast_reuse, ...);
```

When enabled, reuses freed operations immediately in completion handler, reducing latency.

### Lockless Per-CPU Channels

```c
module_param_named(lock_ch_2nd_ch_pcpu_lockless, ...);
```

Eliminates spinlock contention:
- Each CPU has dedicated channel
- No locking needed (preemption disabled)
- Significant latency reduction
- Requires IRQs disabled

### Completion Offload

Callbacks can be offloaded to work queues:

```c
module_param_named(lock_ch_scq_offload_thread, ...);
```

- Reduces interrupt handler time
- Allows blocking operations in callbacks
- Can use system per-CPU work queues

### Atomic Operation Pipelining

Limits concurrent atomic operations:

```c
ch->max_atom_read_ip = ch->net.max_dest_rd_atomic * 2;
```

- Prevents overwhelming target
- Based on QP `max_dest_rd_atomic` capability
- Defers excess operations

## Configuration Parameters

### Channel Configuration

```bash
# Maximum secondary lock channels (IB)
/sys/module/nvmesh_ib_client/parameters/max_lock_channels
Default: 5

# Maximum secondary lock channels (TCP)
/sys/module/nvmesh_ib_client/parameters/max_lock_channels_tcp
Default: varies

# Channel selection method
/sys/module/nvmesh_ib_client/parameters/lock_ch_get_method
# 0=LRU, 1=BY_CPU, 2=SHARDING
Default: 0 (LRU)

# TCP channel selection method
/sys/module/nvmesh_ib_client/parameters/lock_ch_get_method_tcp
Default: 1 (BY_CPU)
```

### Per-CPU Channel Configuration

```bash
# Enable per-CPU secondary channels
/sys/module/nvmesh_ib_client/parameters/lock_ch_2nd_ch_pcpu
# 0=Disabled, 1=Max CPUs, N=Specific count
Default: 0

# Make per-CPU channels lockless
/sys/module/nvmesh_ib_client/parameters/lock_ch_2nd_ch_pcpu_lockless
Default: true

# CPUs for per-CPU channels (hex mask)
/sys/module/nvmesh_ib_client/parameters/lock_ch_pcpu_cpus
# Example: "ff,ff" for CPUs 0-15
Default: NULL (all CPUs)
```

### Coremask Channel Configuration

```bash
# Number of channels per coremask
/sys/module/nvmesh_ib_client/parameters/lock_ch_2nd_ch_coremask
# 0=Disabled, N=Max channels per mask
Default: 0
```

### Completion Configuration

```bash
# Use thread for send completion offload
/sys/module/nvmesh_ib_client/parameters/lock_ch_scq_offload_thread
Default: true

# Use thread for send completion offload (TCP)
/sys/module/nvmesh_ib_client/parameters/lock_ch_scq_offload_thread_tcp
Default: true

# Fast reuse of freed operations
/sys/module/nvmesh_ib_client/parameters/disk_locks_fast_reuse
Default: false

# Use system per-CPU work queue for callbacks
/sys/module/nvmesh_ib_client/parameters/disk_locks_use_system_pcpu_wq
Default: false
```

### Debug/Test Configuration

```bash
# Skip lock commands (UNSAFE - for testing only)
/sys/module/nvmesh_ib_client/parameters/skip_lock_cmds_flags
# Bit 0: CAS, Bit 1: Active Lock, Bit 2: Read Lock, Bit 3: Write Block-Info
Default: 0
```

## Lock Value Format

### Standard Lock (64-bit)

```
┌────────────────────────────────────────────────────────────┐
│                      Lock ID (64 bits)                      │
│                                                              │
│  0 = Unlocked                                               │
│  Non-zero = Client ID holding the lock                     │
└────────────────────────────────────────────────────────────┘
```

### Lock with Blockset Info (64-bit)

```
┌──────────────────────────┬──────────────────────────────────┐
│    Lock ID (32 bits)     │   Blockset Info (32 bits)       │
│                          │                                  │
│  0 = Unlocked            │  Transaction ID, dirty flags     │
│  Client ID otherwise     │  etc.                            │
└──────────────────────────┴──────────────────────────────────┘
```

When using locks with blockset info:
- `comp->lock.id` = Lock ID (32-bit)
- `comp->lock.bi` = Blockset Info (32-bit)
- Requires masked atomic operations or retry logic

## Statistics and Monitoring

### Per-Channel Statistics

```c
ch->n_comp_llp_opr;      // Lock operation completions
ch->n_comp_llp_ka;       // Keep-alive completions
ch->n_comp_llp_test;     // Atomic test completions
ch->num_in_progress;     // Operations in progress
ch->num_defered;         // Deferred operations
ch->num_of_free;         // Free operation slots
```

### Lock Contention

```c
comp->n_retries_cmpxcng; // CAS retry count
```

Warning issued after 512 contended retries.

### Keep-Alive Monitoring

```c
ch->ka_post_cnt;         // Keep-alives posted
ch->ka_comp_jif;         // Last KA completion time
ch->ka_skip_use;         // Times skipped due to timeout
```

### Probes (if enabled)

```c
struct nvmeibc_disk_command_probes probes;
```

Tracks:
- `ulp_post_jif`: Upper layer post time
- `llp_post_jif`: Lower layer post time
- `llp_comp_jif`: Lower layer completion time
- `ulp_comp_jif`: Upper layer completion time

## Debugging

### Trace Points

Key trace events:
- `trace_locks_channel_on_login_lock_ch`: Channel login
- `trace_disk_locks_lock_prepare_and_send`: Operation submission
- `trace_disk_locks_nvmeibc_disk_locks_on_completion`: Completion
- `trace_locks_channel_choose_locks_channel`: Channel selection
- `trace_disk_locks_nvmeibc_disk_locks_interlocked_cmp_exchange`: CAS operation

### State Guards

When `NVMEIBC_LOCKS_CHANNEL_GUARD_STATE` is enabled:

```c
enum lock_opr_state {
    LOCK_OPR_IN_FREE_LIST,
    LOCK_OPR_PREPARING,
    LOCK_OPR_IN_PROGRESS,
    LOCK_OPR_ABORTED,
    LOCK_OPR_ERROR,
};

enum lock_opr_rdma_bypass_state {
    LOCK_OPR_NO_BYPASS,
    LOCK_OPR_USING_BYPASS,
    LOCK_OPR_BYPASS_COMPLETE,
    LOCK_OPR_BYPASS_ERROR,
};
```

Validates state transitions and detects invalid states.

## Common Issues and Solutions

### Issue: Lock Timeouts

**Symptoms**: Operations timeout, channel disconnects

**Possible Causes**:
1. Network connectivity issues
2. Target overload
3. QP errors

**Solutions**:
- Check network stability
- Verify target responsiveness
- Increase `wd_timeout_jif` if needed
- Check for QP errors in logs

### Issue: High Lock Contention

**Symptoms**: Many retries, "Contended after 512 retries" warnings

**Possible Causes**:
1. Many clients competing for same lock
2. Long lock hold times
3. Poor workload distribution

**Solutions**:
- Review lock granularity
- Reduce lock hold times
- Distribute workload across locksets
- Use finer-grained locking

### Issue: Low Performance

**Symptoms**: High latency, low throughput

**Possible Causes**:
1. Single channel bottleneck
2. Spinlock contention
3. Callback overhead

**Solutions**:
- Enable more secondary channels
- Use per-CPU lockless channels
- Enable completion offload
- Use fast reuse (`disk_locks_fast_reuse`)

### Issue: Non-Masked CAS Retries

**Symptoms**: Many `CMPSWAP: fail cs, store bi and resubmit` traces

**Possible Causes**:
1. Target doesn't support masked atomics
2. Incorrect initial blkset_info guess

**Solutions**:
- Update target firmware (if available)
- This is normal fallback behavior
- Monitor retry counts don't exceed limits

### Issue: Channel Keep-Alive Failures

**Symptoms**: `No KA comp` warnings, channels marked as timeout

**Possible Causes**:
1. Network issues
2. Target hung
3. QP errors

**Solutions**:
- Check network connectivity
- Verify target health
- Check for earlier QP errors

## See Also

- **RDMA_COMPLETION_MODES.md**: RDMA completion handling details
- **Admin Channel documentation**: Channel management and lifecycle
- **NRDDA Channel documentation**: Non-RDMA disk access bypass mechanism

## Implementation Files

- `clnt/nvmeibc_locks_channel.c`: Main locks channel implementation
- `clnt/nvmeibc_locks_channel.h`: Locks channel header
- `clnt/nvmeibc_disk_locks.c`: Lock operation implementations
- `clnt/nvmeibc_disk_locks.h`: Lock operation header

