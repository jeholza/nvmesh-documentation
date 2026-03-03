# Per-CPU NoRDDA Channels and Coremask Support

## Overview

The NVMesh client supports two advanced channel management modes for optimizing performance in multi-CPU systems:

1. **Per-CPU NoRDDA Channels**: Dedicated channels for specific CPUs to reduce contention
2. **Coremask Support**: Dynamic channel allocation based on CPU affinity masks

Both features aim to improve performance by reducing lock contention and increasing CPU cache locality.

## Per-CPU NoRDDA Channels

### Concept

Standard NoRDDA channels are "any-CPU" - any CPU can use them. This requires locking:

```c
// Any-CPU channel access
spin_lock(&disk->spinlock);
nrch = get_available_nordda_channel();
context = get_io_context(nrch);
spin_unlock(&disk->spinlock);
```

Per-CPU channels are dedicated to specific CPUs, eliminating contention:

```c
// Per-CPU channel access (lockless mode)
int cpu = smp_processor_id();
nrch = disk->info->avail_nordda_for_cpu[cpu];
context = get_io_context(nrch);  // No lock needed!
```

### Configuration

#### Module Parameters

```bash
# Number of per-CPU channels
nr_pcpu_channels_per_disk=0     # 0=disabled (default)
                                # 1=auto (one per CPU)
                                # N=specific number

# Lockless mode (requires channel for every submission CPU)
nr_pcpu_ch_lockless=false       # false=locking (default)
                                # true=lockless (fastest)

# CPU mask for lockless channels (hex comma-separated)
nr_pcpu_ch_ll_cpus=""           # ""=all CPUs (default)
                                # "ff"=CPUs 0-7
                                # "ff,ff"=CPUs 0-15
```

### Modes of Operation

#### Mode 1: Disabled (Default)
```bash
nr_pcpu_channels_per_disk=0
```
- All channels are any-CPU
- Standard locking applies
- Simplest configuration

#### Mode 2: Per-CPU with Locking
```bash
nr_pcpu_channels_per_disk=16
nr_pcpu_ch_lockless=false
```
- Creates 16 per-CPU channels
- CPU assignment: `cpu % 16`
- Multiple CPUs can share a channel (with locking)
- Better than any-CPU but not lockless

**Use case**: More CPUs than channels available

#### Mode 3: Lockless Per-CPU (Highest Performance)
```bash
nr_pcpu_channels_per_disk=1     # Auto: one per CPU
nr_pcpu_ch_lockless=true
nr_pcpu_ch_ll_cpus=""           # All CPUs
```
- Each submission CPU gets dedicated channel
- **No locking required**
- Best performance, highest channel count

**Requirements**:
- Must have channel for **every** submission CPU
- CPUs without channel fall back to any-CPU channels (with locking)

#### Mode 4: Lockless with CPU Mask
```bash
nr_pcpu_channels_per_disk=8
nr_pcpu_ch_lockless=true
nr_pcpu_ch_ll_cpus="ff"         # CPUs 0-7
```
- Channels dedicated to CPUs 0-7
- These CPUs operate locklessly
- Other CPUs use any-CPU channels with locking

**Use case**: Optimize specific CPU range, limit channel count

### Implementation

#### Initialization

```c
static void pcpu_nrch_init_mode(struct nvmeibc_disk *disk)
{
    disk->pcpu_nrchs = nvmeibc_nr_pcpu_channels_per_disk;
    disk->pcpu_nrchs_ll = nvmeibc_nr_pcpu_ch_lockless;
    
    if (disk->pcpu_nrchs == 1) {
        // Auto mode: one per CPU
        disk->pcpu_nrchs = min(num_online_cpus(), 128);
    }
    
    // Parse CPU mask for lockless mode
    if (disk->pcpu_nrchs_ll && nvmeibc_nr_pcpu_ch_ll_cpus) {
        parse_cpu_mask(nvmeibc_nr_pcpu_ch_ll_cpus, &disk->pcpu_nrchs_ll_cpus);
    }
}
```

#### Pool Initialization

```c
static void pcpu_nrch_init_pool(struct nvmeibc_disk_info *info)
{
    // Array mapping CPU -> NoRDDA channel
    info->avail_nordda_for_cpu = 
        kzalloc(sizeof(*info->avail_nordda_for_cpu) * NVMEIB_DFLT_MAX_CPUS, ...);
    
    // Initialize all entries to NULL
    memset(info->avail_nordda_for_cpu, 0, ...);
}
```

