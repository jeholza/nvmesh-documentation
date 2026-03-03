# Disk Release Flows

## Overview

Disk release is the process of cleanly tearing down a disk's network connections, channels, and resources, typically followed by rediscovery to re-establish connectivity. This document describes all paths that can trigger disk release and the detailed flow through various contexts and workqueues.

### Key Concepts

- **Release**: Graceful teardown of disk resources and channels
- **Rediscovery**: Attempt to reconnect to the disk after release
- **Restart Loop**: Mechanism to handle concurrent release requests
- **Dying Flag**: `disk->dying` atomic counter preventing concurrent releases
- **Restart Called**: `disk->restart_called` atomic counter coordinating release work scheduling

## Release Triggers

### Release Reasons Enumeration

```c
enum nvmeibc_disk_release_reason {
    // Direct triggers
    NVMEIBC_DISK_RELEASE_UNKNOWN,
    NVMEIBC_DISK_RELEASE_OK,
    NVMEIBC_DISK_RELEASE_REDISCOVER_CALLED,         // Manual rediscovery request
    NVMEIBC_DISK_RELEASE_DISK_REMOVE,               // Volume detachment
    
    // Channel disconnections
    NVMEIBC_DISK_RELEASE_LOCK_CHAN_DISCONNECT,      // Lock channel disconnected
    NVMEIBC_DISK_RELEASE_LOCK_CHAN_WD_EVENT,        // Lock channel watchdog timeout
    NVMEIBC_DISK_RELEASE_ADMIN_CHAN_DISCONNECT,     // Admin channel disconnected
    NVMEIBC_DISK_RELEASE_ADMIN_CHAN_KA_FAILED,      // Admin keepalive failed
    
    // I/O channel failures
    NVMEIBC_DISK_RELEASE_REM_DRV_ERR,               // Remote driver error
    NVMEIBC_DISK_RELEASE_IOCH_DRAINED_EVENT,        // I/O channel drained
    NVMEIBC_DISK_RELEASE_RETURN_RCOOKIE,            // Resource cookie return failed
    NVMEIBC_DISK_RELEASE_COMMON_PREFIX_PRIORITY_NEW, // Priority channel creation
    NVMEIBC_DISK_RELEASE_COREMASK_LOCK_CHAN_EXHAUSTED, // Coremask lock channels full
    
    // Local operations
    NVMEIBC_DISK_RELEASE_LOCAL_IO_FAILED,           // Local I/O failure
    
    // Configuration updates
    NVMEIBC_DISK_RELEASE_CONFIG_UPDATE_LOCAL_SRV,   // Local server config change
    NVMEIBC_DISK_RELEASE_CONFIG_UPDATE_NIC_REMOVE,  // NIC removed
    NVMEIBC_DISK_RELEASE_CONFIG_UPDATE_PORT_REMOVE, // Port removed
    NVMEIBC_DISK_RELEASE_CONFIG_UPDATE_PORT_UPDATE, // Port updated
    NVMEIBC_DISK_RELEASE_CONFIG_UPDATE_COMPLETE,    // Config update completed
    
    // TOMA failures
    NVMEIBC_DISK_RELEASE_TOMA_SUBSCRIBE_FAILURE,    // TOMA subscription failed
    
    // JAM/Journal issues
    NVMEIBC_DISK_RELEASE_JOURNAL_ENTRY_ERASE_FAILURE, // Journal erase failed
    NVMEIBC_DISK_RELEASE_DISK_ABANDONED_JENTRY,     // Abandoned journal entry
    NVMEIBC_DISK_RELEASE_GET_REQUESTED_BINJE,       // Get requested binje
    
    // State management
    NVMEIBC_DISK_RELEASE_DISK_PAUSED,               // Disk manually paused
    NVMEIBC_DISK_RELEASE_STARTED_OFFLINE,           // Disk started offline
    
    // Server-initiated (LOGOUT messages)
    NVMEIBC_DISK_RELEASE_SRV_LOGOUT_*,              // Various server logout reasons
};
```

### Primary Trigger Points

1. **Channel Disconnection Callbacks**
   - Lock channel: `on_disconnect_lock_ch()`
   - Admin channel: `on_disconnect_admin_ch()`
   - I/O channel errors

2. **Direct Requests**
   - Volume detach
   - Manual rediscovery
   - Configuration updates

3. **Error Conditions**
   - I/O failures
   - JAM errors
   - TOMA failures

## Core Release Function

### Entry Point: `nvmeibc_disk_start_release()`

```c
int nvmeibc_disk_start_release(struct nvmeibc_disk *disk, 
                               enum nvmeibc_disk_release_reason reason)
{
    struct disk_workq *rwork;
    int restart_called;
    unsigned long flags;
    
    // Allocate work structure
    rwork = kzalloc(sizeof(*rwork), GFP_ATOMIC);
    
    // Critical section: coordinate release work scheduling
    spin_lock_irqsave(&disk->restart_lock, flags);
retry:
    // Atomic increment - only first caller proceeds
    if ((restart_called = atomic_inc_return(&disk->restart_called)) == 1) {
        // First caller - schedule release work
        WQ_INIT_WORK(&rwork->work, nvmeibc_disk_release_work);
        rwork->disk = disk;
        rwork->work_data = (void *)reason;
        atomic_inc(&disk->shut_down_triggered);
        disk->drw_cnt++;
        
        // Add to disk's workqueue
        rv = nvmeibc_disk_add_work(disk, &rwork->work);
        if (rv < 0) {
            // Failed to add - reset and retry
            atomic_set(&disk->restart_called, 0);
            goto retry;
        }
    }
    else {
        // Concurrent call - release already in progress
        // The running release will detect this via restart_called counter
        kfree(rwork);
    }
    spin_unlock_irqrestore(&disk->restart_lock, flags);
    
    return rv;
}
```

