# Client Disk Documentation

This directory contains comprehensive documentation for the NVMesh client disk subsystem (`nvmeibc_disk.c`).

## Overview

The client disk layer is responsible for:
- Discovering and connecting to remote disks
- Managing I/O channels (RDMA and No-RDMA)
- Routing I/O requests to remote targets
- Supporting local bypass for co-located storage
- Managing resources and lifecycle
- Integrating with management services (TOMA)

## Documentation Files

### Core Functionality

#### [DISK_DISCOVERY_REDISCOVERY.md](DISK_DISCOVERY_REDISCOVERY.md)
Describes how the client discovers remote disks and establishes connectivity:
- **Finding NICs**: Discovering local and remote NICs
- **Creating channels**: Setting up admin channels for management
- **Access maps**: Building connectivity maps for I/O operations
- **Rediscovery**: Handling network changes and failover
- **Troubleshooting**: Common discovery issues

**Start here** if you want to understand how the client connects to storage.

#### [DISK_CHANNEL_MANAGEMENT.md](DISK_CHANNEL_MANAGEMENT.md)
Explains I/O execution and channel management:
- **execute_io_remote**: Main remote I/O execution path
- **Channel types**: RDDA vs No-RDDA channels
- **Channel selection**: Getting and managing channels
- **I/O submission**: Command queuing and execution
- **Channel lifecycle**: Creation, usage, and cleanup

**Essential reading** for understanding how I/O flows through the system.

#### [DISK_RESOURCE_MANAGEMENT.md](DISK_RESOURCE_MANAGEMENT.md)
Documents resource allocation and lifecycle:
- **Channel resources**: Allocating QPs, buffers, completion queues
- **Resource pools**: Managing available vs owned resources
- **Lifecycle**: Allocation, usage, and freeing
- **Error handling**: Cleanup and recovery

**Important** for understanding memory and RDMA resource management.

### Advanced Features

#### [DISK_LOCAL_BYPASS.md](DISK_LOCAL_BYPASS.md)
Details the local bypass optimization:
- **When to use**: Detecting local disks
- **Registration**: Connecting to local server
- **I/O execution**: Direct vs RPC paths
- **Performance**: Benefits and tradeoffs

**Read this** if working with hyper-converged deployments.

#### [DISK_PER_CPU_NRCH_AND_COREMASK.md](DISK_PER_CPU_NRCH_AND_COREMASK.md)
Covers per-CPU channel optimization:
- **Per-CPU channels**: Lockless I/O from dedicated CPUs
- **Coremask support**: Restricting I/O to specific CPUs
- **Configuration**: Module parameters and tuning
- **Use cases**: When and how to use per-CPU channels

**Advanced topic** for performance optimization.

#### [DISK_REUSED_BB_RELEASE.md](DISK_REUSED_BB_RELEASE.md)
Explains bounce buffer reuse:
- **Erasure coding**: Why buffers need reuse
- **Release logic**: When and how buffers are freed
- **Synchronization**: Coordinating with upper layers
- **Debugging**: Tracking buffer usage

**Specialized topic** for EC and buffer management.

### Management and Updates

#### [DISK_UPDATES.md](DISK_UPDATES.md)
Documents disk configuration updates:
- **NIC changes**: Adding/removing network interfaces
- **Coremask updates**: Changing CPU restrictions
- **Status writes**: Persisting disk state
- **QP stats**: Resetting performance counters
- **Workqueue**: Deferred update processing

**Important** for understanding dynamic configuration.

#### [DISK_TOMA_INTEGRATION.md](DISK_TOMA_INTEGRATION.md)
Explains TOMA (TOpology MAnager) integration:
- **Subscribe/Unsubscribe**: Registering for management messages
- **Send/Receive**: Communicating with target
- **Rediscovery**: Maintaining subscriptions across failover
- **Async vs Sync**: Operating modes

**Read this** if working with management or control plane.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     Block/Volume Layer                       │
│                  (nvmesh_cinst, nvmesh_admin)               │
└────────────────────┬──────────────┬──────────────────────────┘
                     │              │
                     │ I/O          │ TOMA
                     │              │
