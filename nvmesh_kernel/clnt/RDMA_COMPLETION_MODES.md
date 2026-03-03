# RDMA Completion Processing Modes in nvmeibc_ib_net Layer

## Overview

The client (`clnt`) nvmeibc_ib_net layer supports multiple modes for processing incoming RDMA completions from Completion Queues (CQs). The mode selection is controlled by module parameters and connection-specific parameters, providing flexibility for different deployment scenarios and performance requirements.

## Mode Selection

The primary mode selection is controlled by the `nvmeibc_use_pcpu_cq` module parameter:

- **`nvmeibc_use_pcpu_cq=true` (default)**: Uses per-CPU shared CQs with interrupt+polling framework
- **`nvmeibc_use_pcpu_cq=false`**: Uses private CQs per connection with various processing strategies

### Module Parameter
```c
// Location: clnt/core/nvmeibc_core_ibdev.inc.c
bool nvmeibc_use_pcpu_cq = true; /* Disabled by service script for TCP */
module_param_named(use_pcpu_cq, nvmeibc_use_pcpu_cq, bool, 0444);
MODULE_PARM_DESC(use_pcpu_cq, "Use a per CPU shared completion queue (SCQ) and shared receive queue (SRQ)");
```

---

## Mode 1: Per-CPU Shared CQ with Interrupt+Polling Framework (dev_cq)

**Active when:** `nvmeibc_use_pcpu_cq=true`

**Location:** `common/nvmeib.c` (`nvmeib_dev_cq` structures)

### Architecture

This mode creates a completion queue per CPU, managed by a sophisticated interrupt and polling thread framework.

#### Key Components

1. **Per-CPU CQ Creation**
   - One CQ per online CPU (or up to `nvmeib_pcpu_cq_max_cqs_per_dev`)
   - Located in: `common/nvmeib.c::init_cqs()`
   - Structure: `struct nvmeib_dev_cq` (line 391-457)

2. **Interrupt+Polling Framework**
   - Generic framework in: `common_public/poll/nvmeib_public_intr_poll.c`
   - Per-CPU polling threads called "ipollers"
   - Structure: `struct intr_poller` (line 11-28)

3. **Poll Modes**
   ```c
   enum nvmeib_dev_cq_poll_mode {
       NVMEIB_DEV_CQ_POLL_DISABLED,
       NVMEIB_DEV_CQ_INTR_MODE,         // Interrupt-driven
       NVMEIB_DEV_CQ_POLL_MODE,         // Polling mode
       NVMEIB_DEV_CQ_USER_POLL_MODE,    // User-space polling (SPDK)
       NVMEIB_DEV_CQ_ENTER_USER_POLL_MODE,
       NVMEIB_DEV_CQ_EXIT_USER_POLL_MODE,
   };
   ```

### Operation Flow

1. **Interrupt Arrives**
   - CQ completion interrupt handler: `cq_completion_intr()` (common/nvmeib.c:1583)
   - Interrupt shaper decides whether to wake up polling thread based on load

2. **Transition to Polling**
   - If interrupt rate is high, transitions from `NVMEIB_DEV_CQ_INTR_MODE` to `NVMEIB_DEV_CQ_POLL_MODE`
   - Schedules work to the appropriate CPU's ipoller thread: `__poll_sched(&cqw->iop, cqw->cpu_id_sched)`

3. **Polling Thread (ipoller)**
   - Continuously polls CQ while work is available
   - Uses `ib_poll_cq()` to retrieve completions in batches
   - Processes completions via `cqw->process()` callback

4. **Return to Interrupt Mode**
   - When CQ becomes empty, re-arms interrupt with `ib_req_notify_cq()`
   - Transitions back to `NVMEIB_DEV_CQ_INTR_MODE`

### Advantages

- **Load Balancing**: Completions distributed across CPUs
- **Cache Affinity**: Processing stays on same CPU as application
- **Interrupt Coalescing**: Reduces interrupt overhead under high load
- **Shared Resources**: Fewer CQs and SRQs reduce memory footprint
- **SPDK Support**: Supports user-space polling via `NVMEIB_DEV_CQ_USER_POLL_MODE`