**Key Points**:
- `GFP_ATOMIC` allocation (can be called from softirq context)
- `restart_lock` spinlock protects work scheduling
- `restart_called` atomic counter prevents duplicate scheduling
- Multiple concurrent calls are safe - only first schedules work
- Failed work add triggers immediate retry

### Main Release Function: `nvmeibc_disk_release()`

```c
static void nvmeibc_disk_release(struct nvmeibc_disk *disk, 
                                 enum nvmeibc_disk_release_reason reason)
{
    int dying;
    enum rediscovery rv;
    int restart_called;
    bool disk_used;
    u64 ts, dt;
    
    disk->restart_num = 0;
    nvmeib_trend_insert(&disk->release_trend, reason);
    
restart:
    disk_toma_subscribe_freeze(disk);
    disk_version_update(disk);  // Inc version, prevents stale I/O
    disk->restart_num++;
    
    // Check dying flag - only first release proceeds
    if ((dying = atomic_inc_return(&disk->dying)) > 1) {
        // Already releasing - exit
        goto out;
    }
    
    /* === STEP 1: Remove Periodic Operations === */
    if (disk->periodic.remove_periodic && disk->periodic.ch) {
        disk->periodic.remove_periodic(disk->periodic.ch, &disk->periodic);
        memset(&disk->periodic, 0, sizeof(disk->periodic));
    }
    
    /* === STEP 2: Stop Disk I/O === */
    disk->io_stopped = stop_disk_io(disk, &disk_used);
    
    /* === STEP 3: Disconnect Disk === */
    disconnect_disk(disk);
    
    /* === STEP 4: Verify I/O Stopped === */
    if (!disk->io_stopped) {
        if (try_wait_for_completion(&disk->disk_paused)) {
            disk->io_stopped = true;
        } else {
            // Block layer did not pause!
            nvmeibc_pd_dump_transfers(disk);
        }
    }
    
    /* === STEP 5: Determine Rediscovery Eligibility === */
    if (disk_used && disk->io_stopped) {
        nvmeib_reinit_completion(&disk->disk_paused);
        disk->allow_rediscovery = true;
    }
    else {
        disk->allow_rediscovery = false;
    }
    
    /* === STEP 6: Clear Local Data === */
    if (disk->io_stopped) {
        disk->local.p = NULL;
    }
    
    /* === STEP 7: Free Disk Resources === */
    free_disk_rsc(disk);
    
    /* === STEP 8: Attempt Rediscovery === */
    atomic_set(&disk->restart_called, 1);
    rv = rediscovery(disk);
    
    switch (rv) {
    case rr_try_again:
        // Rediscovery failed, restart the whole process
        goto restart;
        
    case rr_ok:
        // Rediscovery succeeded
        if ((restart_called = atomic_dec_return(&disk->restart_called)) > 0) {
            // Another release was requested during rediscovery
            goto restart;
        }
        else {
            disk_toma_subscribe_unfreeze(disk);
            goto done;
        }
        
    case rr_out:
        // Rediscovery failed permanently
        goto out;
        
    case rr_cfg_update:
        // Config update in progress, rediscovery rescheduled
        goto out;
        
    default:
        goto out;
    }
    
out:
    atomic_set(&disk->restart_called, 0);
done:
    return;
}
```

## Detailed Release Steps

### Step 1: Stop Disk I/O

```c
static bool stop_disk_io(struct nvmeibc_disk *disk, bool *disk_used)
{
    struct nvmeibc_disk_id *disk_id;
    unsigned long flags;
    bool io_stopped = false;
    unsigned long wait_pause_timeout;
    
    // Mark disk as pausing
    spin_lock_irqsave(&disk->volume_spinlock, flags);
    disk->should_pause = true;
    atomic_inc(&disk->paused);
    atomic_set(&disk->n_volumes_paused, 1);
    *disk_used = !list_empty(&disk->volumes);
    
    // Pause all volumes using this disk
    list_for_each_entry(disk_id, &disk->volumes, slink) {
        if (disk_id->volume) {
            nvmeibc_volume_get(disk_id->volume, NULL);
            is_ready = nvmeibc_volume_is_ready_for_pause_cont(disk_id->volume);
            if (is_ready > 0)
                nvmeibc_block_pause(disk_id->volume->block_dev, disk);
            nvmeibc_volume_put(disk_id->volume, NULL);
        }
    }
    spin_unlock_irqrestore(&disk->volume_spinlock, flags);
    
    // Wait for block layer to acknowledge pause
    for (i = 0; i < iterations && !io_stopped; i++) {
        rv = wait_for_completion_interruptible_timeout(&disk->disk_paused,
                                                       NVMEIB_WAIT_BLOCK_DEV_PAUSE);
        if (rv > 0) {
            io_stopped = true;
        }
    }
    
    return io_stopped;
}
```

**What it does**:
1. Sets `disk->should_pause` flag
2. Increments `disk->paused` atomic
3. Calls `nvmeibc_block_pause()` for each volume
4. Waits for `disk->disk_paused` completion (signaled by block layer)
5. Timeout: `nvmeibc_disk_pause_timeout` seconds

**Block Layer Response**:
- Block layer drains in-flight I/O
- Calls `complete(&disk->disk_paused)` when done
- Prevents new I/O submission

### Step 2: Disconnect Disk