┌────────────────────▼──────────────▼──────────────────────────┐
│                     Disk Layer                                │
│                  (nvmeibc_disk.c)                            │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Discovery   │  │   Channel    │  │   Resource   │      │
│  │              │  │  Management  │  │  Management  │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │    Local     │  │   Per-CPU    │  │    TOMA      │      │
│  │   Bypass     │  │    NRCH      │  │ Integration  │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└────────────────────┬──────────────┬──────────────────────────┘
                     │              │
                     │ RDMA         │ Admin
                     │              │
┌────────────────────▼──────────────▼──────────────────────────┐
│                   Channel Layer                               │
│         (nvmeibc_channel.c, nvmeibc_ib_admin_channel.c)      │
└────────────────────┬──────────────┬──────────────────────────┘
                     │              │
                     │ RDMA Verbs   │ IB/RoCE
                     │              │
┌────────────────────▼──────────────▼──────────────────────────┐
│                  RDMA/IB Layer                                │
│               (rdma_cm, ib_core, mlx5_ib)                    │
└───────────────────────────────────────────────────────────────┘
                            │
                            │ Network
                            │
┌───────────────────────────▼───────────────────────────────────┐
│                     Target Server                             │
│                  (srv/nvmeib_server.c)                        │
└───────────────────────────────────────────────────────────────┘
```

## Key Concepts

### Disk States

```c
enum disk_status {
    d_offline,     // Not initialized
    d_discovery,   // Discovering resources
    d_online,      // Ready for I/O
    d_releasing,   // Cleaning up
    d_released     // Fully released
};
```

### Channel Types

- **Admin Channels**: Management operations (discovery, TOMA)
- **RDDA Channels**: RDMA-based I/O (high performance)
- **No-RDDA Channels (NRCH)**: Non-RDMA I/O (fallback, metadata)
- **Per-CPU NRCH**: Lockless channels for dedicated CPUs

### NICs

- **Local NICs**: Network interfaces on client machine
- **Admin RNIC**: Remote NIC providing management access
- **I/O RNIC**: Remote NIC handling I/O operations

## Common Workflows

### Starting I/O to a New Disk

1. **Discovery** ([DISK_DISCOVERY_REDISCOVERY.md](DISK_DISCOVERY_REDISCOVERY.md))
   - Find local and remote NICs
   - Create admin channels
   - Request disk resources

2. **Channel Setup** ([CHANNEL_MANAGEMENT.md](CHANNEL_MANAGEMENT.md))
   - Allocate channel resources
   - Create QPs and completion queues
   - Connect channels

3. **I/O Execution** ([CHANNEL_MANAGEMENT.md](CHANNEL_MANAGEMENT.md))
   - Get available channel
   - Submit I/O command
   - Process completion

### Handling Network Changes

1. **Detection** ([DISK_DISCOVERY_REDISCOVERY.md](DISK_DISCOVERY_REDISCOVERY.md))
   - NIC/port state change
   - Admin channel failure

2. **Release** ([RESOURCE_MANAGEMENT.md](RESOURCE_MANAGEMENT.md))
   - Drain in-flight I/O
   - Close channels
   - Free resources

3. **Rediscovery** ([DISK_DISCOVERY_REDISCOVERY.md](DISK_DISCOVERY_REDISCOVERY.md))
   - Re-discover NICs
   - Rebuild channels
   - Resubscribe TOMA

### Updating Configuration

1. **Request** ([DISK_UPDATES.md](DISK_UPDATES.md))
   - Upper layer requests update
   - Schedule workqueue item

2. **Update** ([DISK_UPDATES.md](DISK_UPDATES.md))
   - Acquire necessary locks
   - Apply changes
   - Update state

3. **Notify** ([DISK_UPDATES.md](DISK_UPDATES.md))
   - Inform upper layers
   - Log changes

## Module Parameters

Critical parameters for configuring the disk subsystem:

```bash
# Channel counts
modprobe nvmeibc nr_channels=16              # I/O channels per disk
modprobe nvmeibc nr_noRDDA_channels=4        # No-RDDA channels per disk
modprobe nvmeibc nr_pcpu_channels_per_disk=0 # Per-CPU channels (0=disabled)

# Local bypass
modprobe nvmeibc use_local_bypass=1          # Enable local bypass

# Coremask
modprobe nvmeibc coremask_support=1          # Enable coremask
modprobe nvmeibc max_coremask_nrch=4         # Max NRCH per coremask