### Configuration

- Enabled by default
- Number of CQs: `nvmeib_pcpu_cq_max_cqs_per_dev` (default: num_comp_vectors)
- CQ size: `nvmeib_pcpu_cq_size`
- Poll duration: `nvmeib_public_ipoller_poll_duration_jif`
- Requires `use_srq=true` (Shared Receive Queues)

### Connection Setup

When `nvmeibc_use_pcpu_cq=true`, connections are created via:
- `create_qp_per_dev_cq()` (clnt/nvmeibc_ib_net.c:2642)
- Uses `nvmeib_rdma_connect()` with `c_rdma_cm_event()` handler for CM events

---

## Mode 2: Private CQ with Dedicated Kthread per CQ

**Active when:** `nvmeibc_use_pcpu_cq=false` AND `rcq_offload_enb=true` / `scq_offload_enb=true`

**Location:** `clnt/nvmeibc_ib_net.c`

### Architecture

Each connection has its own send and receive CQs, with dedicated kernel threads for offloading completion processing.

#### Key Components

1. **Private CQs**
   - One send CQ and one receive CQ per connection (unless `shared_cq=true`)
   - Created in: `create_qp_private_cq()` (clnt/nvmeibc_ib_net.c:2492)

2. **Dedicated Kthreads**
   - **RCQ Kthread**: `rcq_kthread_func()` (line 1754)
     - Processes receive completions
   - **SCQ Kthread**: `scq_kthread_func()` (line 1033)
     - Processes send completions

3. **Poll Mode States**
   ```c
   enum cq_poll_mode {
       NVMEIBC_IB_CQ_INTR = 0,          // Interrupt mode
       NVMEIBC_IB_CQ_POLLING = 1,        // Actively polling
       NVMEIBC_IB_CQ_KEEP_POLLING = 2,   // Keep polling (missed event)
   };
   ```

### Operation Flow

1. **Interrupt Arrives**
   - Completion handler: `send_completion_intr()` or `recv_completion_intr()`
   - Example: `intr_process_recv_cq_()` (line 1967)

2. **Wake Up Kthread**
   - Interrupt shaper determines if polling should be activated
   - Thread state changes: `NVMEIBC_IB_CQ_INTR` → `NVMEIBC_IB_CQ_POLLING`
   - Wakes up kthread: `wake_up_process(net->rcq_kthread)`

3. **Kthread Polling**
   - Thread continuously polls: `polling_process_recv_cq_()` (line 1780)
   - Polls until CQ is empty or budget exhausted
   - Uses interrupt shaper to decide when to continue polling

4. **Return to Sleep**
   - Thread re-arms interrupt: `ib_req_notify_cq()`
   - Sets state back to `NVMEIBC_IB_CQ_INTR`
   - Calls `schedule()` to sleep

### CPU Binding

Threads can be bound to specific CPUs:
- `rcq_offload_cpu` parameter in `nvmeibc_ib_net_params`
- `scq_offload_cpu` parameter in `nvmeibc_ib_net_params`
- Created with: `rcq_kthread_create_()` / `scq_kthread_create_()`
- Binding: `kthread_create_on_node()` and `kthread_bind()`

### Use Cases

#### Nordda Channels (RDMA Read/Write)
```c
// clnt/nvmeibc_ib_nordda_channel.c:1025
params->rcq_offload_enb = true;  // Enable RCQ offload thread
params->scq_offload_enb = false; // Disable SCQ offload (processed in interrupt)
params->rcq_offload_cpu = is_pcpu_nrch(ch) ? pcpu_nrch_cpu_get(ch) : WORK_CPU_UNBOUND;
```

#### IO Channels (NVMe commands)
```c
// clnt/nvmeibc_ib_io_channel.c:796
params->rcq_offload_enb = true;   // Enable RCQ offload thread
params->scq_offload_enb = false;  // SCQ processed directly
```

