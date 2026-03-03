# Client Disk Channel Management and Remote I/O Execution

## Overview

This document describes how the client manages I/O channels to remote storage and executes remote I/O operations. The system supports two types of channels:
- **RDDA Channels** (I/O channels with Remote Disk Direct Access): Use RDMA read/write for data transfer
- **NoRDDA Channels** (No-Remote Disk Direct Access): Use send/receive verbs, more flexible routing

## Channel Types

### RDDA vs NoRDDA

| Feature | RDDA Channels | NoRDDA Channels |
|---------|---------------|-----------------|
| Data Transfer | RDMA Read/Write | Send/Receive |
| Memory Registration | Client registers memory | Server registers memory |
| Resource Requirements | Higher (server resources) | Lower (client-side only) |
| Routing Flexibility | Limited | High |
| Performance | Lower latency (direct DMA) | Slightly higher latency |
| Scalability | Limited by server resources | Scales to many clients |
| Use Case | High-performance, fewer clients | High-scale deployments |

### Channel Structure Hierarchy

```
nvmeibc_disk
    └── nvmeibc_disk_info
            ├── available_channels (RDDA)
            │       └── List of nvmeibc_ib_io_channel
            └── available_norddas (NoRDDA)
                    └── List of nvmeibc_ib_nordda_channel
```

Both channel types inherit from `nvmeibc_channel` base structure.

## Remote I/O Execution Flow

### High-Level Flow

```c
int nvmeibc_disk_execute_io(struct nvmeibc_disk *disk,
                            struct nvmeibc_disk_io_command *block_cmd)
{
    // Entry point from block layer
    return execute_io(disk, block_cmd);
}

static int execute_io(struct nvmeibc_disk *disk,
                     struct nvmeibc_disk_io_command *block_cmd)
{
    if (disk->access_local)
        return execute_io_local(disk, &block_cmd->disk_cmd);
    else
        return execute_io_remote(disk, &block_cmd->disk_cmd);  // <--- Remote path
}
```

### execute_io_remote() Main Flow

```c
static int execute_io_remote(struct nvmeibc_disk *disk,
                             struct nvmeibc_disk_command *disk_cmd)
{
    struct nvmeibc_channel *ch;
    void *context = NULL;
    int rv;
    
    // 1. Check for channel reuse (fast path)
    ch = execute_io_remote_check_reuse(disk, disk_cmd, &context);
    if (ch) {
        // Fast path: reuse previous channel
        return ch->execute_io(ch, disk_cmd, context);
    }
    
    // 2. Acquire disk lock
    spin_lock_irqsave(&disk->spinlock, flags);
    
    // 3. Check pending or select channel
    if (per-CPU NoRDDA enabled)
        rv = pcpu_nrch_get_channel(disk, disk_cmd, &ch, &context);
    else
        rv = nvmeibc_disk_get_channel(disk, disk_cmd, &ch, &context);
    
    if (rv == 0 && context) {
        // Got channel with context - execute immediately
        spin_unlock_irqrestore(&disk->spinlock, flags);
        return ch->execute_io(ch, disk_cmd, context);
    } else {
        // No channel available - add to pending queue
        list_add_tail(&disk_cmd->dcmd_link, 
                     &disk->info->pending_disk_cmds[priority]);
        disk->info->tot_pending++;
        spin_unlock_irqrestore(&disk->spinlock, flags);
        return 0;  // Will be processed later
    }
}
```

## Channel Reuse Mechanism

### Reuse Cookie

Commands can "remember" their channel for fast reuse:

```c
struct nvmeib_data_reuse_buf_params {
    enum nvmeib_data_reuse_buf_action action;
    u64 channel_ver;        // Channel version for validation
    void *channel_private;  // Channel-specific context
};
```

### Reuse Validation

```c
static struct nvmeibc_channel *execute_io_remote_check_reuse(
    struct nvmeibc_disk *disk,
    struct nvmeibc_disk_command *disk_cmd,
    void **context)
{
    struct nvmeibc_channel *ch;
    
    // Basic reuse check
    ch = check_reuse(disk, disk_cmd, context);
    if (!ch)
        return NULL;
    
    // Per-CPU channel additional validation
    if (nvmeibc_channel_is_pcpu_ch(ch)) {
        if (!pcpu_nrch_check_reuse(disk, disk_cmd, ch, context)) {
            // Reuse failed - must get new channel
            ch = NULL;
            *context = NULL;
        }
    }
    
    return ch;
}
```