# TOMA
modprobe nvmeibc use_async_subscribe=1       # Async TOMA subscribe

# Performance
modprobe nvmeibc nr_pcpu_ch_lockless=1       # Lockless per-CPU channels
```

## Debugging

### Check Disk Status

```bash
# View disk state
cat /proc/nvmesh/disks/<disk>/status

# Check discovery progress
cat /proc/nvmesh/disks/<disk>/discovery_log
```

### Enable Tracing

```bash
# Enable all disk events
echo 1 > /sys/kernel/debug/tracing/events/nvmesh/enable

# Watch discovery
cat /sys/kernel/debug/tracing/trace_pipe | grep discover

# Watch I/O path
cat /sys/kernel/debug/tracing/trace_pipe | grep execute_io
```

### Common Issues

1. **Discovery fails** → See [DISK_DISCOVERY_REDISCOVERY.md](DISK_DISCOVERY_REDISCOVERY.md#troubleshooting)
2. **I/O slow** → See [CHANNEL_MANAGEMENT.md](CHANNEL_MANAGEMENT.md#performance-considerations)
3. **Local bypass not working** → See [LOCAL_BYPASS.md](LOCAL_BYPASS.md#troubleshooting)
4. **Coremask issues** → See [PER_CPU_NRCH_AND_COREMASK.md](PER_CPU_NRCH_AND_COREMASK.md#debugging)
5. **TOMA messages lost** → See [TOMA_INTEGRATION.md](TOMA_INTEGRATION.md#common-issues)

## Performance Tuning

### High Throughput

```bash
# More channels for parallelism
nr_channels=32

# Lockless per-CPU channels
nr_pcpu_channels_per_disk=8
nr_pcpu_ch_lockless=1
```

### Low Latency

```bash
# Fewer channels (less contention)
nr_channels=8

# Pin to specific CPUs
coremask_support=1
```

### Mixed Workload

```bash
# Balance RDDA and No-RDDA
nr_channels=16
nr_noRDDA_channels=8
```

## Code Structure

The main file `nvmeibc_disk.c` is organized as:

```
Lines      Section
------     -------
1-500      Includes, definitions, module parameters
500-1000   Data structures and helpers
1000-3000  Resource management
3000-5000  Channel allocation and management
5000-7000  Local bypass
7000-8500  Remote I/O execution
8500-9500  Discovery and rediscovery
9500-11000 Disk lifecycle (init, release)
11000-12000 TOMA integration
12000-13000 Disk updates
13000+      Module init/exit
```

## Testing

### Unit Tests

Located in `core_unitest/`:
- `test_disk_discovery.c`
- `test_channel_management.c`
- `test_resource_management.c`

### Integration Tests

Located in `testing/`:
- `test_discovery_failover.sh`
- `test_local_bypass.sh`
- `test_coremask.sh`

### Performance Tests

Located in `perfTest/`:
- `disk_latency.sh`
- `disk_throughput.sh`

## Related Documentation

- `../README.md`: Client subsystem overview
- `nvmeibc_channel.c`: I/O channel implementation
- `nvmeibc_ib_admin_channel.c`: Admin channel implementation
- `../docs/CLIENT_ARCHITECTURE.md`: Overall client architecture

## Contributing

When modifying the disk subsystem:

1. **Update documentation**: Keep these docs in sync with code changes
2. **Add tracepoints**: Ensure new code paths have debugging support
3. **Test thoroughly**: Run unit, integration, and performance tests
4. **Review checklist**:
   - [ ] Discovery/rediscovery still works
   - [ ] Local bypass tested
   - [ ] Coremask compatibility
   - [ ] TOMA subscriptions survive rediscovery
   - [ ] No resource leaks
   - [ ] Performance not regressed

## References

### Internal

- Design doc: `docs/design/client_disk.md`
- API reference: `include/nvmeibc_disk.h`
- Protocol spec: `docs/protocol/nvmeib_protocol.md`

### External

- [RDMA Verbs Documentation](https://www.rdmamojo.com/)
- [NVMe Specification](https://nvmexpress.org/)
- [Linux Kernel RDMA Subsystem](https://www.kernel.org/doc/html/latest/infiniband/index.html)

## Support
TBD