#### Lock Channels
```c
// clnt/nvmeibc_locks_channel.c:1527
params->scq_offload_enb = nvmeibc_lock_ch_scq_offload_thread; // Configurable
```

#### Admin Channels
```c
// clnt/nvmeibc_ib_admin_channel.c:1369
params->rcq_offload_enb = false;  // Process in interrupt
params->scq_offload_enb = false;  // Process in interrupt
```

### Advantages

- **Isolation**: Each connection has independent processing
- **Predictability**: Dedicated thread ensures consistent latency
- **CPU Affinity**: Can bind to specific CPUs for NUMA optimization
- **Fine Control**: Per-connection configuration

### Disadvantages

- **Scalability**: One thread per CQ can be resource-intensive
- **Context Switching**: More threads = more scheduling overhead

---

## Mode 3: Private CQ with Workqueue Deferral

**Active when:** `nvmeibc_use_pcpu_cq=false` AND `defer_recv_intr_wq != NULL`

**Location:** `clnt/nvmeibc_ib_net.c`

### Architecture

Completions are initially handled in interrupt context, then deferred to a workqueue for processing.

#### Key Components

1. **Interrupt Handler**
   - Receives completion interrupt
   - Schedules work to workqueue: `defer_recv_interrupts_()` (line 2158)

2. **Workqueue**
   - Per-connection workqueue (`struct workq_struct *defer_recv_intr_wq`)
   - Work function: `poll_cq_and_process_work()` (line 2160)
   - Processes completions via `poll_cq_and_process()` (line 2074)

3. **CQ Polling**
   - `ib_poll_cq()` retrieves completions in batches
   - Processes via `process_num_mixed_comps_()` (line 2120)
   - Re-schedules if more completions available

### Operation Flow

1. **Interrupt Arrives**
   - Completion interrupt handler executes
   - Calls `defer_recv_interrupts_()` (line 2158)

2. **Defer to Workqueue**
   - Adds work to WQ: `wq_add_work(net->defer_recv_intr_wq, &net->defer_recv_work)` (line 2258)
   - Returns from interrupt immediately

3. **Workqueue Processes**
   - Work handler: `poll_cq_and_process_work()` (line 2160)
   - Polls CQ: `ib_poll_cq(net->recv_cq, net->n_wc_mixed, net->wc_mixed)` (line 2107)
   - Processes completions: `process_num_mixed_comps_()` (line 2120)

4. **Completion or Re-schedule**
   - If CQ not empty: re-schedules itself (`*resched = true`, line 2135)
   - If CQ empty: re-arms interrupt and exits

### Configuration

```c
// clnt/nvmeibc_ib_nordda_channel.c:1023
if (params->nr_defer_recv_comps) {
    // Create workqueue
    ch->rc_wq = rc_wq_create(ch, pname, params->comp_cpu);
    params->defer_recv_intr_wq = ch->rc_wq;
    params->rcq_offload_enb = false;  // Mutually exclusive with offload thread
}
```

### Use Cases

#### Nordda Channels with nr_defer_recv_comps
```c
// Enabled when nr_defer_recv_comps module parameter is set
params->nr_defer_recv_comps = P2NV(ch->lionic->port)->dev_type == DT_siw ? 
                               nr_defer_recv_comps_tcp : nr_defer_recv_comps;
```

### Advantages

- **Fast Interrupt Path**: Minimal work in interrupt context
- **Flexible Scheduling**: Workqueue handles CPU scheduling
- **CPU Binding**: Workqueue can be bound to specific CPU

### Disadvantages

- **Latency**: Additional scheduling latency vs. direct processing
- **Complexity**: More state transitions

### Mutual Exclusivity

Cannot be used with `rcq_offload_enb`:
```c
// clnt/nvmeibc_ib_net.c:3500
if (params->rcq_offload_enb && params->defer_recv_intr_wq) {
    _NT(trace_1_ib_net_nvmeibc_ib_net_alloc, "Invalid params: mutual exclusive features");
    return -1;
}
```