**Reuse fails when**:
- Channel version changed (disconnected/reconnected)
- Channel is dying
- Per-CPU channel accessed from wrong CPU (lockless mode)
- Coremask changed (coremask mode)

## Channel Selection

### Channel Selection Priority

```c
int nvmeibc_disk_get_channel(struct nvmeibc_disk *disk,
                             struct nvmeibc_disk_command *disk_cmd,
                             struct nvmeibc_channel **ch,
                             void **context)
{
    // 1. Try RDDA channels (if available and below watermark)
    if (info->mine > NVMEIBC_WATERMARK_GOTO_NORDDA) {
        ioch = list_first_entry_or_null(&info->available_channels,
                                       struct nvmeibc_ib_io_channel, base.link);
        if (ioch && ioch->state == IO_CHANNEL_ALLOCATED) {
            ioch->state = IO_CHANNEL_IN_USE;
            *ch = &ioch->base;
            *context = NULL;
            return 0;
        }
    }
    
    // 2. Try NoRDDA channels
    if (USE_NORDDA_FOR_IO || info->mine <= NVMEIBC_WATERMARK_GOTO_NORDDA) {
        list_for_each_entry(nrch, &info->available_norddas, available_link) {
            if ((*context = nvmeibc_ib_nordda_channel_get_io_context(nrch))) {
                *ch = &nrch->base;
                // Rotate list for load balancing
                list_move_tail(&nrch->available_link, &info->available_norddas);
                return 0;
            }
        }
    }
    
    // 3. No channel available
    return -1;
}
```

**Selection logic**:
1. Prefer RDDA if `info->mine` (available RDDA resources) is above watermark
2. Fall back to NoRDDA if RDDA unavailable or below watermark
3. If neither available, command goes to pending queue

### Watermark System

```c
#define NVMEIBC_WATERMARK_GOTO_NORDDA  0

info->mine;  // Number of available RDDA channels
```

When `info->mine <= NVMEIBC_WATERMARK_GOTO_NORDDA`, switch to NoRDDA exclusively.

## Pending Command Queue

### Pending Queue Structure

```c
enum disk_pend_prio {
    DISK_PEND_PRIO_IO_RECOV = 0,     // Recovery I/O (highest priority)
    DISK_PEND_PRIO_EC_JRNL_WRITE,     // EC journal writes
    DISK_PEND_PRIO_EC_DATA_WRITE,     // EC data writes
    DISK_PEND_PRIO_NON_EC_WRITE,      // Non-EC writes
    DISK_PEND_PRIO_OTHER_IO,          // Other I/O
    DISK_PEND_PRIO_RPC_LOCKS,         // Lock operations
    DISK_PEND_PRIO_GEN_CMDS,          // General commands (lowest priority)
    DISK_PEND_PRIO_MAX
};

struct nvmeibc_disk_info {
    struct list_head pending_disk_cmds[DISK_PEND_PRIO_MAX];
    int tot_pending;       // Total across all priorities
    int tot_io_pending;    // Just I/O commands
    int n_use_nrch_only;   // Commands requiring NoRDDA
};
```

### Adding to Pending Queue

```c
static int execute_io_remote(...)
{
    // ... failed to get channel ...
    
    int prio = disk_cmd_pend_prio(disk, disk_cmd);
    list_add_tail(&disk_cmd->dcmd_link, 
                 &disk->info->pending_disk_cmds[prio]);
    disk->info->tot_pending++;
    
    if (disk_cmd->cmd_type == NVMEIBC_DISK_CMD_IO)
        disk->info->tot_io_pending++;
    
    return 0;  // Successfully queued
}
```

### Processing Pending Queue

Pending commands are processed when a channel becomes available:

#### RDDA Channel Completion

