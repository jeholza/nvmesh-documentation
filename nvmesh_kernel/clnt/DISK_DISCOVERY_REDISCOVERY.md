# Client Disk Discovery and Rediscovery

## Overview

The disk discovery and rediscovery mechanism in `nvmeibc_disk.c` is responsible for establishing and maintaining connectivity between the client and remote (or local) disks. This process involves:

1. **Initial Discovery**: Connecting to disks for the first time
2. **Rediscovery**: Re-establishing connections after failures or configuration changes
3. **Path Finding**: Discovering viable network paths to reach remote storage
4. **Resource Allocation**: Obtaining remote disk resources for I/O operations

## Key Data Structures

### Discovery State Tracking
- `disk->discover_id`: Monotonically increasing discovery attempt counter
- `disk->discover_trend`: Trend tracking for discovery status history
- `disk->rediscover_timeout`: Timeout before attempting rediscovery
- `disk->rediscovery_now`: Flag to trigger immediate rediscovery

### Admin NICs (arnics)
- `disk->arnics`: List of `nvmeibc_admin_rnic` structures representing remote admin NICs
- Each arnic tracks:
  - `alive`: Whether the arnic is currently reachable
  - `channel`: Associated admin channel for communication
  - `priority`: Connection priority based on NUMA distance and other factors
  - `local`: Whether this arnic is on the local machine
  - `discover_trend`: Discovery status history for this arnic

### Local NICs
- `disk->local_nics`: List of local network interfaces that can reach remote storage
- Shuffled on each discovery attempt for load balancing
- Sorted by priority (e.g., hardware NICs preferred over software NICs like siw)

## Discovery Process

The main discovery function is `discover()` which follows these stages:

### Stage 1: Pre-Discovery Checks
```c
// Key checks performed:
- disk->force_pause: Debug flag to halt discovery
- disk->pause_at_first_discover: Pause on first discovery attempt
- Low memory conditions (LOW_MEM builds)
```

### Stage 2: Local NIC Preparation (DD_STG_FILL_LOCAL_NICS)
```c
fill_local_nics(disk)
```
- Shuffles local NICs for load balancing across discovery attempts
- Uses atomic counter to rotate through available NICs
- Validates that usable ports exist
- Fails with `NVMEIBC_DISK_DISCOVER_NO_LOCAL_NICS` if no NICs available

### Stage 3: Admin Channel Creation (DD_STG_CREATE_ADMIN_CHANNELS)
```c
create_admin_channels(disk)
```
- Creates admin channels for each arnic that matches `disk->config_node_id`
- Filters arnics by node configuration
- Sets `arnic->channel` to newly created admin channel
- Each channel enables communication with a remote admin NIC

### Stage 4: Path Discovery (DD_STG_DISCOVER_USING_PORT)
```c
call_for_each_lport(disk, discover_using_port, disk)
```
For each local port:
- Iterates through all arnics
- Checks compatibility (link layer, transport type, local vs remote)
- Calls `nvmeibc_disk_find_path()` to discover RDMA path
- Saves discovered path in `net->path` structure

Key path compatibility checks:
- Link layer must match (IB, Ethernet, etc.)
- Transport type must match (IWarp, RoCE, etc.)
- For local arnics, interface IDs must match

### Stage 5: Admin Channel Access Verification (DD_STG_CHECK_ARNICS_ACCESS)
```c
check_arnics_access(disk)
```
- Verifies that at least one arnic has a viable path
- Fails with `NVMEIBC_DISK_DISCOVER_NO_ARNICS_ACCESS` if none reachable

### Stage 6: Read I/O Resources (DD_STG_READ_ALL_IO_RSCS)
```c
read_all_io_rscs(disk)
```
- Connects admin channel for first reachable arnic
- Reads remote disk configuration via `read_all_io_rsc()`
- Gets list of I/O RNICs (remote NICs) from target
- For local disks:
  - Sets `disk->local_admin_ch`
  - Marks channel as `is_main`
  - Registers with local server

### Stage 7: I/O NIC Access (DD_STG_ACCESS_USING_PORT)
```c
call_for_each_lport(disk, access_using_port, disk)
```
- For each alive arnic, calls `nvmeibc_ib_admin_channel_access_iornics()`
- Establishes paths from local ports to remote I/O NICs
- Builds connectivity matrix for I/O operations

### Stage 8: Disk Access Check (DD_STG_CHECK_DISKS_ACCESS)
```c
check_disks_access(disk)
```
- Verifies remote disk accessibility
- Ensures at least one I/O RNIC is reachable
- Fails with `NVMEIBC_DISK_DISCOVER_NO_RIONICS` if not accessible

### Stage 9: Access Map Creation (DD_STG_CREATE_ACCESS_MAP)
```c
create_access_map(disk, is_rediscover)
```
- Creates access map for routing I/O requests
- Maps local ports to remote I/O RNICs
- Disconnects admin channels that provide no I/O RNICs

### Stage 10: Transport Type Check (DD_STG_CHECK_TRANSPORT_TYPE)
```c
disk_is_transport_tcp(disk)
```
- Verifies all arnics use same transport type
- Sets `disk->is_tcp` for TCP-based transports (iWarp)
- Critical for proper channel operation

### Stage 11: Resource Requests (DD_STG_REQUEST_DISKS_RESOURCES)
```c
request_disks_resources(disk)
```
- Requests RDDA resources from target (if supported)
- Selects main admin channel based on priority
- Sets `disk->cid` (client ID)
- Configures `disk->nr_get_by_cpu_index` function pointer