#### Channel Assignment

```c
static void pcpu_nrch_add(struct nvmeibc_disk *disk,
                         struct nvmeibc_ib_nordda_channel *nrch)
{
    int cpu = pcpu_nrch_get_next_cpu(disk);
    
    if (cpu < 0) {
        // All CPUs assigned, add to any-CPU list
        list_add_tail(&nrch->available_link, &disk->info->available_norddas);
        return;
    }
    
    // Assign to specific CPU
    nrch->base.pcpu_ch_cpu = cpu;
    nrch->base.flags |= NVMEIBC_CHANNEL_FLAG_PCPU_CH;
    
    if (disk->pcpu_nrchs_ll && 
        NVMEIB_CPU_MASK_TEST_CPU(cpu, &disk->pcpu_nrchs_ll_cpus)) {
        // Lockless mode for this CPU
        nrch->base.flags |= NVMEIBC_CHANNEL_FLAG_LL_PCPU_CH;
    }
    
    disk->info->avail_nordda_for_cpu[cpu] = nrch;
}
```

#### Channel Selection

```c
static int pcpu_nrch_get_channel(struct nvmeibc_disk *disk,
                                struct nvmeibc_disk_command *disk_cmd,
                                struct nvmeibc_channel **ch,
                                void **context)
{
    int cpu = smp_processor_id();
    struct nvmeibc_ib_nordda_channel *nrch;
    
    // Try per-CPU channel first
    nrch = disk->info->avail_nordda_for_cpu[cpu];
    if (nrch && nvmeibc_channel_is_ll_pcpu_ch(&nrch->base)) {
        // Lockless per-CPU channel
        *context = nvmeibc_ib_nordda_channel_get_io_context(nrch);
        if (*context) {
            *ch = &nrch->base;
            return 0;
        }
    }
    
    // Fall back to standard channel selection
    return nvmeibc_disk_get_channel(disk, disk_cmd, ch, context);
}
```

### Lockless Operation

#### Key Insight

For lockless operation to work:
```c
// CPU 0 always submits on CPU 0
// CPU 1 always submits on CPU 1
// ...
// CPU N always submits on CPU N

// Each CPU has its own channel
// No cross-CPU access = no locks needed
```

#### Validation

```c
static bool pcpu_nrch_check_reuse(struct nvmeibc_disk *disk,
                                  struct nvmeibc_disk_command *disk_cmd,
                                  struct nvmeibc_channel *ch,
                                  void *context)
{
    int cpu = smp_processor_id();
    int ch_cpu = nvmeibc_channel_pcpu_ch_get_cpu(ch);
    
    if (!disk->pcpu_nrchs_ll || ch_cpu == cpu) {
        // Not lockless OR correct CPU
        return true;
    }
    
    // Lockless channel accessed from wrong CPU - violation!
    WARN_ON_ONCE(1);
    return false;
}
```

If a lockless channel is accessed from the wrong CPU, the reuse is rejected and a warning is issued.

### Cleanup

```c
static void pcpu_nrch_del(struct nvmeibc_disk *disk,
                         struct nvmeibc_ib_nordda_channel *nrch,
                         struct list_head *pend_list)
{
    int cpu = nvmeibc_channel_pcpu_ch_get_cpu(&nrch->base);
    
    if (cpu >= 0) {
        // Remove from per-CPU array
        disk->info->avail_nordda_for_cpu[cpu] = NULL;
        
        // Move pending commands to provided list
        // (Will be resubmitted on any-CPU channels)
        // ...
    }
}
```

## Coremask Support

### Concept

Coremask support allows dynamic allocation of dedicated channels for specific CPU masks. This is useful for:
- **DPDK/SPDK applications**: Dedicate channels to specific worker threads
- **NUMA optimization**: Channels local to specific NUMA nodes
- **QoS**: Separate fast/slow paths by CPU

### Architecture

