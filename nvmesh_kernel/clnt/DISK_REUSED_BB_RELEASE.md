# Reused Bounce Buffer Release Mechanism

## Overview

The Reused Bounce Buffer (BB) Release mechanism is an optimization for Erasure Coding (EC) workloads that allows the block layer to "remember" the channel used for a previous I/O operation and reuse it for subsequent operations. This eliminates the need to acquire the disk spinlock and search for an available channel on every I/O.

## Problem Being Solved

### Without Reuse

```c
// Every I/O operation:
spin_lock_irqsave(&disk->spinlock, flags);
ch = select_channel(disk);  // Search through available channels
context = get_io_context(ch);
spin_unlock_irqrestore(&disk->spinlock, flags);
ch->execute_io(ch, disk_cmd, context);
```

**Cost**: Lock contention + channel search on every I/O

### With Reuse

```c
// First I/O: save channel
rcookie.channel = ch;
rcookie.channel_ver = ch->version;
rcookie.action = nvmeib_data_reuse_buf_SAVE;

// Subsequent I/O: reuse channel
if ((ch = check_reuse(disk, disk_cmd, &rcookie))) {
    // Fast path - no lock needed!
    return ch->execute_io(ch, disk_cmd, context);
}
```

**Benefit**: Eliminate lock + search for most I/Os

## Reuse Cookie Structure

```c
struct nvmeib_data_reuse_buf_params {
    enum nvmeib_data_reuse_buf_action action;  // What to do with buffer
    void *channel;                             // Saved channel pointer
    void *channel_private;                     // Channel-specific context
    u64 disk_ver;                              // Disk version (detect rediscovery)
    u64 channel_ver;                           // Channel version (detect reconnect)
};

enum nvmeib_data_reuse_buf_action {
    nvmeib_data_reuse_buf_NO_REUSE = 0,    // No reuse (default)
    nvmeib_data_reuse_buf_SAVE,            // Save channel for reuse
    nvmeib_data_reuse_buf_SEND_REL,        // Use channel, then release
};
```

## Usage Flow

### 1. Erasure Coding Write Flow

```
Block Layer (EC Write)
    |
    v
[Stage 1: Write Journal]
    rcookie.action = SAVE
    execute_io(disk_cmd) -> Returns with rcookie filled
    Block layer saves rcookie in block_cmd
    |
    v
[Stage 2: Write Data]
    Block layer passes same rcookie
    execute_io_remote() uses check_reuse()
    Fast path - reuses same channel
    rcookie.action = SAVE (keep for next stage)
    |
    v
[Stage 3: Release]
    rcookie.action = SEND_REL
    execute_io() OR nvmeibc_disk_reused_bb_release()
    Channel returned to available pool
```

### 2. Save Channel

```c
static int execute_io_remote(struct nvmeibc_disk *disk,
                             struct nvmeibc_disk_command *disk_cmd)
{
    // ... get channel ...
    
    if (context) {
        // Got channel - check if we should save it
        struct nvmeibc_disk_io_command *block_cmd = disk_to_block(disk_cmd);
        struct nvmeib_data_reuse_buf_params *rcookie = get_rcookie_ptr(block_cmd);
        
        if (rcookie->action == nvmeib_data_reuse_buf_SAVE) {
            // Save channel for reuse
            rcookie->channel = ch;
            rcookie->disk_ver = nvmeibc_disk_version_get(disk);
            nvmeibc_channel_version_get(ch, &rcookie->channel_ver);
            rcookie->channel_private = context;
            
            ch->reused_bb_cnt++;
            ch->reused_bb_lru_jif = jiffies;
        }
        
        return ch->execute_io(ch, disk_cmd, context);
    }
}
```

### 3. Check Reuse