```c
struct nvmeibc_disk_io_command *nvmeibc_disk_get_block_cmd_rdda(
    struct nvmeibc_disk *disk, struct nvmeibc_channel *ch, u32 version)
{
    // Called when RDDA channel completes I/O
    
    if (info->tot_io_pending) {
        // Get highest priority pending I/O command
        for (i = DISK_PEND_PRIO_IO_START; i < DISK_PEND_PRIO_MAX; i++) {
            if ((disk_cmd = list_first_entry_or_null(
                    &info->pending_disk_cmds[i], ...))) {
                block_cmd = disk_to_block(disk_cmd);
                list_del_init(&disk_cmd->dcmd_link);
                info->tot_pending--;
                info->tot_io_pending--;
                return block_cmd;
            }
        }
    }
    
    // No pending - mark channel as available
    ioch->state = IO_CHANNEL_ALLOCATED;
    list_add(&ioch->base.link, &info->available_channels);
    return NULL;
}
```

#### NoRDDA Channel Completion

```c
struct nvmeibc_disk_command *nvmeibc_disk_get_disk_cmd_nordda(
    struct nvmeibc_disk *disk, struct nvmeibc_ib_nordda_channel *ch,
    u64 version, ...)
{
    // Called when NoRDDA channel completes I/O
    
    if (info->tot_pending) {
        // Get highest priority pending command (any type)
        for (i = 0; i < DISK_PEND_PRIO_MAX; i++) {
            if ((disk_cmd = list_first_entry_or_null(
                    &info->pending_disk_cmds[i], ...))) {
                list_del_init(&disk_cmd->dcmd_link);
                info->tot_pending--;
                // ... adjust counters ...
                return disk_cmd;
            }
        }
    }
    
    // No pending - release channel context
    (*put_req_fn)(ch, req);
    return NULL;
}
```

### Priority System Benefits

- **Recovery I/O**: Highest priority ensures quick recovery from errors
- **EC Journal Writes**: Protect data integrity
- **Write Prioritization**: Prevents read amplification in write-heavy workloads
- **Lock Operations**: Reduce lock hold time
- **General Commands**: Lowest priority, don't block data path

## Channel Lifecycle

### RDDA Channel States

```c
enum ib_io_channel_state {
    IO_CHANNEL_ALLOCATED,      // Available for use
    IO_CHANNEL_IN_USE,         // Actively executing I/O
    IO_CHANNEL_ERROR,          // Error occurred, needs cleanup
    IO_CHANNEL_CLOSING,        // Being disconnected
};
```

### Channel Availability Lists

#### RDDA Channels

```c
info->available_channels;  // List of available RDDA channels
```

Channel added to list when:
- Discovery completes successfully
- I/O completes and no pending commands
- Connection reestablished

Channel removed from list when:
- Selected for I/O execution
- Channel dies/errors
- Disk release initiated

#### NoRDDA Channels

```c
info->available_norddas;  // List of available NoRDDA channels
```

Management:
- Added during discovery: `nvmeibc_disk_available_norddas_add()`
- Removed on error/disconnect: `nvmeibc_disk_available_norddas_del()`
- Rotated on each use for load balancing

## Resource Management

### RDDA Resources

```c
struct nvmeibc_disk_info {
    int n_rscs;             // Total resources from server
    int max_client_rscs;     // Max this client can use
    int mine;               // Currently available to this client
};
```

**Resource lifecycle**:
1. **Request**: During discovery, request resources from server
2. **Grant**: Server grants some number ≤ `max_client_rscs`
3. **Use**: `info->mine` decreases when channel used
4. **Return**: `info->mine` increases when I/O completes

### NoRDDA Context Management

```c
struct nvmeibc_ib_nordda_channel {
    struct nvmeibc_volume_req_info *reqs;  // Pool of request contexts
    int n_req_infos;                       // Size of pool
    int n_used_reqs;                       // Currently in use
};
```

Each NoRDDA channel has a fixed pool of request contexts. When `n_used_reqs == n_req_infos`, the channel can't accept more I/O.

## Channel Selection for Different Command Types

### I/O Commands

```c
if (disk_cmd->cmd_type == NVMEIBC_DISK_CMD_IO &&
    !disk_cmd->server_side_only) {
    // Can use RDDA or NoRDDA
    // Prefer RDDA if available
}
```

### Server-Side Only Commands

```c
if (disk_cmd->server_side_only) {
    // Must use NoRDDA
    // Examples: MD_READ, WRITE_UNCOR
    info->n_use_nrch_only++;
}
```

### General Commands

```c
if (disk_cmd->cmd_type == NVMEIBC_DISK_CMD_GEN) {
    // Must use NoRDDA
    info->n_use_nrch_only++;
}
```