```
Disk
 └── Coremask Info
      ├── Coremask 1 (CPUs 0-7, UID: 12345)
      │    ├── NoRDDA Channel 1 (CPU 0)
      │    ├── NoRDDA Channel 2 (CPU 4)
      │    └── Pending Queue
      ├── Coremask 2 (CPUs 8-15, UID: 12346)
      │    ├── NoRDDA Channel 3 (CPU 8)
      │    ├── NoRDDA Channel 4 (CPU 12)
      │    └── Pending Queue
      └── Masks Tree (radix tree for lookup)
```

### Configuration

#### Module Parameters

```bash
# Enable coremask support
disk_coremask_support=false     # false=disabled (default)
                                # true=enabled

# Max channels per coremask
disk_max_coremask_nrch=2        # Default: 2 channels per mask

# Use coremask-specific pending queue
disk_use_coremask_pending=true  # true=coremask has own queue
                                # false=fall back to any-CPU
```

### Coremask Structure

```c
struct nvmeibc_disk_coremask_chs {
    struct list_head link;              // Link in disk's coremask list
    struct nvmeibc_disk *disk;          // Back pointer
    
    // Coremask identification
    u64 uid;                            // Unique ID from management
    struct nvmeib_cpu_mask cpu_coremask; // CPU mask (e.g., CPUs 0-7)
    int n_cpus;                         // Number of CPUs in mask
    
    // Channels
    struct list_head nrchs_list;        // NoRDDA channels for this mask
    struct nvmeib_cpu_mask nrchs_coremask; // CPUs that have channels
    int n_nrchs;                        // Number of NoRDDA channels
    
    // Lock channels
    struct nvmeib_cpu_mask lchs_coremask;  // CPUs for lock channels
    int n_lchs;                         // Number of lock channels
    
    // Pending queue (if disk_use_coremask_pending=true)
    struct list_head pending_cmds;      // Pending commands
    int n_pending;                      // Count
    int max_pending;                    // Limit
    
    // State
    int dying;                          // Coremask being removed
    struct kref refcnt;                 // Reference counting
    spinlock_t spinlock;                // Protects structure
    
    // Statistics
    u64 n_io_mask_chan;                 // I/Os using coremask channel
    u64 n_io_mask_dying;                // Rejected: mask dying
    u64 n_io_mask_uid_mismatch;         // Rejected: UID changed
    u64 n_io_mask_no_nrch;              // No channel available
    u64 n_io_mask_nrch_busy;            // Channel busy
    // ... more stats ...
};
```

### Coremask Lifecycle

#### 1. Adding a Coremask

From management (CLI/API):
```bash
# Add coremask for CPUs 0-7
nvmesh_coremask_add --disk vol1 --cpus 0-7 --uid 12345
```

In kernel:
```c
int nvmeibc_disk_notify_coremask_update(struct nvmeibc_disk *disk)
{
    // Schedule coremask update
    struct nvmeibc_disk_update_data *update_data;
    update_data->update_type = DISK_UPDATE_COREMASK_UPDATE;
    nvmeibc_disk_update_config(disk, update_data, true);
}
```

#### 2. Update Processing

Two-phase update:

**Phase 1: Main Work Queue**
```c
static void disk_coremask_update_main_work(struct workqe_struct *work)
{
    // 1. Lock masks tree
    mutex_lock(&cinfo->masks_guard);
    
    // 2. Update masks tree from management DB
    update_masks_from_db(disk, cinfo);
    
    // 3. Mark deprecated masks
    mark_deprecated_masks(cinfo);
    
    // 4. Allocate new coremask structures
    allocate_new_coremask_chs(disk, cinfo);
    
    mutex_unlock(&cinfo->masks_guard);
}
```

**Phase 2: Admin Work Queue**
```c
static void disk_coremask_update_admin_work(struct workqe_struct *work)
{
    // Connect channels for new coremasks
    list_for_each_entry(coremask_chs, &cinfo->coremask_chs, link) {
        if (needs_channels(coremask_chs)) {
            // Connect up to disk_max_coremask_nrch channels
            for (i = 0; i < disk_max_coremask_nrch; i++) {
                connect_coremask_nrch(disk, coremask_chs);
            }
        }
    }
}
```

#### 3. Removing a Coremask

```c
static void remove_coremask(struct nvmeibc_disk *disk, u64 uid)
{
    // 1. Mark as dying
    coremask_chs->dying = 1;
    
    // 2. Fail pending commands
    fail_pending_commands(coremask_chs);
    
    // 3. Disconnect channels
    disconnect_coremask_channels(coremask_chs);
    
    // 4. Remove from tree
    radix_tree_delete(&cinfo->masks_tree, uid);
    
    // 5. Release structure
    kref_put(&coremask_chs->refcnt, release_coremask_chs);
}
```

