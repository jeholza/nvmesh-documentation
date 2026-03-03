# Client Disk Configuration Updates

## Overview

The disk update mechanism in `nvmeibc_disk.c` provides a framework for dynamically updating disk configuration without requiring a full restart. Updates are processed asynchronously on the disk's work queue, allowing hot-plugging of NICs/ports, configuration changes, and runtime parameter adjustments.

## Update Architecture

### Core Update Function
```c
int nvmeibc_disk_update_config(struct nvmeibc_disk *disk,
                               struct nvmeibc_disk_update_data *update_data, 
                               bool in_interrupt)
```

This function:
1. Allocates a work queue entry
2. Increments `disk->update_count` to track pending updates
3. Schedules work on disk's work queue
4. Processes update in `update_disk_config_work()`

### Update Data Structure
```c
struct nvmeibc_disk_update_data {
    enum nvmeibc_disk_update_type update_type;
    void *update_data;  // Type-specific data
    void (*done_cb)(void *ctx);  // Completion callback
    void *done_cb_ctx;
};
```

## Update Types

### 1. DISK_UPDATE_LOCAL_SRV
**Purpose**: Update local server interface

**When**: Local server module loads/unloads

**Processing**:
- If disk is accessed locally, triggers disk release
- Sets `disk->local_server` pointer
- Initiates rediscovery for local access

```c
if (disk->access_local) {
    nvmeibc_disk_release(disk, NVMEIBC_DISK_RELEASE_CONFIG_UPDATE_LOCAL_SRV);
}
disk->local_server = update_data->update_data;
```

### 2. DISK_UPDATE_REMOVE_NIC
**Purpose**: Handle NIC removal (hot-unplug)

**When**: Network interface card is removed from system

**Processing**:
- Checks if disk uses the removed NIC via `disk_uses_nic()`
- If disk is online and uses NIC, triggers disk release
- Removes NIC from `disk->local_nics` list
- Cleans up associated port structures

```c
if (!atomic_read(&disk->paused) && disk_uses_nic(disk, nic_dev)) {
    nvmeibc_disk_release(disk, NVMEIBC_DISK_RELEASE_CONFIG_UPDATE_NIC_REMOVE);
}
remove_disk_lnic(disk, nic_dev);
```

### 3. DISK_UPDATE_ADD_NIC
**Purpose**: Handle NIC addition (hot-plug)

**When**: New network interface card is added to system

**Processing**:
- Checks if NIC is already in disk's local NICs list
- If first port of previously unused device becomes active:
  - May defer to subsequent `DISK_UPDATE_ADD_PORT` event
- Otherwise adds NIC to disk's local NICs
- May trigger rediscovery if additional paths needed

```c
if (!nic_in_disks_local_nics(disk, nic_dev)) {
    nvmeibc_add_nic_disk_local_nics(disk, nic_dev);
}
```

### 4. DISK_UPDATE_REMOVE_PORT
**Purpose**: Handle port removal on existing NIC

**When**: Network port on NIC goes down or is removed

**Processing**:
- Checks if disk uses the port via `disk_uses_port()`
- If disk is online and uses port, triggers disk release
- Removes/updates port in local NICs list

```c
if (!atomic_read(&disk->paused) && disk_uses_port(disk, ib_port)) {
    nvmeibc_disk_release(disk, NVMEIBC_DISK_RELEASE_CONFIG_UPDATE_PORT_REMOVE);
}
remove_disk_lport(disk, ib_port->nic_dev, ib_port);
```

### 5. DISK_UPDATE_ADD_PORT
**Purpose**: Handle port addition on existing NIC

**When**: New network port becomes active on NIC

**Processing**:
- Adds/updates port in local NICs list
- May enable new paths to remote storage

```c
add_disk_lport(disk, ib_port->nic_dev, ib_port);
```

### 6. DISK_UPDATE_PORT_UPDATE
**Purpose**: Handle port state changes

**When**: Port state changes (active/inactive, link up/down)

**Processing**:
- If port becomes inactive and disk uses it, triggers disk release
- If port is on "cold" NIC (newly added), updates local port info
- Allows dynamic path reconfiguration

```c
if (!ib_port->port_active && !atomic_read(&disk->paused) && 
    disk_uses_port(disk, ib_port)) {
    nvmeibc_disk_release(disk, NVMEIBC_DISK_RELEASE_CONFIG_UPDATE_PORT_UPDATE);
} else if (is_lport_nic_cold(disk, ib_port)) {
    update_disk_lport(disk, ib_port);
}
```