```c
static void disconnect_disk(struct nvmeibc_disk *disk)
{
    struct nvmeibc_admin_rnic *arnic;
    struct nvmeibc_disk_segments_locks *disk_seg_locks;
    
    /* === 2.1: Disconnect Admin Channels === */
    list_for_each_entry(arnic, &disk->arnics, link) {
        if (arnic->alive || nvmeibc_ib_admin_is_connected(ac_to_iac(arnic->channel))) {
            nvmeibc_ib_admin_channel_disconnect(ac_to_iac(arnic->channel));
            if (arnic->alive && arnic->channel->is_main) {
                nvmeibc_target_arnic_close_conn(arnic);
                disk->main_ach_wq_pid = 0;
            }
        }
        if (arnic->channel) {
            nvmeibc_ib_admin_channel_free(ac_to_iac(arnic->channel));
            arnic->alive = false;
            if (arnic->local) {
                disk->local_admin_ch = NULL;
            }
        }
    }
    
    /* === 2.2: Verify No Lock Channel References === */
    disk_seg_locks = nvmeibc_disk_get_segs_locks(disk, ...);
    if (disk_seg_locks && disk_seg_locks->lock_ch) {
        // Should not happen - lock channel still referenced!
        BUG();
    }
    nvmeibc_disk_put_segs_locks(disk_seg_locks, ...);
    
    /* === 2.3: Disconnect All Lock Channels === */
    nvmeibc_locks_channel_disconnect_all_handles(disk);
    
    /* === 2.4: Destroy JAM Disk === */
    if (disk->jam_disk) {
        // Cache state before destroying
        nvmeibc_jam_disk_del(disk);
    }
    disk->jour.rng_id = NVMEIB_EC_INVALID_JOURNAL_RANGE;
    
    /* === 2.5: Free Local Resources === */
    if (disk->access_local) {
        if (disk->local.dma_pools) {
            free_percpu_pools(disk, disk->local.dma_pools);
            disk->local.dma_pools = NULL;
        }
        // Free dummy MD buffers...
        disk->local.dummy_md_dma_dev = NULL;
    }
    
    /* === 2.6: Unregister Local Disk === */
    if (disk->is_cl_reg)
        local_disk_cl_unregister(disk, cid);
    
    /* === 2.7: Free Dirty Bits Memory === */
    free_dirty_bits_mem(disk);
    
    /* === 2.8: Verify Per-CPU NRCH Empty === */
    pcpu_nrch_check_empty(disk);
}
```

### Step 3: Free Disk Resources

```c
static void free_disk_rsc(struct nvmeibc_disk *disk)
{
    struct nvmeibc_disk_info *info;
    struct nvmeibc_admin_rnic *arnic;
    unsigned long flags;
    struct nvmeibc_disk_segments_locks *disk_segs_locks;
    
    /* === 3.1: Clear Disk Info === */
    spin_lock_irqsave(&disk->spinlock, flags);
    info = disk->info;
    disk->info = NULL;
    nvmeibc_locks_channel_disconnect_all_handles_(disk);
    spin_unlock_irqrestore(&disk->spinlock, flags);
    
    /* === 3.2: Free Discovery Resources === */
    if (info) {
        free_disc_rscs(ac_to_iac(info->ch), info);
        kfree(info->hcaa);
        
        // Free remote segment locks
        down_write(&info->ch->segments_locks_remote.guard);
        if (info->ch->segments_locks_remote.num_of_segments > 0) {
            kfree(info->ch->segments_locks_remote.locks);
            info->ch->segments_locks_remote.locks = NULL;
            info->ch->segments_locks_remote.num_of_segments = 0;
        }
        up_write(&info->ch->segments_locks_remote.guard);
        
        // Free workqueue and coremask info
        if (info->pcpu_wq)
            nvmeib_public_destroy_workqueue(info->pcpu_wq);
        free_coremask_info(info->coremask_info);
        kfree(info);
    }
    
    /* === 3.3: Free Local Segment Locks === */
    disk_segs_locks = nvmeibc_disk_get_segs_locks(disk, ...);
    if (disk_segs_locks->num_of_segments > 0) {
        kfree(disk_segs_locks->locks);
        disk_segs_locks->num_of_segments = 0;
        disk_segs_locks->locks = NULL;
    }
    nvmeibc_disk_put_segs_locks(disk_segs_locks, ...);
    
    /* === 3.4: Free ARNIC I/O Resources === */
    spin_lock_irqsave(&disk->spinlock, flags);
    list_for_each_entry(arnic, &disk->arnics, link) {
        if (!arnic->channel)
            continue;
            
        // Free RIONICs
        free_rionics(arnic->channel);
        
        // Free I/O channels
        if (!list_empty(&arnic->channel->hca->available_channels)) {
            free_iochannels(&arnic->channel->hca->available_channels);
        }
        
        // Free NoRDDA channels
        if (!list_empty(&arnic->channel->hca->available_norddas)) {
            free_nordda_channels(&arnic->channel->hca->available_norddas);
        }
    }
    spin_unlock_irqrestore(&disk->spinlock, flags);
}
```

## Release Flow 1: Direct Disk Release

### Sequence Diagram