### Channel Selection with Coremask

```c
static int coremask_get_channel(struct nvmeibc_disk *disk,
                               struct nvmeibc_disk_command *disk_cmd,
                               struct nvmeibc_channel **ch,
                               void **context)
{
    const struct nvmeib_cpu_mask_info *cmd_coremask = disk_cmd->cpu_mask_info;
    struct nvmeibc_disk_coremask_chs *coremask_chs;
    
    if (!cmd_coremask)
        return -EINVAL;  // No coremask specified
    
    // Lookup coremask by UID
    coremask_chs = radix_tree_lookup(&cinfo->masks_tree, cmd_coremask->gen);
    if (!coremask_chs)
        return -ENOENT;  // Coremask not found
    
    spin_lock(&coremask_chs->spinlock);
    
    if (coremask_chs->dying) {
        // Coremask being removed
        spin_unlock(&coremask_chs->spinlock);
        return -ENODEV;
    }
    
    if (coremask_chs->uid != cmd_coremask->gen) {
        // UID mismatch (coremask changed)
        spin_unlock(&coremask_chs->spinlock);
        return -ESTALE;
    }
    
    // Try to get channel
    list_for_each_entry(nrch, &coremask_chs->nrchs_list, coremask_link) {
        if ((*context = nvmeibc_ib_nordda_channel_get_io_context(nrch))) {
            *ch = &nrch->base;
            spin_unlock(&coremask_chs->spinlock);
            return 0;
        }
    }
    
    // No channel available
    if (disk_use_coremask_pending) {
        // Add to coremask pending queue
        list_add_tail(&disk_cmd->dcmd_link, &coremask_chs->pending_cmds);
        coremask_chs->n_pending++;
        spin_unlock(&coremask_chs->spinlock);
        return 0;
    }
    
    // Fall back to any-CPU channels
    spin_unlock(&coremask_chs->spinlock);
    return nvmeibc_disk_get_channel(disk, disk_cmd, ch, context);
}
```

### Coremask Reuse Validation

```c
static bool pcpu_nrch_coremask_check_reuse(struct nvmeibc_disk *disk,
                                          struct nvmeibc_disk_command *disk_cmd,
                                          struct nvmeibc_channel *ch,
                                          void *context)
{
    const struct nvmeib_cpu_mask_info *cmd_coremask = disk_cmd->cpu_mask_info;
    struct nvmeibc_disk_coremask_chs *coremask_chs;
    
    coremask_chs = nvmeibc_channel_get_coremask_ch_cookie(ch);
    if (!coremask_chs)
        return false;  // No coremask cookie
    
    spin_lock(&coremask_chs->spinlock);
    
    if (coremask_chs->dying) {
        // Coremask dying
        coremask_chs->n_reuse_io_mask_dying++;
        spin_unlock(&coremask_chs->spinlock);
        return false;
    }
    
    if (coremask_chs->uid != cmd_coremask->gen) {
        // UID changed
        coremask_chs->n_reuse_io_mask_uid_mismatch++;
        spin_unlock(&coremask_chs->spinlock);
        return false;
    }
    
    // Reuse valid
    coremask_chs->n_reuse_io_mask_chan++;
    spin_unlock(&coremask_chs->spinlock);
    return true;
}
```

## Per-CPU NRCH vs Coremask

| Feature | Per-CPU NRCH | Coremask |
|---------|--------------|----------|
| Configuration | Module parameters | Management API |
| Dynamic | No (requires rediscovery) | Yes (runtime updates) |
| CPU Assignment | Round-robin or all CPUs | Specific CPU masks |
| Use Case | Static CPU optimization | Dynamic workload isolation |
| Complexity | Lower | Higher |
| Pending Queue | Shared (any-CPU) | Per-coremask (optional) |
| Statistics | Basic | Detailed per-mask |

## Statistics

### Per-CPU NRCH Statistics