---

## Mode 4: Direct Interrupt Processing (Interrupt-Only)

**Active when:** `nvmeibc_use_pcpu_cq=false` AND `rcq_offload_enb=false` AND `defer_recv_intr_wq=NULL`

**Location:** `clnt/nvmeibc_ib_net.c`

### Architecture

All completion processing happens directly in the interrupt handler context. This is the simplest mode.

#### Key Components

1. **Interrupt Handlers**
   - Send completions: `send_completion_intr()` (line 1124)
   - Receive completions: `recv_completion_intr()` (line 2204)

2. **Direct Processing**
   - Polls CQ: `ib_poll_cq()`
   - Processes completions immediately
   - Re-arms interrupt before returning

### Operation Flow

1. **Interrupt Arrives**
   - IB device signals completion
   - Interrupt handler invoked

2. **Poll and Process**
   - Handler polls CQ: `ib_poll_cq(cq, n_wc, wc_array)`
   - Processes each completion:
     - Send: calls `params->call_send_comp_handler()`
     - Receive: calls `params->call_receive_comp_handler()`

3. **Re-arm and Return**
   - Re-arms interrupt: `ib_req_notify_cq(cq, IB_CQ_NEXT_COMP)`
   - Returns from interrupt

### Interrupt Polling Loop

```c
// clnt/nvmeibc_ib_net.c:2268
for (int i = 0; !done && i < nvmeibc_max_notify_cq_iterations; i++) {
    n = ib_poll_cq(recv_cq, n_wc, wc);
    if (n > 0) {
        process_completions(net, wc, n);
        // Continue polling if CQ not empty
    }
    
    // Try to re-arm interrupt
    if (ib_req_notify_cq(recv_cq, IB_CQ_NEXT_COMP) == 0) {
        // Check for race: new completions after re-arm
        n = ib_poll_cq(recv_cq, n_wc, wc);
        if (n == 0)
            done = true;  // Successfully re-armed and CQ empty
    }
}
```

### Use Cases

#### Admin Channels
```c
// clnt/nvmeibc_ib_admin_channel.c:1369
params->rcq_offload_enb = false;
params->scq_offload_enb = false;
// Low rate control messages, simple interrupt processing is sufficient
```

#### Low-Latency Scenarios
- Minimal latency (no context switching)
- Low to moderate completion rates
- Simple completion handlers

### Advantages

- **Lowest Latency**: No context switches or deferrals
- **Simplicity**: Straightforward code path
- **No Thread Overhead**: No additional threads or workqueues

### Disadvantages