```
User/Kernel Thread         Disk WQ                  Block Layer           Admin Channel
       |                       |                           |                      |
       |---nvmeibc_disk_start_release()                   |                      |
       |    (reason: REDISCOVER)                          |                      |
       |                       |                           |                      |
       | atomic_inc(restart_called)                       |                      |
       | alloc work            |                           |                      |
       |---add to disk WQ----->|                           |                      |
       |<---return 0-----------|                           |                      |
       |                       |                           |                      |
       |                    [Work Scheduled]               |                      |
       |                       |                           |                      |
       |              nvmeibc_disk_release_work()          |                      |
       |                       |---nvmeibc_disk_release()  |                      |
       |                       |    (reason: REDISCOVER)   |                      |
       |                       |                           |                      |
       |                       | atomic_inc(dying)         |                      |
       |                       | [dying == 1, proceed]     |                      |
       |                       |                           |                      |
       |                       |---disk_version_update()   |                      |
       |                       |                           |                      |
       |                       |---stop_disk_io()--------->|                      |
       |                       |    disk->should_pause=1   |                      |
       |                       |    atomic_inc(paused)     |                      |
       |                       |                           |                      |
       |                       |                           |---nvmeibc_block_pause()
       |                       |                           |   (for each volume)  |
       |                       |                           |                      |
       |                       | wait_for_completion_timeout()                    |
       |                       |    (&disk->disk_paused)   |                      |
       |                       |<--------------------------+                      |
       |                       |    [Block layer drains I/O]                     |
       |                       |    [complete(&disk_paused)]                     |
       |                       |                           |                      |
       |                       |<---io_stopped=true--------|                      |
       |                       |                           |                      |
       |                       |---disconnect_disk()-------|---------------------->|
       |                       |                           |    nvmeibc_ib_admin_channel_disconnect()
       |                       |                           |                      |
       |                       |                           |    [Disconnect all arnics]
       |                       |                           |                      |
       |                       |    nvmeibc_locks_channel_disconnect_all_handles()
       |                       |                           |                      |
       |                       |    nvmeibc_jam_disk_del() |                      |
       |                       |    [Cache JAM state]      |                      |
       |                       |                           |                      |
       |                       |---free_disk_rsc()---------|                      |
       |                       |    [Free all resources]   |                      |
       |                       |                           |                      |
       |                       |---rediscovery()-----------|                      |
       |                       |    [Attempt to reconnect] |                      |
       |                       |                           |                      |
       |                       | [if success:]             |                      |
       |                       | atomic_dec(dying)         |                      |
       |                       | disk_toma_subscribe_unfreeze()                   |
       |                       |                           |                      |
       |                       |<---return-----------------|                      |
       |                       |                           |                      |
       |                   [Work Complete]                 |                      |
```

### Context Summary

| Context | Operations |
|---------|-----------|
| **Caller Thread** | Allocates work, schedules on disk WQ, returns immediately |
| **Disk WQ Thread** | Executes entire release, waits for block layer, performs rediscovery |
| **Block Layer** | Drains I/O, signals completion via `complete(&disk->disk_paused)` |
| **Admin Channel WQ** | Handles channel disconnection callbacks (if triggered independently) |

### Key Synchronization Points

1. **`disk->restart_lock`**: Protects work scheduling
2. **`disk->restart_called`**: Atomic counter for release coordination
3. **`disk->dying`**: Atomic counter preventing concurrent releases
4. **`disk->disk_paused`**: Completion signaled by block layer
5. **`disk->spinlock`**: Protects disk state during resource cleanup

## Release Flow 2: Lock Channel Disconnect

### Trigger Point

```c
static void on_disconnect_lock_ch(struct nvmeibc_ib_net *net)
{
    struct nvmeibc_locks_channel *ch = lock_ch_from_net(net);
    struct nvmeibc_disk *disk = ch->base.disk;
    enum nvmeibc_disk_release_reason reason = ch->release_reason;
    int dying;
    
    if (!(dying = atomic_read(&disk->dying))) {
        // Disk not already releasing - trigger release
        nvmeibc_disk_start_release(disk, reason ? :
                                   NVMEIBC_DISK_RELEASE_LOCK_CHAN_DISCONNECT);
    }
    else {
        // Disk already releasing - skip (avoid endless loop)
    }
}
```

**Called From**:
- IB layer: CM callback when lock channel connection breaks
- QP error handler: When lock channel QP fails
- Watchdog timeout: If lock channel becomes unresponsive

**Contexts**:
- **Softirq** (completion handler)
- **Workqueue** (CM event handler)
- **Timer** (watchdog)

### Sequence Diagram

```
Lock Channel           IB Softirq/WQ          Disk WQ              Block Layer       Admin Channel
     |                       |                    |                      |                  |
     |<--QP Error--->        |                    |                      |                  |
     |                       |                    |                      |                  |
     |              nvmeibc_ib_net_handle_qp_err()|                      |                  |
     |                       |                    |                      |                  |
     |                       |---on_disconnect_lock_ch()                 |                  |
     |                       |    (reason: LOCK_CHAN_DISCONNECT)         |                  |
     |                       |                    |                      |                  |
     |                       | atomic_read(dying) |                      |                  |
     |                       | [dying == 0]       |                      |                  |
     |                       |                    |                      |                  |
     |                       |---nvmeibc_disk_start_release()            |                  |
     |                       |    (GFP_ATOMIC)    |                      |                  |
     |                       |                    |                      |                  |
     |                       | atomic_inc(restart_called)                |                  |
     |                       | alloc work (GFP_ATOMIC)                   |                  |
     |                       |---add to disk WQ-->|                      |                  |
     |                       |<---return 0--------|                      |                  |
     |                       |<---return----------|                      |                  |
     |                       |                    |                      |                  |
     |                       |                [Work Scheduled]           |                  |
     |                       |                    |                      |                  |
     |                       |            nvmeibc_disk_release_work()    |                  |
     |                       |                    |---nvmeibc_disk_release()                |
     |                       |                    |    (reason: LOCK_CHAN_DISCONNECT)       |
     |                       |                    |                      |                  |
     |                       |                    | atomic_inc(dying)    |                  |
     |                       |                    | [dying == 1, proceed]|                  |
     |                       |                    |                      |                  |
     |                       |                    |---stop_disk_io()---->|                  |
     |                       |                    |                      |                  |
     |                       |                    |<---io_stopped--------|                  |
     |                       |                    |                      |                  |
     |                       |                    |---disconnect_disk()--|----------------->|
     |                       |                    |                      |                  |
     |<--locks_rw_disconnect_ch()                 |                      |  nvmeibc_ib_admin_channel_disconnect()
     |   [Disconnect lock channel]                |                      |                  |
     |   [Abort in-progress ops]                  |                      |                  |
     |   [Break QP]                               |                      |                  |
     |   [Free resources]                         |                      |                  |
     |                       |                    |                      |                  |
     |                       |                    |---free_disk_rsc()----|                  |
     |                       |                    |                      |                  |
     |                       |                    |---rediscovery()------|                  |
     |                       |                    |    [Reconnect]       |                  |
     |                       |                    |                      |                  |
     |                       |                    |<---return------------|                  |
     |                       |                    |                      |                  |
     |                       |                [Work Complete]            |                  |
```