Available in disk status output:
```json
{
  "pcpu_nrchs_enabled": true,
  "pcpu_nrchs_lockless": true,
  "pcpu_nrchs_count": 16,
  "pcpu_nrch_cpu_mask": "ffff",
  "per_cpu_channels": [
    {"cpu": 0, "channel": "nrch0", "used_reqs": 5},
    {"cpu": 1, "channel": "nrch1", "used_reqs": 3},
    ...
  ]
}
```

### Coremask Statistics

Per coremask statistics:
```json
{
  "coremasks": [
    {
      "uid": 12345,
      "cpu_mask": "ff",
      "n_cpus": 8,
      "n_nrchs": 2,
      "n_pending": 0,
      "n_io_mask_chan": 123456,
      "n_io_mask_dying": 0,
      "n_io_mask_uid_mismatch": 5,
      "n_reuse_io_mask_chan": 98765
    }
  ]
}
```

## Performance Impact

### Per-CPU NRCH (Lockless Mode)

**Benefits**:
- Eliminates disk spinlock contention
- Improves CPU cache locality
- Reduces context switches

**Measured improvements** (YMMV):
- **IOPS**: 20-40% improvement on high-core-count systems
- **Latency**: 10-20% reduction in P99 latency
- **CPU**: 5-10% reduction in CPU utilization

**Best for**:
- High core count (16+ cores)
- High IOPS workloads (>100K IOPS per disk)
- Latency-sensitive applications

### Coremask

**Benefits**:
- Workload isolation (QoS)
- NUMA locality
- Integration with DPDK/SPDK

**Best for**:
- Mixed workloads (fast + slow)
- Multi-tenant environments
- Applications with specific CPU affinity requirements

## Configuration Examples

### Example 1: High-Performance OLTP

```bash
# Enable lockless per-CPU channels for all CPUs
modprobe nvmeibc \
    nr_pcpu_channels_per_disk=1 \
    nr_pcpu_ch_lockless=true \
    nr_pcpu_ch_ll_cpus=""
```

### Example 2: NUMA-Aware Configuration

```bash
# Node 0: CPUs 0-15, use lockless per-CPU
# Node 1: CPUs 16-31, use standard any-CPU
modprobe nvmeibc \
    nr_pcpu_channels_per_disk=16 \
    nr_pcpu_ch_lockless=true \
    nr_pcpu_ch_ll_cpus="ffff"
```

### Example 3: SPDK with Coremask

```bash
# Enable coremask support
modprobe nvmeibc \
    disk_coremask_support=true \
    disk_max_coremask_nrch=4

# Add coremask for SPDK worker threads (CPUs 2-9)
nvmesh_coremask_add --disk vol1 --cpus 2-9 --uid 1000

# SPDK application submits I/O with coremask UID 1000
# I/Os will use dedicated channels for CPUs 2-9
```

## Troubleshooting

### Issue: Per-CPU Channels Not Used

**Check**:
```bash
cat /sys/module/nvmeibc/parameters/nr_pcpu_channels_per_disk
cat /proc/nvmesh/disks/<disk>/status | grep pcpu_nrchs
```

**Common causes**:
- Parameter not set or set to 0
- Not enough channels available (check `nr_max_channels_per_disk`)
- Discovery not completed

### Issue: Lockless Mode Warning

```
WARNING: Reuse for channel nrch5 came on wrong CPU 3 (instead of 5)
```

**Cause**: Lockless channel accessed from wrong CPU

**Solutions**:
1. Ensure application pins threads to CPUs
2. Disable lockless mode: `nr_pcpu_ch_lockless=false`
3. Adjust CPU mask: `nr_pcpu_ch_ll_cpus=<mask>`

### Issue: Coremask Commands Failing

**Check**:
```bash
cat /sys/module/nvmeibc/parameters/disk_coremask_support
cat /proc/nvmesh/disks/<disk>/status | grep coremask
```

**Common causes**:
- Coremask support not enabled
- UID mismatch (management DB out of sync)
- Coremask being removed (dying=1)

## Related Documentation

- `CHANNEL_MANAGEMENT.md`: Channel selection and lifecycle
- `DISK_UPDATES.md`: Coremask update mechanism
- `REUSED_BB_RELEASE.md`: Channel reuse with per-CPU/coremask

## Related Files

- `nvmeibc_disk.c`: Per-CPU and coremask implementation
- `nvmeib_cpu_masks.h`: CPU mask utilities
- `nvmeibc_ib_nordda_channel.c`: NoRDDA channel operations