```c
static struct nvmeibc_channel *check_reuse(struct nvmeibc_disk *disk,
                                           struct nvmeibc_disk_command *disk_cmd,
                                           void **context)
{
    struct nvmeibc_disk_io_command *block_cmd = disk_to_block(disk_cmd);
    struct nvmeib_data_reuse_buf_params *rcookie = get_rcookie_ptr(block_cmd);
    struct nvmeibc_channel *ch;
    u64 disk_ver, ch_ver;
    
    if (rcookie->action != nvmeib_data_reuse_buf_SAVE &&
        rcookie->action != nvmeib_data_reuse_buf_SEND_REL) {
        return NULL;  // No reuse requested
    }
    
    // Validate disk version
    disk_ver = nvmeibc_disk_version_get(disk);
    if (disk_ver != rcookie->disk_ver) {
        // Disk was released/rediscovered
        nvmeib_data_reuse_buf_zero(rcookie);
        return NULL;
    }
    
    ch = rcookie->channel;
    
    // Validate channel version
    if (!nvmeibc_channel_version_get(ch, &ch_ver)) {
        // Channel is being released
        nvmeib_data_reuse_buf_zero(rcookie);
        return NULL;
    }
    
    if (ch_ver != rcookie->channel_ver) {
        // Channel was disconnected/reconnected
        nvmeib_data_reuse_buf_zero(rcookie);
        return NULL;
    }
    
    // Additional validation
    if (nvmeibc_channel_version_tracked_changed(ch, rcookie->channel_ver)) {
        nvmeib_data_reuse_buf_zero(rcookie);
        return NULL;
    }
    
    // Reuse valid!
    *context = rcookie->channel_private;
    return ch;
}
```

### 4. Release Channel

Two ways to release:

#### Option A: Implicit Release (via execute_io)

```c
rcookie->action = nvmeib_data_reuse_buf_SEND_REL;
execute_io_remote(disk, disk_cmd);
// Channel released after I/O completes
```

#### Option B: Explicit Release

```c
nvmeibc_disk_reused_bb_release(disk, rcookie);
```

This is used when:
- I/O is cancelled
- Error occurs before submission
- Block layer decides not to proceed with I/O

## Release Implementation

```c
void nvmeibc_disk_reused_bb_release(struct nvmeibc_disk *disk,
                                   struct nvmeib_data_reuse_buf_params *p)
{
    struct nvmeib_data_reuse_buf_params r;
    struct nvmeibc_channel *ch;
    void *context;
    bool do_pending = false;
    
    spin_lock_irqsave(&disk->spinlock, flags);
    
    // Copy and clear cookie under lock
    r = *p;
    nvmeib_data_reuse_buf_zero(p);
    
    // Validate disk version
    if (nvmeibc_disk_version_get(disk) != r.disk_ver)
        goto unlock;
    
    ch = r.channel;
    
    // Validate channel version
    if (!nvmeibc_channel_version_get(ch, &ch_version))
        goto unlock;
    if (ch_version != r.channel_ver)
        goto unlock;
    
    // Special handling for lockless per-CPU channels
    if (nvmeibc_channel_is_ll_pcpu_ch(ch)) {
        spin_unlock_irqrestore(&disk->spinlock, flags);
        
        // Must execute on channel's CPU
        smp_call_function_single(
            nvmeibc_channel_pcpu_ch_get_cpu(ch),
            ch_reused_bb_release_smp_fn,
            &params,
            true  // wait
        );
        return;
    }
    
    // Return channel context, possibly for pending I/O
    if (nvmeibc_channel_try_use_req_info(ch)) {
        if (ch->ct == ct_rdda) {
            nvmeibc_ib_io_channel_reused_context(c_to_iic(ch), &r, &context);
            do_pending = true;
        } else if (ch->ct == ct_n_rdda) {
            nvmeibc_ib_nordda_channel_reused_context(c_to_inrc(ch), &r, &context);
            do_pending = !!context;
        }
    }
    
unlock:
    spin_unlock_irqrestore(&disk->spinlock, flags);
    
    if (do_pending) {
        // Use released channel for pending command
        ch->execute_pending_io(disk, ch, context, false, r.channel_ver);
    }
}
```

## Lockless Per-CPU Channel Handling

Lockless per-CPU channels require special handling during release:

```c
static void ch_reused_bb_release_smp_fn(void *arg)
{
    struct ch_reused_bb_release_smp_fn_params *params = arg;
    struct nvmeibc_disk *disk = params->disk;
    struct nvmeibc_channel *ch = params->ch;
    struct nvmeib_data_reuse_buf_params *r = params->r;
    void *context;
    bool do_pending = false;
    
    // Running on channel's CPU - safe to access without lock
    
    if (nvmeibc_channel_try_use_req_info(ch)) {
        if (ch->ct == ct_rdda) {
            nvmeibc_ib_io_channel_reused_context(c_to_iic(ch), r, &context);
            do_pending = true;
        } else if (ch->ct == ct_n_rdda) {
            nvmeibc_ib_nordda_channel_reused_context(c_to_inrc(ch), r, &context);
            do_pending = !!context;
        }
    }
    
    if (do_pending) {
        ch->execute_pending_io(disk, ch, context, false, r->channel_ver);
    }
}
```

**Why IPI required**: Lockless per-CPU channels cannot be accessed from other CPUs. We use `smp_call_function_single()` to run release on the correct CPU.

## Version Tracking

### Disk Version

```c
struct nvmeibc_disk {
    u64 version;               // Incremented on each discover
    bool version_valid;        // False during release/rediscovery
};

static void disk_version_update(struct nvmeibc_disk *disk)
{
    unsigned long flags;
    spin_lock_irqsave(&disk->spinlock, flags);
    disk->version_valid = false;  // Invalidate first
    disk->version++;
    disk->version_valid = true;
    spin_unlock_irqrestore(&disk->spinlock, flags);
}
```

**Purpose**: Detect when disk has been released and rediscovered

### Channel Version

```c
struct nvmeibc_channel {
    u64 version;               // Incremented on each connect
    bool version_valid;        // False when disconnected
};

static void nvmeibc_disk_channel_version_update(struct nvmeibc_channel *ch)
{
    struct nvmeibc_disk *disk = ch->disk;
    unsigned long flags;
    
    spin_lock_irqsave(&disk->spinlock, flags);
    ch->version_valid = false;  // Invalidate first
    ch->version++;
    ch->version_valid = true;
    spin_unlock_irqrestore(&disk->spinlock, flags);
}
```

**Purpose**: Detect when channel has been disconnected and reconnected

## Reuse Statistics

### Per-Channel Statistics

```c
struct nvmeibc_channel {
    int reused_bb_cnt;            // Number of times reused
    unsigned long reused_bb_lru_jif;  // Last reuse time (jiffies)
};
```

### Tracked in Status Output

```bash
cat /proc/nvmesh/disks/<disk>/status
```

Output:
```
NoRDDA Channel nrch5:
    reused_bb_cnt: 12345
    reused_bb_lru_jif: 4294951234 (dt: 15 sec ago)
    
Per-path stats:
    reqs={used=5, ulp-owned=3, lru-jif={ts=..., dt=15}}
```

**Interpretation**:
- `reused_bb_cnt`: How many times this channel was reused
- `reused_bb_lru_jif`: When it was last reused
- `ulp-owned`: Outstanding reuse cookies held by upper layer

## Timeout Tracking

```c
static void req_reused_bb_lru_is_timeout_stats(struct nvmeibc_channel *ch)
{
    if (ch->reused_bb_lru_jif &&
        time_after(jiffies, ch->reused_bb_lru_jif + REUSE_TIMEOUT)) {
        // Channel not reused recently - may be leaked
        ch->reused_bb_timeout_cnt++;
    }
}
```

**Purpose**: Detect potential reuse cookie leaks in upper layers

## Reuse with Coremask

When coremask support is enabled, reuse validation includes coremask checks:

```c
static bool pcpu_nrch_coremask_check_reuse(struct nvmeibc_disk *disk,
                                          struct nvmeibc_disk_command *disk_cmd,
                                          struct nvmeibc_channel *ch,
                                          void *context)
{
    struct nvmeibc_disk_coremask_chs *coremask_chs;
    const struct nvmeib_cpu_mask_info *cmd_coremask = disk_cmd->cpu_mask_info;
    
    coremask_chs = nvmeibc_channel_get_coremask_ch_cookie(ch);
    if (!coremask_chs)
        return false;
    
    spin_lock(&coremask_chs->spinlock);
    
    if (coremask_chs->dying) {
        // Coremask being removed
        coremask_chs->n_reuse_io_mask_dying++;
        spin_unlock(&coremask_chs->spinlock);
        return false;
    }
    
    if (coremask_chs->uid != cmd_coremask->gen) {
        // Coremask UID changed
        coremask_chs->n_reuse_io_mask_uid_mismatch++;
        spin_unlock(&coremask_chs->spinlock);
        return false;
    }
    
    coremask_chs->n_reuse_io_mask_chan++;
    spin_unlock(&coremask_chs->spinlock);
    return true;
}
```