### Lock Channel Disconnection Details

```c
static void locks_rw_disconnect_ch(struct nvmeibc_locks_channel *ch)
{
    unsigned long flags;
    
    /* === 1: Pause Channel === */
    nvmeibc_locks_channel_spin_lock_irqsave(ch, flags);
    ch->paused = true;
    nvmeibc_locks_channel_spin_unlock_irqrestore(ch, flags);
    
    /* === 2: Handle Coremask Channels === */
    if (nvmeibc_channel_is_coremask_ch(&ch->base)) {
        // Disconnect from coremask map
        struct nvmeibc_locks_channel *primary_ch = get_primary_ch(ch);
        int cpu;
        NVMEIB_CPU_MASK_FOR_EACH_CPU(cpu, ch->_2nd_ch_coremask_mask) {
            primary_ch->_2nd_ch_coremask_map[cpu] = NULL;
        }
        // Release coremask reference
        if (ch->coremask_ops->put_ref_fn)
            (*ch->coremask_ops->put_ref_fn)(coremask_cookie);
    }
    
    /* === 3: Drain Deferred Operations === */
    nvmeibc_disk_locks_drain_defered(ch, false, true);
    
    /* === 4: Abort In-Progress Operations === */
    nvmeibc_disk_locks_abort_in_progress_oprs(ch, true);
    
    /* === 5: Disconnect Network Layer === */
    nvmeibc_ib_net_disconnect(&ch->net);
    
    /* === 6: Break QP === */
    nvmeibc_ib_net_break_qp(&ch->net);
    
    /* === 7: Free IB Resources === */
    nvmeibc_ib_net_free(&ch->net);
    
    /* === 8: Remove from List === */
    list_del_init(&ch->link);
    
    /* === 9: Free DMA Resources === */
    free_dma_resources(ch);
}
```

### Context Summary

| Context | Operations |
|---------|-----------|
| **IB Softirq/Event WQ** | Detects error, calls disconnect callback, schedules disk release |
| **Disk WQ** | Executes full disk release including lock channel cleanup |
| **Lock Channel Per-CPU WQ** | Handles per-CPU secondary channel disconnection (if enabled) |

### Why Lock Channel Triggers Disk Release

Lock channels are **critical** for correctness:
- Provide distributed locking for data consistency
- Required for all write operations in EC volumes
- Failure means data corruption risk

**Therefore**: Lock channel failure → Full disk release → Rediscovery → Re-establish all connections

## Release Flow 3: Admin Channel Disconnect

### Trigger Point

```c
static void on_disconnect_admin_ch(struct nvmeibc_ib_net *net)
{
    struct nvmeibc_ib_admin_channel *ch = in_to_iac(net);
    struct nvmeibc_disk *disk;
    int dying;
    
    // Signal admin-wq that may be waiting for send-comp
    ach_send_done_on_dying(ch);
    
    // Kill disk iff admin channel is the one that holds the disk
    if (ch->base.n_disks && ch->base.arnic->alive) {
        if ((disk = ch->base.base.disk)) {
            if (!(dying = atomic_read(&disk->dying))) {
                nvmeibc_disk_start_release(disk,
                    ch->logout ?
                        NVMEIB_TREND_HEAD(disk->peer_release_reason_trend).data
                      : ch->release_reason ? :
                          NVMEIBC_DISK_RELEASE_ADMIN_CHAN_DISCONNECT);
            }
            else {
                // Disk already releasing - skip
            }
        }
    }
}
```

**Called From**:
- IB layer: CM callback when admin channel connection breaks
- QP error handler: When admin channel QP fails
- Keep-alive failure: If admin channel becomes unresponsive
- Server LOGOUT message: Server-initiated disconnect

**Special Handling**:
- `ch->logout`: If true, use reason from server's LOGOUT message
- `ch->release_reason`: If set, use specific reason from channel
- Default: `NVMEIBC_DISK_RELEASE_ADMIN_CHAN_DISCONNECT`

### Sequence Diagram