### Stage 12: Journal Range (DD_STG_GET_JOURNAL_RANGE)
```c
get_journal_range(disk) // or get_local_jrnl_rng(disk) for local
```
- Obtains journal range for EC (erasure coding) support
- Required for metadata operations
- Integrates with SERJIO journal manager

### Stage 13: Lock Channel Setup
For local disks:
```c
start_local_lock_channel(disk)
```
- Allocates local lock segments
- Connects lock channel for distributed locking
- Maps lock memory for RDMA access

For remote disks:
```c
start_lock_channels(disk)
```
- Establishes lock channels to remote lock servers
- Enables distributed lock management

### Stage 14: TOMA Registration
```c
toma_rereg_disk_handles(disk)
```
- Re-registers existing TOMA (target-offload management agent) connections
- Maintains volume subscriptions across rediscovery

### Stage 15: I/O Channel Startup
```c
nvmeibc_disk_start_io_channels(disk)
```
- Starts RDDA channels (if available)
- Starts NoRDDA channels
- Establishes per-CPU channels (if configured)
- Applies coremask constraints

### Stage 16: Completion and Status
- Marks disk as online: `atomic_set(&disk->paused, 0)`
- Wakes up waiters: `wake_up_interruptible(&disk->wait_queue)`
- Logs discovery status to trend tracking
- Records downtime statistics

## Local Disk Discovery

Local disks (where target and client run on same machine) use a simplified path:

1. **is_local_disk()**: Queries local server to determine if disk is local
2. **local_disk_cl_register()**: Registers with local server's client interface
3. **build_local_lock_segments()**: Sets up local lock memory
4. **start_local_lock_channel()**: Connects to local lock channel

Benefits of local disk detection:
- Direct memory access instead of RDMA
- Lower latency
- Reduced NIC utilization
- Enabled via `nvmeibc_use_local_bypass` module parameter

## Rediscovery Triggers

Rediscovery can be triggered by:

1. **Channel Failures**: Admin or I/O channel disconnections
2. **Configuration Updates**: Changes to disk configuration
3. **Path Failures**: Loss of network connectivity
4. **Timeouts**: Periodic reconnection attempts
5. **Manual Triggers**: Debug/administrative commands

Rediscovery flow:
```c
nvmeibc_disk_release(disk, reason) 
  -> cleanup existing channels
  -> reset disk state
  -> schedule rediscovery work
  -> discover(disk, is_rediscover=true)
```

## Discovery Status Tracking

Each discovery stage can fail with a specific status code:

```c
enum nvmeibc_disk_discover_reasons {
    NVMEIBC_DISK_DISCOVER_UNKNOWN,
    NVMEIBC_DISK_DISCOVER_OK,
    NVMEIBC_DISK_DISCOVER_NO_LOCAL_NICS,
    NVMEIBC_DISK_DISCOVER_NO_LOCAL_PORTS,
    NVMEIBC_DISK_DISCOVER_NO_ARNICS_ACCESS,
    NVMEIBC_DISK_DISCOVER_NO_RIONICS,
    NVMEIBC_DISK_DISCOVER_ADMIN_CH_CREATE_FAILED,
    NVMEIBC_DISK_DISCOVER_ACCESS_MAP_CREATE_FAILED,
    // ... many more status codes
};
```

Status tracked via:
- `DISK_DISCOVER_STATUS(disk, status)`: Records disk-level status
- `ARNIC_DISCOVER_STATUS(arnic, status)`: Records arnic-level status
- Trend tracking: `nvmeib_trend_insert(&disk->discover_trend, status)`

## Discovery Stages Macro System

The discovery process uses a macro system for tracing and timing:

```c
DD_STG_START(disk, stage_name)  // Start timing stage
DD_STG_END(disk, stage_name, rv, status_code)  // End timing and record status
```

Example:
```c
DD_STG_START_WITH_DECLARE(disk, DD_STG_READ_ALL_IO_RSCS);
// ... stage code ...
DD_STG_END(disk, DD_STG_READ_ALL_IO_RSCS, rv, NVMEIBC_DISK_DISCOVER_IO_RESOURCES_READ_FAIL);
```

This provides detailed tracing of:
- Stage start/end
- Time spent in each stage
- Success/failure status
- Correlation with `disk->discover_id`

## Key Module Parameters

- `nvmeibc_use_local_bypass`: Enable local disk bypass (default: true)
- `nvmeibc_restart_io_timeout_secs`: Time between I/O channel reconnection cycles (default: 30s)
- `nvmeibc_max_ioch_path_fails`: Max failures per path in connection cycle (default: 1)

## Debugging Discovery Issues

To debug discovery problems:

1. **Check discovery trend**: Look at `disk->discover_trend` for failure history
2. **Verify arnic status**: Check each `arnic->discover_trend` for path-specific issues
3. **Enable discovery tracing**: Look for "DD_STG_" and "DISCOVER_ID=" trace messages
4. **Check status buffer**: Read `/proc/.../disk_status` for current state
5. **Verify network paths**: Ensure local ports can reach remote arnics
6. **Check configuration**: Verify `config_node_id` matches between disk config and arnics

## Related Files

- `nvmeibc_ib_admin_channel.c`: Admin channel implementation
- `nvmeibc_ib_io_channel.c`: I/O channel implementation (RDDA)
- `nvmeibc_ib_nordda_channel.c`: NoRDDA channel implementation
- `nvmeibc_targets.c`: Target management and arnic tracking