### 7. DISK_UPDATE_WRITE_STATUS
**Purpose**: Generate status reports for procfs/debugging

**When**: User reads status from procfs

**Status Types**:
- `WRITE_STATUS_TEXT`: Human-readable text status
- `WRITE_STATUS_JSON`: JSON-formatted status
- `WRITE_STATUS_NRCH`: NoRDDA channel status
- `WRITE_STATUS_IOCH`: RDDA I/O channel status
- `WRITE_STATUS_QPS`: Queue pair statistics
- `WRITE_STATUS_IOCH_JSON`: JSON I/O channel status
- `WRITE_STATUS_COREMASK_JSON`: Coremask configuration
- `WRITE_STATUS_COREMASK_STATS`: Coremask statistics

Processing delegated to specialized functions:
```c
if (write_buf_data->status_type == WRITE_STATUS_JSON)
    write_status_json_buf(write_buf_data);
else if (write_buf_data->status_type == WRITE_STATUS_COREMASK_JSON)
    write_coremask_json_buf(write_buf_data);
// ... etc
```

### 8. DISK_UPDATE_JAM_ABND2FREE
**Purpose**: Process journal abandon-to-free requests

**When**: Journal entries need to be freed

**Processing**:
- Gets alive admin channel
- If disk has JAM (journal management) support, processes request
- Integrates with SERJIO journal manager

```c
if (ch && disk->jam_disk) {
    process_jmd_free_abandoned(disk, ch, a2f);
}
```

### 9. DISK_UPDATE_REMOTE_GID
**Purpose**: Handle remote GID changes

**When**: Remote target's GID (Global Identifier) changes

**Processing**:
- Gets alive admin channel
- Schedules work on admin channel work queue
- Handles path reconfiguration for new GID

```c
WQ_INIT_WORK(&rgidw->work, nvmeibc_disk_handle_rgid_change_work);
nvmeibc_admin_channel_add_work(&rgidw->ch->base, &rgidw->work);
```

### 10. DISK_UPDATE_DISCONNECT_IO_PATH
**Purpose**: Disconnect specific I/O path

**When**: Specific path needs to be torn down

**Processing**:
- Synchronously disconnects I/O path
- Uses completion to wait for disconnect
- Processed on admin channel work queue

```c
WQ_INIT_WORK(&disconnect_io_path_work->work, disconnect_io_path_work_fn);
nvmeibc_admin_channel_add_work(&disconnect_io_path_work->ch->base, 
                               &disconnect_io_path_work->work);
wait_for_completion(disconnect_io_path_work->comp);
```

### 11. DISK_UPDATE_RESET_QP_STATS
**Purpose**: Reset queue pair statistics

**When**: User requests stats reset via procfs

**Processing**:
- Iterates through all channels
- Resets statistics counters
- Runs callback on each QP's per-CPU data

```c
common_qps_run_cb(disk, _nvmeib_qp_stats_reset_safe, NULL);
```

### 12. DISK_UPDATE_RESET_COREMASK_STATS
**Purpose**: Reset coremask statistics

**When**: User requests coremask stats reset

**Processing**:
- Runs on each CPU
- Resets per-CPU coremask statistics
- Uses `on_each_cpu()` for SMP-safe reset

```c
if (disk->info && disk->info->coremask_info) {
    on_each_cpu(__reset_coremask_stats_pcpu_fn, disk->info->coremask_info, true);
}
```

### 13. DISK_UPDATE_COREMASK_UPDATE
**Purpose**: Update coremask channel configuration

**When**: Coremask masks are added/removed/modified

**Processing**:
- Schedules work on both main WQ and admin WQ
- Main WQ: Updates mask tree and channel allocation
- Admin WQ: Connects/disconnects coremask channels
- See COREMASK_SUPPORT.md for details

```c
nvmeibc_add_work(nvmeibc_isnt_params_core2main(p), 
                 &cinfo->update_masks_main_work);
nvmeibc_admin_channel_add_work(&ach->base, 
                               &cinfo->update_masks_admin_work);
```

## Update Flow Control

### Update Counter
```c
atomic_t update_count;  // In struct nvmeibc_disk
```

- Incremented when update is scheduled
- Decremented when update completes
- When counter reaches 0 and disk is paused:
  - Triggers rediscovery
  - Applies accumulated configuration changes

### Completion Callback
```c
if (update_data->done_cb)
    (*update_data->done_cb)(update_data->done_cb_ctx);
```

- Called after update processing completes
- Allows caller to synchronize with update
- Often used with completions for synchronous updates