```
Admin Channel          IB Softirq/WQ          Admin WQ             Disk WQ          Block Layer
     |                       |                    |                    |                  |
     |<--QP Error/Logout-->  |                    |                    |                  |
     |                       |                    |                    |                  |
     |              nvmeibc_ib_net_handle_qp_err()|                    |                  |
     |                       |                    |                    |                  |
     |                       |---on_disconnect_admin_ch()              |                  |
     |                       |    (reason: ADMIN_CHAN_DISCONNECT)      |                  |
     |                       |                    |                    |                  |
     |                       |---ach_send_done_on_dying()              |                  |
     |                       |    [Wake waiting admin work]            |                  |
     |                       |                    |<--wake up----------|                  |
     |                       |                    |    [Pending admin ops]                |
     |                       |                    |    [Get error returns]                |
     |                       |                    |                    |                  |
     |                       | atomic_read(dying) |                    |                  |
     |                       | [dying == 0]       |                    |                  |
     |                       |                    |                    |                  |
     |                       |---nvmeibc_disk_start_release()          |                  |
     |                       |    (GFP_ATOMIC)    |                    |                  |
     |                       |                    |                    |                  |
     |                       | atomic_inc(restart_called)              |                  |
     |                       | alloc work (GFP_ATOMIC)                 |                  |
     |                       |---add to disk WQ------------------>     |                  |
     |                       |<---return 0-----------------------------|                  |
     |                       |<---return----------|                    |                  |
     |                       |                    |                    |                  |
     |                       |                    |                [Work Scheduled]       |
     |                       |                    |                    |                  |
     |                       |                    |            nvmeibc_disk_release_work()|
     |                       |                    |                    |---nvmeibc_disk_release()
     |                       |                    |                    |    (reason: ADMIN_CHAN_DISCONNECT)
     |                       |                    |                    |                  |
     |                       |                    |                    | atomic_inc(dying)|
     |                       |                    |                    | [dying == 1]     |
     |                       |                    |                    |                  |
     |                       |                    |                    |---stop_disk_io()-->
     |                       |                    |                    |                  |
     |                       |                    |                    |<---io_stopped----|
     |                       |                    |                    |                  |
     |                       |                    |                    |---disconnect_disk()
     |                       |                    |                    |                  |
     |<--nvmeibc_ib_admin_channel_disconnect()   |                    |                  |
     |   nvmeibc_ib_admin_channel_free()         |                    |                  |
     |   [Close CM]          |                    |                    |                  |
     |   [Break QP]          |                    |                    |                  |
     |   [Free TX ring]      |                    |                    |                  |
     |   [Free RX ring]      |                    |                    |                  |
     |                       |                    |                    |                  |
     |                       |                    |   nvmeibc_locks_channel_disconnect_all()
     |                       |                    |   nvmeibc_jam_disk_del()              |
     |                       |                    |                    |                  |
     |                       |                    |                    |---free_disk_rsc()
     |                       |                    |                    |                  |
     |                       |                    |                    |---rediscovery()--|
     |                       |                    |                    |    [Attempt reconnect]
     |                       |                    |                    |                  |
     |                       |                    |                    |<---return--------|
     |                       |                    |                    |                  |
     |                       |                    |                [Work Complete]       |
```

### Admin Channel Disconnection Details

```c
void nvmeibc_ib_admin_channel_disconnect(struct nvmeibc_ib_admin_channel *ch)
{
    /* === 1: Close Connection Manager === */
    if (ch->net.cm.cm_id) {
        rdma_disconnect(ch->net.cm.cm_id);
    }
    
    /* === 2: Disconnect Network Layer === */
    nvmeibc_ib_net_disconnect(&ch->net);
    
    /* === 3: Break QP === */
    nvmeibc_ib_net_break_qp(&ch->net);
}

void nvmeibc_ib_admin_channel_free(struct nvmeibc_ib_admin_channel *ch)
{
    /* === 4: Free IB Resources === */
    nvmeibc_ib_net_free(&ch->net);
    
    /* === 5: Free TX Ring === */
    nvmeib_free_ioctx_ring(ch->tx_ring, ch->tx_ring_size);
    ch->tx_ring = NULL;
    
    /* === 6: Free RX Ring === */
    free_iu_bufs(&ch->net.base);
    
    /* === 7: Free Channel Structure === */
    // (Done later after all references released)
}
```

### Context Summary

| Context | Operations |
|---------|-----------|
| **IB Softirq/Event WQ** | Detects error, calls disconnect callback |
| **Admin WQ** | Wakes up waiting operations, returns errors |
| **Disk WQ** | Executes full disk release including admin channel cleanup |

### Why Admin Channel Triggers Disk Release

Admin channels are the **lifeline** to the target:
- Provide all management operations
- Required for resource allocation/deallocation
- Handle TOMA messages
- Process server commands
- Manage all I/O channels

**Therefore**: Admin channel failure → Cannot manage disk → Full disk release → Rediscovery

## Concurrent Release Handling

### Race Conditions and Solutions

#### Problem 1: Multiple Threads Calling `nvmeibc_disk_start_release()`

**Scenario**:
```
Thread A: Lock channel disconnect
Thread B: Admin channel disconnect
Both call nvmeibc_disk_start_release() simultaneously
```

**Solution**:
```c
// In nvmeibc_disk_start_release()
spin_lock_irqsave(&disk->restart_lock, flags);
if ((restart_called = atomic_inc_return(&disk->restart_called)) == 1) {
    // Only first thread schedules work
    nvmeibc_disk_add_work(disk, &rwork->work);
}
else {
    // Other threads just increment counter and return
    kfree(rwork);
}
spin_unlock_irqrestore(&disk->restart_lock, flags);
```

**Result**: Only one work item scheduled, no duplicates

#### Problem 2: Release Called During Rediscovery

**Scenario**:
```
Disk WQ: Executing nvmeibc_disk_release()
         Currently in rediscovery()
Thread C: New error occurs, calls nvmeibc_disk_start_release()
```

**Solution**:
```c
// In nvmeibc_disk_release()
atomic_set(&disk->restart_called, 1);  // Set to 1 before rediscovery
rv = rediscovery(disk);

if (rv == rr_ok) {
    if ((restart_called = atomic_dec_return(&disk->restart_called)) > 0) {
        // New release request came in during rediscovery
        goto restart;  // Restart the whole release process
    }
}
```

**Result**: New release request triggers restart after rediscovery completes

#### Problem 3: Release During Release