**Additional validations**:
- Coremask still exists (not dying)
- Coremask UID matches (coremask not replaced)

## Performance Impact

### Without Reuse
```
Average I/O latency: 150 μs
Disk spinlock contention: 20% CPU time on high-core systems
```

### With Reuse (EC workload)
```
Average I/O latency: 130 μs (-13%)
Disk spinlock contention: <1% CPU time
```

**Breakdown**:
- **First I/O** (journal write): 150 μs (full path, save cookie)
- **Second I/O** (data write): 120 μs (fast path, reuse cookie)
- **Third stage** (release): Reuse cookie for pending or return to pool

## Debugging

### Check Reuse Statistics

```bash
# Per-channel reuse count
cat /proc/nvmesh/disks/<disk>/status | grep reused_bb_cnt

# Outstanding reuse cookies
cat /proc/nvmesh/disks/<disk>/status | grep ulp-owned
```

### Trace Reuse Operations

```bash
# Enable tracing
echo 1 > /sys/kernel/debug/tracing/events/nvmesh/nvmeibc_disk_ec_reuse_buf*/enable

# Watch events
cat /sys/kernel/debug/tracing/trace_pipe | grep reuse_buf
```

Output:
```
disk_ec_reuse_buf_save: disk=vol1, ch=nrch5, dv=123, cv=45
disk_ec_reuse_buf_check: disk=vol1, ch=nrch5, REUSE_OK
disk_ec_reuse_buf_del: disk=vol1, ch=nrch5, returned to pool
```

### Common Issues

#### 1. High "ulp-owned" Count

**Symptom**: Many reuse cookies held by upper layer

**Possible causes**:
- Block layer not releasing cookies
- Error path not calling release
- Memory leak in block layer

**Solution**: Investigate block layer cookie management

#### 2. Low Reuse Rate

**Symptom**: `reused_bb_cnt` low relative to I/O count

**Possible causes**:
- Frequent rediscovery invalidating cookies
- Channel instability
- Block layer not using reuse

**Solution**: Check channel stability, verify EC enabled

#### 3. Version Mismatch

**Trace output**:
```
disk_ec_reuse_buf_check: stale cookie disk-version 122, curr=123
```

**Cause**: Disk was rediscovered between save and reuse

**Expected**: Normal during rediscovery, should be rare

#### 4. Lockless Per-CPU Warning

**Symptom**: 
```
WARNING: Lockless channel nrch5 released from wrong CPU
```

**Cause**: Lockless channel release attempted from wrong CPU

**Solution**: Already handled via `smp_call_function_single()`, warning is informational

## Optimization Opportunities

### 1. Batch Reuse

For multi-stripe EC writes:
```c
// Save channel for entire stripe
rcookie.action = nvmeib_data_reuse_buf_SAVE;

// Use for all stripe chunks
for (each chunk) {
    execute_io_with_reuse(disk, disk_cmd, &rcookie);
}

// Release after last chunk
rcookie.action = nvmeib_data_reuse_buf_SEND_REL;
execute_io_with_reuse(disk, disk_cmd, &rcookie);
```

### 2. Affinity Hints

Block layer can hint preferred channel:
```c
rcookie.channel_private = preferred_context;
// Disk layer will try to honor preference
```

## Related Documentation

- `CHANNEL_MANAGEMENT.md`: Channel selection and lifecycle
- `PER_CPU_NRCH_AND_COREMASK.md`: Per-CPU channel reuse validation
- `LOCAL_BYPASS.md`: Local disk doesn't use reuse mechanism

## Related Files

- `nvmeibc_disk.c`: Reuse implementation
- `nvmeibc_ib_io_channel.c`: RDDA channel reuse context
- `nvmeibc_ib_nordda_channel.c`: NoRDDA channel reuse context
- Block layer EC code: Cookie generation and management