### Rediscovery Trigger
```c
if (atomic_dec_and_test(&disk->update_count)) {
    if (atomic_read(&disk->paused) && !disk->detached) {
        disk->rediscover_timeout = 0;
        disk->rediscovery_now = 1;
        nvmeibc_disk_release(disk, NVMEIBC_DISK_RELEASE_CONFIG_UPDATE_COMPLETE);
    }
}
```

After all pending updates complete:
- If disk is paused, initiates rediscovery
- Applies all accumulated configuration changes
- Re-establishes connections with new configuration

## Update Context

### Interrupt Context
Some updates can arrive in interrupt context (e.g., from netlink events):
```c
int nvmeibc_disk_update_config(struct nvmeibc_disk *disk,
                               struct nvmeibc_disk_update_data *update_data, 
                               bool in_interrupt)
{
    if (!(work_qe = kzalloc(sizeof(*work_qe),
                            likely(!in_interrupt) ? GFP_KERNEL : GFP_ATOMIC)))
        ...
}
```

- `in_interrupt=true`: Uses `GFP_ATOMIC` for allocations
- `in_interrupt=false`: Uses `GFP_KERNEL` (can sleep)

### Work Queue Context
Updates are processed on disk's dedicated work queue:
```c
nvmeibc_disk_add_work(disk, &work_qe->work)
```

Benefits:
- Serialized processing (one update at a time)
- Can sleep/wait if needed
- Isolated from interrupt context
- Controlled by `disk->main_wq`

## Hot-Plug Support

The update mechanism enables full hot-plug support:

### NIC Hot-Plug
1. **Addition**: `DISK_UPDATE_ADD_NIC` → adds NIC to disk's local NICs
2. **Removal**: `DISK_UPDATE_REMOVE_NIC` → triggers release if in use

### Port Hot-Plug
1. **Addition**: `DISK_UPDATE_ADD_PORT` → adds new path option
2. **Removal**: `DISK_UPDATE_REMOVE_PORT` → triggers release if in use
3. **State Change**: `DISK_UPDATE_PORT_UPDATE` → handles link up/down

### Path Redundancy
- Disk release only triggered if removing last viable path
- Multiple paths provide failover capability
- Rediscovery finds alternative paths when available

## Synchronous vs Asynchronous Updates

### Asynchronous Updates (Common)
```c
nvmeibc_disk_update_config(disk, &update_data, false);
// Returns immediately, update processed later
```

### Synchronous Updates (Status Reporting)
```c
DECLARE_COMPLETION_ONSTACK(comp);
struct nvmeibc_disk_update_data disk_update_data = {
    .update_type = DISK_UPDATE_WRITE_STATUS,
    .update_data = &write_status_data,
    .done_cb = write_status_buf_done_cb,
    .done_cb_ctx = &comp,
};
nvmeibc_disk_update_config(disk, &disk_update_data, false);
wait_for_completion(&comp);
```

## Error Handling

### Update Failures
- Failed updates complete via callback
- Caller notified of failure
- Disk state left consistent

### Race Conditions
- Update counter prevents premature rediscovery
- Disk spinlock protects critical sections
- Pause state prevents conflicting operations

## Debugging Updates

### Trace Points
- `trace_update_disk_config_work_start`: Update begins
- `trace_update_disk_config_work_done`: Update completes
- Type-specific traces for each update type

### Monitoring
```bash
# Watch pending updates
cat /proc/nvmesh/disks/<disk>/status | grep update_count

# Monitor disk state
cat /proc/nvmesh/disks/<disk>/status | grep paused
```

### Common Issues
1. **Stuck updates**: Check `update_count`, may indicate deadlock
2. **Repeated releases**: May indicate flapping network/config
3. **Missing NICs**: Check hot-plug event delivery

## Related Mechanisms

- **Discovery**: Updates often trigger rediscovery (see DISK_DISCOVERY_REDISCOVERY.md)
- **Coremask**: Coremask updates use this framework (see COREMASK_SUPPORT.md)
- **Pausable**: Updates coordinate with pausable layer (see nvmeibc_pausable.c)
- **Channel Management**: Updates affect channel lifecycle (see CHANNEL_MANAGEMENT.md)

## Module Parameters

- `nvmeibc_disk_coremask_support`: Enable coremask update support (default: false)
- `nvmeibc_disk_max_coremask_nrch`: Max channels per coremask (default: 2)
- `nvmeibc_disk_use_coremask_pending`: Use coremask-aware pending queue (default: true)