**Scenario**:
```
Disk WQ: Executing nvmeibc_disk_release()
         Currently in stop_disk_io()
Thread D: Another error, calls nvmeibc_disk_start_release()
```

**Solution**:
```c
// In nvmeibc_disk_release()
if ((dying = atomic_inc_return(&disk->dying)) > 1) {
    // Already releasing - exit immediately
    goto out;
}
```

**Combined with**:
```c
// In channel disconnect callbacks
if (!(dying = atomic_read(&disk->dying))) {
    // Only trigger release if not already dying
    nvmeibc_disk_start_release(...);
}
```

**Result**: Subsequent release attempts are ignored until first completes

### State Machine

```
State: ONLINE                State: RELEASING              State: ONLINE/OFFLINE
  dying=0                       dying=1                       dying=0
  restart_called=0              restart_called≥1              restart_called=0
       |                             |                             |
       |--Error occurs               |                             |
       |  start_release()            |--rediscovery succeeds       |
       |  restart_called: 0→1        |  restart_called: 1→0        |
       |  Schedule work              |  dying: 1→0                 |
       |                             |                             |
       v                             v                             |
  dying: 0→1                    [Check restart_called]            |
  Begin release                       |                           |
       |                         >1? ---yes--> goto restart       |
       |                          |                               |
       |                         no                               |
       |                          |                               |
       |                          +-------------------------------+
       |
       |--Another error
       |  start_release()
       |  restart_called: 1→2
       |  [Work already scheduled]
       |  
       v
  [Release continues]
  [disconnect_disk()]
  [free_disk_rsc()]
       |
       v
  restart_called: set to 1
  rediscovery()
       |
       v
  restart_called: 1→0  (if no new errors)
  dying: 1→0
       |
       v
  ONLINE (rediscovered)
```

## Workqueue Architecture

### Workqueues Involved in Release

| Workqueue | Owner | Purpose | Thread Count |
|-----------|-------|---------|--------------|
| **Disk WQ** | `disk->remove_wq` | Disk lifecycle operations | 1 (serialized) |
| **Admin WQ** | `admin_ch->base.remove_wq` | Admin channel operations | 1 per admin ch |
| **Lock Ch Callback WQ** | `lock_ch->callback_wq` | Lock operation callbacks | 1 or unbound |
| **Lock Ch Per-CPU WQ** | `lock_ch->_2nd_ch_pcpu_wq` | Per-CPU channel ops | 1 per CPU |
| **Disk Info Per-CPU WQ** | `disk->info->pcpu_wq` | Per-CPU NRCH operations | 1 per CPU |
| **Main WQ** | Global NVMesh main WQ | Configuration updates | N threads |

### Work Submission Paths

#### From Softirq Context

```c
// IB completion handler (softirq)
nvmeibc_ib_net_handle_qp_err()
    └─> on_disconnect_lock_ch()  // or on_disconnect_admin_ch()
        └─> nvmeibc_disk_start_release()
            ├─> alloc work (GFP_ATOMIC)
            └─> nvmeibc_disk_add_work()
                └─> wq_add(disk->remove_wq, work)
```

**Requirements**:
- Must use `GFP_ATOMIC`
- Cannot sleep
- Fast execution

#### From Workqueue Context

```c
// Admin WQ work
handle_req()
    └─> process_command()
        └─> [Error detected]
            └─> nvmeibc_disk_start_release()
                ├─> alloc work (GFP_KERNEL possible)
                └─> nvmeibc_disk_add_work()
```

**Requirements**:
- Can use `GFP_KERNEL`
- Can sleep if needed
- Should avoid long operations

#### From User Thread

```c
// User ioctl
nvmeibc_volume_detach()
    └─> nvmeibc_disk_remove()
        └─> nvmeibc_disk_start_release(reason: DISK_REMOVE)
            ├─> alloc work (GFP_KERNEL)
            └─> nvmeibc_disk_add_work()
```

**Requirements**:
- Can sleep
- May wait for completion
- User-interruptible

### Workqueue Ordering and Dependencies

```
[Disk WQ - Serialized]
     |
     +-- nvmeibc_disk_release_work
          |
          +-- nvmeibc_disk_release()
               |
               +-- stop_disk_io()
               |    └─> [Waits for block layer]
               |
               +-- disconnect_disk()
               |    |
               |    +-- [Disconnects admin channels]
               |    |    └─> nvmeibc_ib_admin_channel_disconnect()
               |    |         └─> [Breaks QP, frees resources]
               |    |
               |    +-- [Disconnects lock channels]
               |    |    └─> nvmeibc_locks_channel_disconnect_all_handles()
               |    |         └─> locks_channel_locks_remove_work()
               |    |              └─> [Scheduled on admin WQ]
               |    |
               |    +-- [Destroys JAM]
               |         └─> nvmeibc_jam_disk_del()
               |              └─> [Drains jam workqueue]
               |
               +-- free_disk_rsc()
               |
               +-- rediscovery()
                    └─> discover()
                         └─> [May re-create everything]
```

**Key Points**:
- Disk WQ is **serialized** - only one disk operation at a time
- Admin WQ operations may run concurrently with disk WQ
- Lock channel cleanup may be deferred to admin WQ
- JAM workqueue is drained before JAM is destroyed

### Deadlock Prevention

#### Rule 1: No Nested Work Submissions

**Bad**:
```c
// Disk WQ work
nvmeibc_disk_release_work() {
    // ...
    nvmeibc_disk_add_work(disk, &some_other_work);
    wait_for_completion(&some_other_work.done);  // DEADLOCK!
}
```

**Why**: Disk WQ is serialized, waiting for another work on same WQ deadlocks

**Solution**: Use different WQ or make async

#### Rule 2: Admin WQ → Disk WQ OK, Reverse Not OK