- **Interrupt Time**: All processing in interrupt context (can't sleep)
- **CPU Monopolization**: High completion rates can monopolize CPU
- **Limited Complexity**: Completion handlers must be interrupt-safe

### Configuration

```c
// Module parameter controls max iterations in interrupt
uint nvmeibc_max_notify_cq_iterations = 10;
module_param_named(max_notify_cq_iterations, nvmeibc_max_notify_cq_iterations, uint, 0644);
```

---

## Shared CQ Mode

**Active when:** `shared_cq=true` (within private CQ modes)

**Location:** `clnt/nvmeibc_ib_net.c`

### Overview

Instead of separate send and receive CQs, uses a single shared CQ for both.

### Configuration

```c
// clnt/nvmeibc_ib_nordda_channel.c:1050
params->shared_cq = P2NV(ch->lionic->port)->dev_type == DT_siw ? 
                    nr_shared_cq_tcp : nr_shared_cq;
```

### Implementation

```c
// clnt/nvmeibc_ib_net.c:2562
if (params->shared_cq) {
    _NTn(create_qp_private_cq_t1, net,
         "sharing recv cq @PTR as send cq", send_cq);
    send_cq = recv_cq;  // Reuse receive CQ for send
    send_intr = recv_intr;
}
```

### Advantages

- **Resource Efficiency**: One CQ instead of two
- **Unified Processing**: Single polling/interrupt path
- **Nordda Optimization**: RDMA read/write workloads can benefit

### Module Parameters

```c
// Nordda channels
bool nr_shared_cq = false;        // For RDMA (InfiniBand/RoCE)
bool nr_shared_cq_tcp = false;    // For iWARP/TCP

// Performance optimization
bool EC_PERF_CLNT_NORDDA_SHARED_CQ = false;
// When enabled, disables send CQ re-arming (clnt/nvmeibc_ib_nordda_channel.c:993)
```

---

## Mode Selection Decision Tree

```
nvmeibc_use_pcpu_cq=true?
├─ YES: Mode 1 - Per-CPU Shared CQ (dev_cq)
│   ├─ Per-CPU CQs with ipoller threads
│   ├─ Uses SRQ (Shared Receive Queue)
│   └─ Interrupt + polling framework
│
└─ NO: Private CQ modes
    ├─ defer_recv_intr_wq != NULL?
    │   ├─ YES: Mode 3 - Workqueue Deferral
    │   │   ├─ Interrupt → workqueue → process
    │   │   └─ Fast interrupt path
    │   │
    │   └─ NO: Continue...
    │
    ├─ rcq_offload_enb=true OR scq_offload_enb=true?
    │   ├─ YES: Mode 2 - Dedicated Kthread
    │   │   ├─ Per-CQ kernel thread
    │   │   ├─ Interrupt → wake thread → poll
    │   │   └─ Optional CPU binding
    │   │
    │   └─ NO: Mode 4 - Direct Interrupt Processing
    │       ├─ All processing in interrupt
    │       └─ Lowest latency, simplest path
    │
    └─ shared_cq=true? (applies to all private CQ modes)
        ├─ YES: Use single CQ for send+receive
        └─ NO: Separate send and receive CQs
```

---

## Channel Type Default Configurations

### Admin Channels
```c
// clnt/nvmeibc_ib_admin_channel.c
use_pcpu_cq=false:
  rcq_offload_enb = false
  scq_offload_enb = false
  → Mode 4: Direct interrupt processing
```

### IO Channels (NVMe Commands)
```c
// clnt/nvmeibc_ib_io_channel.c
use_pcpu_cq=false:
  rcq_offload_enb = true
  scq_offload_enb = false
  → Mode 2: RCQ with dedicated kthread, SCQ direct interrupt
```

### Nordda Channels (RDMA Read/Write)
```c
// clnt/nvmeibc_ib_nordda_channel.c
use_pcpu_cq=false:
  if nr_defer_recv_comps:
    defer_recv_intr_wq = <workqueue>
    → Mode 3: Workqueue deferral
  else:
    rcq_offload_enb = true
    → Mode 2: RCQ with dedicated kthread
    
  scq_offload_enb = false
  shared_cq = nr_shared_cq / nr_shared_cq_tcp
```

### Lock Channels
```c
// clnt/nvmeibc_locks_channel.c
use_pcpu_cq=false:
  scq_offload_enb = nvmeibc_lock_ch_scq_offload_thread (default: true)
  → Mode 2: SCQ with dedicated kthread
```

---

## Performance Considerations

### Mode 1 (Per-CPU Shared CQ)
**Best for:**
- High scalability (many connections)
- Multi-core systems
- Mixed workloads across CPUs
- Memory-constrained environments

**Considerations:**
- Requires SRQ support
- More complex debugging
- Higher memory per CQ (but fewer CQs)

### Mode 2 (Dedicated Kthread)
**Best for:**
- Medium to high completion rates
- NUMA-aware deployments (CPU binding)
- Consistent latency requirements
- Mixed interrupt and deferred processing

**Considerations:**
- Thread overhead (one per CQ)
- Context switch latency
- CPU binding helps NUMA performance

### Mode 3 (Workqueue Deferral)
**Best for:**
- Bursty traffic patterns
- CPU-bound completion handlers
- Need to avoid interrupt context restrictions

**Considerations:**
- Additional scheduling latency
- Workqueue scheduling overhead

### Mode 4 (Direct Interrupt)
**Best for:**
- Ultra-low latency requirements
- Low to moderate completion rates
- Simple completion handlers
- Control plane operations

**Considerations:**
- Limited by interrupt budget
- Can monopolize CPU under load
- Must be interrupt-safe

---

## Related Module Parameters

```bash
# Primary mode selection
nvmeibc.use_pcpu_cq=1              # Enable per-CPU shared CQ mode (default)

# Per-CPU CQ settings (Mode 1)
nvmeib_common.nvmeib_pcpu_cq_size=8192          # CQ size
nvmeib_common.nvmeib_pcpu_cq_max_cqs_per_dev=0  # 0=num_comp_vectors
nvmeib_common.nvmeib_pcpu_cq_all_cpus=1         # One CQ per CPU
nvmeib_common.ipoller_poll_duration_jif=0       # Polling duration

# Private CQ settings (Modes 2-4)
nvmeibc.max_notify_cq_iterations=10             # Max CQ polling iterations in interrupt

# Nordda-specific
nvmeibc.nr_defer_recv_comps=0      # Enable workqueue deferral (Mode 3)
nvmeibc.nr_shared_cq=0             # Enable shared CQ for Nordda
nvmeibc.nr_shared_cq_tcp=0         # Enable shared CQ for Nordda over TCP

# Lock channel
nvmeibc.lock_ch_scq_offload_thread=1     # Enable SCQ kthread for lock channels

# Interrupt shaping (affects transition to polling)
nvmeib_common.nvmeib_intr_shaper_enb=1   # Enable interrupt shaper
```

---

## Code References

### Mode 1: Per-CPU Shared CQ
- **CQ Structure**: `common/nvmeib.c:391-457` (`struct nvmeib_dev_cq`)
- **CQ Init**: `common/nvmeib.c:2110` (`init_cqs()`)
- **Interrupt Handler**: `common/nvmeib.c:1583` (`cq_completion_intr()`)
- **Polling Framework**: `common_public/poll/nvmeib_public_intr_poll.c:72` (`ipoller_run()`)
- **QP Creation**: `clnt/nvmeibc_ib_net.c:2642` (`create_qp_per_dev_cq()`)

### Mode 2: Dedicated Kthread
- **RCQ Kthread**: `clnt/nvmeibc_ib_net.c:1754` (`rcq_kthread_func()`)
- **SCQ Kthread**: `clnt/nvmeibc_ib_net.c:1033` (`scq_kthread_func()`)
- **Thread Creation**: `clnt/nvmeibc_ib_net.c:1876` (`rcq_kthread_create_()`)
- **Interrupt Handler**: `clnt/nvmeibc_ib_net.c:1967` (`intr_process_recv_cq_()`)

### Mode 3: Workqueue Deferral
- **Defer Function**: `clnt/nvmeibc_ib_net.c:2158` (`defer_recv_interrupts_()`)
- **Work Handler**: `clnt/nvmeibc_ib_net.c:2160` (`poll_cq_and_process_work()`)
- **Poll Function**: `clnt/nvmeibc_ib_net.c:2074` (`poll_cq_and_process()`)
- **WQ Creation**: `clnt/nvmeibc_ib_nordda_channel.c:875` (`rc_wq_create()`)

### Mode 4: Direct Interrupt
- **Send Interrupt**: `clnt/nvmeibc_ib_net.c:1124` (`send_completion_intr()`)
- **Recv Interrupt**: `clnt/nvmeibc_ib_net.c:2204` (`recv_completion_intr()`)
- **Private CQ Creation**: `clnt/nvmeibc_ib_net.c:2492` (`create_qp_private_cq()`)

### Common Infrastructure
- **Net Alloc**: `clnt/nvmeibc_ib_net.c:3418` (`nvmeibc_ib_net_alloc()`)
- **Net Params**: `clnt/nvmeibc_ib_net.h:238-276` (`struct nvmeibc_ib_net_params`)
- **Poll Mode Enum**: `clnt/nvmeibc_ib_net.h:317-323` (`enum cq_poll_mode`)

---

## Debugging and Monitoring

### Proc Files

```bash
# Per-CPU CQ statistics (when use_pcpu_cq=1)
/proc/nvmeib_common/nics/<nic>/cq_stats

# Connection statistics (all modes)
/proc/nvmeibc/<instance>/disks/<disk>/channels/<channel>/net/stats
```

### Trace Points

Key trace points for debugging completion processing:

```c
// Mode 1 tracing
_NI_dmesg(e1_cq_completion_intr, "Dev CQ @DEV_CQ - Interrupt while disabled", cqw);

// Mode 2 tracing  
_NTn(trace_ib_net_rcq_kthread_func, net, "rcq thread ready");
_NDn(trace_2_ib_net_rcq_kthread_func, net, "transition back to IRQ");

// Mode 3 tracing
_NTn(trace_ib_net_poll_cq_and_process_work, net, "defer work");

// Mode 4 tracing
_NDn(trace_recv_completion_intr, net, "recv completion");
```

### Statistics Fields

```c
struct cq_stats {
    u64 n_intr;              // Completions processed in interrupt mode
    u64 n_poll;              // Completions processed in polling mode
    u64 n_wakeups_burst;     // Thread wakeups due to burst
    u64 n_defer_wq;          // Completions deferred to workqueue
    u64 n_missed_events;     // Missed completion events
    u64 n_spurious_intr;     // Spurious interrupts
    // ... more fields
};
```

---

## Migration Guide

### From Direct Interrupt to Per-CPU CQ

```bash
# Before (direct interrupt)
modprobe nvmeibc use_pcpu_cq=0

# After (per-CPU CQ)
modprobe nvmeibc use_pcpu_cq=1
```

**Changes:**
- Connections will use shared CQs and SRQs
- Interrupt processing moves to per-CPU ipoller threads
- May need to adjust CQ size: `nvmeib_pcpu_cq_size`
- Consider IRQ affinity: `set_irq_affinity_cpulist.sh`

### From Kthread to Workqueue

```bash
# Enable workqueue deferral for Nordda channels
modprobe nvmeibc nr_defer_recv_comps=1
```

**Changes:**
- RCQ processing moves from dedicated kthread to workqueue
- May affect latency characteristics
- Reduces number of threads

---

## Future Enhancements

Potential areas for future development:

1. **Adaptive Mode Selection**: Automatically choose mode based on workload
2. **Hybrid Modes**: Combine strategies for different traffic types
3. **User-Space Polling**: Enhanced SPDK integration
4. **Dynamic Rebalancing**: Move CQs between CPUs based on load
5. **Completion Batching**: Improved batch processing strategies

---

## Summary Table

| Mode | Key Feature | Best Use Case | Latency | Scalability | Complexity |
|------|-------------|---------------|---------|-------------|------------|
| **1: Per-CPU CQ (dev_cq)** | Shared CQ per CPU with ipoller | High connection count, multi-core | Medium | Excellent | High |
| **2: Dedicated Kthread** | Per-CQ kernel thread | Consistent latency, NUMA-aware | Low-Med | Medium | Medium |
| **3: Workqueue Deferral** | Defer to workqueue | Bursty traffic, complex handlers | Medium | Good | Medium |
| **4: Direct Interrupt** | Process in interrupt | Ultra-low latency, low rate | Lowest | Limited | Low |

---

## References

- InfiniBand Architecture Specification
- Linux kernel `Documentation/infiniband/`
- RDMA Core library documentation
- Mellanox/NVIDIA RDMA performance tuning guides

---

**Document Version:** 1.0  
**Last Updated:** 2026-01-27  
**Maintainer:** NVMe over IB Team