### Lock Commands

```c
if (disk_cmd->cmd_type == NVMEIBC_DISK_CMD_LOCK) {
    // Must use NoRDDA (lock channel)
}
```

## Load Balancing

### RDDA Channel Rotation

```c
if (ioch->base.cnt_io_ok == 0) {
    // First I/O on this channel - add to tail
    list_add_tail(&ioch->base.link, &info->available_channels);
} else {
    // Subsequent I/O - add to head for hot-channel optimization
    list_add(&ioch->base.link, &info->available_channels);
}
```

**Strategy**: Keep recently-used channels hot (cache affinity).

### NoRDDA Channel Rotation

```c
// After selecting NoRDDA channel
list_del(&nrch->available_link);
list_add_tail(&nrch->available_link, &info->available_norddas);
```

**Strategy**: Round-robin distribution across channels.

### Least-Used Selection (Optional)

```c
if (nvmeibc_nr_get_least_used) {
    struct nvmeibc_ib_nordda_channel *min_ch = ...; 
    list_for_each_entry(nrch, &info->available_norddas, available_link) {
        if (nrch->n_used_reqs < min_ch->n_used_reqs)
            min_ch = nrch;
    }
    // Use min_ch
}
```

Module parameter to select channel with fewest active requests.

## Channel Draining

When a disk needs to release (rediscovery, configuration change):

```c
static void drop_old_execute_pending(struct nvmeibc_disk *disk)
{
    // 1. Stop accepting new I/O
    atomic_set(&disk->paused, 1);
    
    // 2. Wait for in-flight I/O to complete
    // Channels call execute_pending_io() as they complete
    
    // 3. Abort remaining pending commands
    pending_cmds_abort(disk, ...);
}
```

## Channel Version Tracking

```c
struct nvmeibc_channel {
    u64 version;           // Incremented on each connect
    bool version_valid;    // False when disconnected
};
```

**Purpose**: Detect stale channel references

**Usage**:
```c
if (version && (!ch->version_valid || ch->version != saved_version)) {
    // Channel was disconnected and reconnected
    // Must get fresh channel
}
```

## Module Parameters

```bash
# Maximum NoRDDA channels per disk
nr_max_channels_per_disk=64

# Use NoRDDA for I/O
use_norrda_for_io=true

# Use ONLY NoRDDA (disable RDDA)
use_only_norrda_for_io=false

# Prioritize pending queue by command type
disk_prio_pending=true

# NoRDDA: rotate channel list when processing pending
nr_rotate_in_pending=false

# NoRDDA: select least-used channel
nr_get_least_used=false
```

## Debugging

### Check Available Channels

```bash
cat /proc/nvmesh/disks/<disk>/status | grep -A20 "Available Channels"
```

### Check Pending Queue

```bash
cat /proc/nvmesh/disks/<disk>/status | grep -E "tot_pending|tot_io_pending"
```

### Monitor Channel Selection

Enable tracing:
```bash
echo 1 > /sys/kernel/debug/tracing/events/nvmesh/nvmeibc_disk_get_channel/enable
cat /sys/kernel/debug/tracing/trace_pipe
```

## Common Issues

1. **High pending queue depth**: Not enough channels or channels too slow
   - **Solution**: Increase `nr_max_channels_per_disk`
   - **Solution**: Check network/target performance

2. **Channel reuse failures**: Frequent reuse validation failures
   - **Check**: Channel stability (disconnects)
   - **Check**: Per-CPU affinity settings

3. **Watermark thrashing**: Switching between RDDA and NoRDDA too often
   - **Tune**: Adjust `NVMEIBC_WATERMARK_GOTO_NORDDA`
   - **Solution**: Use NoRDDA exclusively if RDDA resources limited

## Related Documentation

- `PER_CPU_NRCH.md`: Per-CPU NoRDDA channel management
- `COREMASK_SUPPORT.md`: Coremask-aware channel selection
- `REUSED_BB_RELEASE.md`: Channel reuse and buffer release
- `RESOURCE_MANAGEMENT.md`: RDDA resource lifecycle

## Related Files

- `nvmeibc_ib_io_channel.c`: RDDA channel implementation
- `nvmeibc_ib_nordda_channel.c`: NoRDDA channel implementation
- `nvmeibc_locks_channel.c`: Lock channel implementation