**OK**:
```c
// Admin WQ work
handle_req() {
    // ...
    nvmeibc_disk_start_release();  // Schedules on disk WQ
    return;  // Doesn't wait
}
```

**Bad**:
```c
// Disk WQ work
nvmeibc_disk_release() {
    // ...
    nvmeibc_admin_channel_add_work(...);
    wait_for_completion(...);  // Might deadlock if admin WQ waiting for disk
}
```

#### Rule 3: Always Check `dying` Before Scheduling Release

**Required**:
```c
if (!(dying = atomic_read(&disk->dying))) {
    nvmeibc_disk_start_release(disk, reason);
}
```

**Why**: Prevents endless release loop where disconnecting channels triggers more releases

## Debugging Disk Release

### Trace Events

```bash
# Enable disk release traces
echo 1 > /sys/kernel/debug/tracing/events/nvmesh/*disk*release*/enable

# Watch release flow
cat /sys/kernel/debug/tracing/trace_pipe | grep disk_release
```

### Key Traces

- `trace_disk_nvmeibc_disk_start_release`: Release initiated
- `trace_nvmeibc_disk_release_work`: Release work started
- `trace_disk_nvmeibc_disk_release`: Release began (with reason)
- `trace_disk_stop_disk_io`: Waiting for I/O to stop
- `trace_disk_disconnect_disk`: Disconnecting channels
- `trace_*_disk_nvmeibc_disk_release`: Progress through steps

### Procfs Information

```bash
# Disk status
cat /proc/nvmesh/disks/<disk_name>

# Check:
# - Status: Online/Paused/Dying
# - Last Discover rv
# - Release reason history (DTREND)
# - Peer release reason history
```

### Common Issues

#### Issue 1: Block Layer Not Pausing

**Symptom**:
```
ULP did not pause after N waits... disk is dead (no rediscovery)
```

**Cause**: Block layer has stuck I/O

**Debug**:
```bash
# Dump transfers
cat /proc/nvmesh/disks/<disk_name> | grep -A 50 "PD TRANSFERS"
```

**Solution**: Check for deadlocked locks, stuck I/O channels

#### Issue 2: Endless Release Loop

**Symptom**:
```
nvmeibc_disk_release() called repeatedly
restart_num keeps increasing
```

**Cause**: Disconnection callbacks triggering new releases

**Debug**:
```
# Check dying flag logic in channel disconnect callbacks
# Should have: if (!(dying = atomic_read(&disk->dying)))
```

**Solution**: Ensure all disconnect callbacks check `dying` flag

#### Issue 3: Rediscovery Fails Repeatedly

**Symptom**:
```
Rediscovery failed, try again...
(repeated many times)
```

**Cause**: Network issues, target unavailable, configuration problems

**Debug**:
```bash
# Check discover status
cat /proc/nvmesh/disks/<disk_name> | grep "Discover status"

# Check network connectivity
# Check target availability
```

## Performance Considerations

### Release Latency

Typical release takes:
- **stop_disk_io()**: 1-5 seconds (waiting for block layer)
- **disconnect_disk()**: 100-500ms (network operations)
- **free_disk_rsc()**: 50-200ms (memory cleanup)
- **rediscovery()**: Variable (0-30 seconds depending on success)

**Total**: 2-35 seconds typical

### Optimization: Fast Path for Clean Shutdown

When disk is cleanly removed (not error):
```c
if (reason == NVMEIBC_DISK_RELEASE_DISK_REMOVE) {
    disk->allow_rediscovery = false;  // Skip rediscovery
}
```

Saves 5-30 seconds by skipping unnecessary rediscovery

### Concurrency: Multiple Disks

Each disk has independent:
- `disk->remove_wq` - Serialized per disk
- `disk->dying` flag - Per disk state
- `disk->restart_called` - Per disk coordination

**Result**: Multiple disks can release concurrently without interference

## Summary

### Release Flow Comparison

| Trigger | Context | Reason | Rediscover? |
|---------|---------|--------|-------------|
| **Lock Ch Disconnect** | IB Softirq/WQ | LOCK_CHAN_DISCONNECT | Yes |
| **Admin Ch Disconnect** | IB Softirq/WQ | ADMIN_CHAN_DISCONNECT | Yes |
| **Volume Detach** | User Thread | DISK_REMOVE | No |
| **Config Update** | Main WQ | CONFIG_UPDATE_* | Yes |
| **I/O Error** | Channel WQ | REM_DRV_ERR | Yes |
| **Manual Rediscover** | User/Proc | REDISCOVER_CALLED | Yes |

### Critical Synchronization

1. **`disk->restart_lock`**: Serializes work scheduling
2. **`disk->restart_called`**: Coordinates concurrent release requests
3. **`disk->dying`**: Prevents nested/concurrent releases
4. **`disk->disk_paused`**: Synchronizes with block layer
5. **Channel-specific locks**: Protect per-channel state during cleanup

### Best Practices

1. **Always check `dying`** before calling `nvmeibc_disk_start_release()`
2. **Use GFP_ATOMIC** when calling from softirq/completion context
3. **Never wait** on disk WQ for another disk WQ work
4. **Set meaningful reasons** to aid debugging
5. **Test concurrent scenarios** (multiple errors simultaneously)

## Related Documentation

- `DISK_DISCOVERY_REDISCOVERY.md`: What happens after successful release
- `ADMIN_CHANNEL.md`: Admin channel lifecycle and operations
- `LOCKS_CHANNEL.md`: Lock channel operations and error handling
- `JAM_JOURNAL_ALLOCATION_MANAGER.md`: JAM destruction and cache preservation
- `DISK_UPDATES.md`: Configuration-triggered releases

