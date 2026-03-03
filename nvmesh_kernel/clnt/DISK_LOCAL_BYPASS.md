# Client Disk Local Bypass

## Overview

Local bypass is an optimization that allows the NVMesh client to access disks directly when they are on the same machine as the target server, bypassing the network stack and RDMA for significantly improved performance and reduced latency.

When local bypass is enabled and a disk is detected as local, I/O operations are submitted directly to the local NVMe driver instead of being sent over RDMA to a remote target.

## Configuration

### Module Parameter
```c
bool nvmeibc_use_local_bypass = true;
module_param_named(use_local_bypass, nvmeibc_use_local_bypass, bool, 0644);
```

- **Default**: `true` (enabled)
- **Runtime Configurable**: Yes (via `/sys/module/nvmeibc/parameters/use_local_bypass`)
- **When to Disable**: Only for testing/debugging RDMA paths on local disks

## Local Disk Detection

### Detection During Discovery

The discovery process determines if a disk is local:

```c
static int discover(struct nvmeibc_disk *disk, bool is_rediscover)
{
    bool use_local_bypass = nvmeibc_use_local_bypass;
    
    // ... discovery stages ...
    
    if (use_local_bypass && is_local_disk(disk)) {
        disk->access_local = true;
        disk->is_local = true;
    }
}
```

### is_local_disk() Function

```c
static bool is_local_disk(struct nvmeibc_disk *disk)
{
    if (!disk->local_server)
        return false;
    
    // Query local server: nvmeibs_is_local_disk()
    return disk->local_server->is_local_disk(disk->name, &disk->md_size, &disk->md_extd);
}
```

**Process**:
1. Checks if local server module is loaded (`disk->local_server` != NULL)
2. Calls into server's `is_local_disk()` function
3. Server checks if it owns a disk with this name
4. Returns metadata configuration (MD size, extended MD flag)

### Local Disk Registration

After detecting a local disk, the client registers with the local server:

```c
static int local_disk_cl_register(struct nvmeibc_disk *disk)
{
    // Register with server: nvmeibs_client_ldisk_register()
    if (disk->local_server->cl_register(disk->local_admin_ch->cid,
                                       disk->name, &disk->local) < 0)
        return -1;
    
    disk->is_cl_reg = true;
    disk->sector_shift = disk->local.sector_shift;
    disk->max_request_size_bytes = disk->local.max_request_size << disk->sector_shift;
    return 0;
}
```

**Registration includes**:
- Client ID (`cid`)
- Disk name
- Receives local disk handle (`disk->local`)
- Obtains disk geometry (sector shift, max request size)

## Local I/O Path

### Execute I/O Decision Point

```c
static int execute_io(struct nvmeibc_disk *disk,
                     struct nvmeibc_disk_io_command *block_cmd)
{
    if (disk->access_local)
        return execute_io_local(disk, &block_cmd->disk_cmd);
    else
        return execute_io_remote(disk, &block_cmd->disk_cmd);
}
```

The `disk->access_local` flag directs I/O operations to the local path.

### Local I/O Execution Flow

#### 1. execute_io_local()

```c
static int execute_io_local(struct nvmeibc_disk *disk,
                            struct nvmeibc_disk_command *disk_cmd)
{
    if (disk_cmd->cmd_type == NVMEIBC_DISK_CMD_IO)
        return execute_io_local_cmd_io(disk, disk_to_block(disk_cmd));
    else if (disk_cmd->cmd_type == NVMEIBC_DISK_CMD_GEN)
        return execute_io_local_cmd_gen(disk, disk_to_gen(disk_cmd));
    else if (disk_cmd->cmd_type == NVMEIBC_DISK_CMD_LOCK)
        return execute_io_local_cmd_lock(disk, disk_to_lock(disk_cmd));
    
    return -EINVAL;
}
```

Dispatches to command-specific local execution functions.

#### 2. execute_io_local_cmd_io()

For I/O commands (read/write/discard):

```c
static int execute_io_local_cmd_io(struct nvmeibc_disk *disk,
                                   struct nvmeibc_disk_io_command *block_cmd)
{
    struct nvmeibc_block_io_req *req = &block_cmd->reqs[0];
    enum nvme_opcode nvme_op;
    
    // Convert block op to NVMe opcode
    if (block_cmd_2_nvme_op_local(block_cmd, &nvme_op) < 0)
        return -1;
    
    // Handle metadata operations
    if (req->op == NVMEIB_BLOCK_IO_OP_MD_READ ||
        req->op == NVMEIB_BLOCK_IO_OP_MD_RD_MOD_WR) {
        if (execute_io_local_md_alloc_sg(block_cmd) < 0)
            return -1;
    }
    
    // Prepare NVMe request
    req->req.nvme_op = nvme_op;
    req->req.disk_block = req->disk_address;
    req->req.data_len = req->ndb->length;
    req->req.callback = execute_io_local_cb;
    req->req.arg = block_cmd;
    
    // Handle PRPL (Physical Region Page List) if needed
    if (disk->local.local_io_use_prpl) {
        if (local_io_req_fill_prpl(disk, block_cmd) < 0)
            return -1;
    }
    
    // Submit to local server: nvmeibs_handle_local_cmd()
    return disk->local_server->local_cmd(&disk->local, &req->req);
}
```

**Key points**:
- Converts block layer operations to NVMe opcodes
- Handles metadata reads (MD_READ, MD_RD_MOD_WR)
- Uses PRPL for efficient DMA if configured
- Submits directly to local server's command handler

#### 3. execute_io_local_cb()

I/O completion callback:

```c
static void execute_io_local_cb(void *arg, int status, u32 result)
{
    struct nvmeibc_disk_io_command *block_cmd = arg;
    struct nvmeibc_disk *disk = block_cmd->disk;
    int comp_code = status ? -EIO : 0;
    
    // Handle metadata trim operations (read-modify-write)
    if (block_cmd->reqs[0].op == NVMEIB_BLOCK_IO_OP_MD_RD_MOD_WR) {
        if (on_md_rd_mod_wr_comp(block_cmd, &comp_code) == 0)
            return; // Will complete after second phase
    }
    
    // Cleanup resources
    if (disk->local.local_io_use_prpl)
        local_io_req_free_prpl(disk, block_cmd);
    
    if (block_cmd->reqs[0].ndb->core_sgl)
        execute_io_local_md_free_sg(block_cmd);
    
    // Check piggyback lock operations
    check_local_piggyback_lock(disk, block_cmd);
    
    // Complete to block layer
    if (NVMEIBC_LOCAL_DEFER_COMPLETE_IOCMD && 
        disk->local_defer_block_cb_on_io_cmd) {
        // Defer completion to work queue
        list_add_tail(&block_cmd->disk_cmd.dcmd_link, &disk->local_defer_io_list);
        schedule_work(&disk->local_defer_io_work);
    } else {
        // Direct completion
        nvmeibc_block_completion(&block_cmd->comp);
    }
}
```

**Completion modes**:
- **Direct**: Completes immediately in callback context
- **Deferred**: Schedules work to complete on different CPU/context
- **Per-CPU**: Can target specific CPU for completion

### Local General Commands

For general commands (journal, EC operations, locks):

```c
static void execute_io_local_gen_work(struct workqe_struct *work)
{
    struct nvmeibc_disk_gen_cmd *gen_cmd = container_of(work, ...);
    struct nvmeibc_disk *disk = gen_cmd->disk;
    
    switch (gen_cmd->opcode) {
    case NVMEIB_GEN_OP_BLKSET_RECOVERED:
    case NVMEIB_GEN_OP_GET_UUID_JOUR:
    case NVMEIB_GEN_OP_GET_EC_DB:
    case NVMEIB_GEN_OP_FREE_JRNL_ENTS:
    case NVMEIB_GEN_OP_GET_JMDC:
    case NVMEIB_GEN_OP_JENTRY_ERASE:
        gen_cmd->param.async_cb = nvmeibc_disk_complete_gen_cmd;
        rv = disk->local_server->gen_cmd(&disk->local, gen_cmd->opcode,
                                        &gen_cmd->param, &gen_cmd->rsp,
                                        disk->cid);
        break;
    }
    
    if (rv != NVMEIBS_IO_RSP_EXPECT_ASYNC_REPLY)
        nvmeibc_disk_complete_gen_cmd(&gen_cmd->rsp, rv);
}
```

**Execution**:
- Scheduled on `disk->local_gen_wq` work queue
- Calls into server's general command handler
- Supports asynchronous completion

### Local Lock Commands

For distributed lock operations:

```c
static void execute_io_local_lock_work(struct workqe_struct *work)
{
    struct nvmeibc_disk_lock_cmd *disk_lock_cmd = container_of(work, ...);
    struct nvmeibc_disk *disk = disk_lock_cmd->disk;
    
    struct nvmeib_gen_cmd_param lock_gen_p = {
        .lock_param = disk_lock_cmd->lock_param,
    };
    
    // nvmeibs_handle_gen_cmd() for lock operations
    rv = disk->local_server->gen_cmd(&disk->local, NVMEIB_GEN_OP_LOCK,
                                     &lock_gen_p, &lock_gen_rsp, disk->cid);
    
    disk_lock_cmd->lock_rsp = lock_gen_rsp.lock_rsp;
    nvmeibc_locks_channel_lock_cmd_completion(disk_lock_cmd, rv,
                                              LOCK_OPR_BYPASS_IN_LOCAL_WQ);
}
```

## Local Disk Resources

### Local Admin Channel

```c
struct nvmeibc_admin_channel *local_admin_ch;  // In struct nvmeibc_disk
```

- Created during discovery even for local disks
- Used for lock channel operations
- Marked as `is_main = true`
- Has client ID (`cid`) for server communication

### Local Lock Segments

For distributed locking on local disks:

```c
static int start_local_lock_channel(struct nvmeibc_disk *disk)
{
    // Allocate lock segments from local server
    local_disk_locks_alloc(disk, disk_segs_locks_local);
    
    // Build segment structures
    build_local_lock_segments(disk, disk_segs_locks_local);
    
    // Connect lock channel
    nvmeibc_locks_channel_connect(lock_ch, ...);
}
```

Even local disks use distributed locks for multi-client coordination.

### Local Journal Range

For erasure coding and journaling:

```c
static int get_local_jrnl_rng(struct nvmeibc_disk *disk)
{
    // Allocate journal metadata cache
    disk->local.jrnl.jmdc = kcalloc(NVMEIB_EC_JOURNAL_MAX_BLKS_PER_RANGE, ...);
    
    // Build journal cache for SERJIO
    nvmeibc_jam_disk_cache_local(disk, &jrc);
    
    // Allocate journal range: nvmeibs_client_ldisk_alloc_jrnl_rng()
    disk->local_server->cl_alloc_jrnl_rng(disk->local_admin_ch->cid,
                                         &disk->local, &jrc, disk->binje_ulp);
    
    // Copy journal info to disk
    disk->jour.rng_id = disk->local.jrnl.rng_idx;
    disk->jour.rng_slba = disk->local.jrnl.rng_slba;
    // ... etc
}
```

## Performance Optimizations

### PRPL (Physical Region Page List)

For efficient DMA without RDMA:

```c
struct local_io_prpl {
    u64 *prp;              // Array of physical page addresses
    dma_addr_t prp_dma;    // DMA address of PRPL
    unsigned int n_prps;    // Number of entries
};

disk->local.local_io_use_prpl = true;  // Enable PRPL mode
```

**Benefits**:
- Single DMA mapping for scatter-gather list
- Reduces CPU overhead vs per-page DMA
- Similar to NVMe PRP (Physical Region Page) mechanism

### DMA Pools

For frequently used buffer sizes:

```c
disk->local.dma_pools;  // Per-CPU DMA pools

// Pool types:
- NVMEIB_DMA_POOL_TYPE_PRPL: PRPL structures
- NVMEIB_DMA_POOL_TYPE_RD_MD: Metadata read buffers
```

**Benefits**:
- Reduces allocation overhead
- Per-CPU pools reduce contention
- DMA-mapped memory ready to use

### Deferred Completion

```c
NVMEIBC_LOCAL_DEFER_COMPLETE_IOCMD = true
disk->local_defer_block_cb_on_io_cmd = true
```

**Options**:
1. **Direct completion**: Complete in callback (fast but may cause latency spikes)
2. **Deferred to work queue**: Completes on dedicated work queue
3. **Per-CPU work queue**: Targets specific CPU for completion

**Rationale**: Moves completion processing off interrupt/callback context.

## Metadata Handling

### MD_READ Operations

For reading just metadata:

```c
// Allocate dummy area for data
execute_io_local_md_alloc_sg(block_cmd);

// Actual read operation reads data + metadata
disk->local_server->local_cmd(&disk->local, &req->req);

// Extract metadata, discard data
// (Metadata placed in req->md by local server)
```

### MD_RD_MOD_WR Operations (Metadata Trim)

Read-modify-write to zero metadata:

```c
// Phase 1: Read data + metadata
execute_io_local_cmd_io(...);
  -> nvme_cmd_read

// Phase 2 (on completion): Write data + zeroed metadata
on_md_rd_mod_wr_comp() {
    req->req.nvme_op = nvme_cmd_write;
    req->req.metadata = page_address(ZERO_PAGE(0));  // Zero page
    disk->local_server->local_cmd(&disk->local, &req->req);
}
```

**Use case**: Trim operation for drives with metadata (DIF/DIX).

## Piggyback Lock Operations

Local disks support piggyback lock reads:

```c
static void check_local_piggyback_lock(struct nvmeibc_disk *disk,
                                      struct nvmeibc_disk_io_command *block_cmd)
{
    // Extract lock address from piggyback parameters
    nvmeibc_disk_locks_extract_info(block_cmd->lpb.handle,
                                   block_cmd->lpb.addr, ...);
    
    // Read lock value from mapped memory
    lock_entry.all = *(u64 *)(page_address(lock_pages[virt_index]) + virt_entry);
    
    // Return lock value in completion
    nvmeibc_set_lock_read(block_cmd, dc, lock_entry.all, ...);
}
```

**Benefit**: Reads lock value without separate RDMA operation.

## Local vs Remote Comparison

| Aspect | Local Bypass | Remote RDMA |
|--------|-------------|-------------|
| Latency | ~10-50 μs | ~100-500 μs |
| CPU | Lower (no RDMA stack) | Higher (RDMA processing) |
| NIC | Not used | Required |
| Bandwidth | Full PCIe | Limited by NIC/network |
| Failure Domain | Same machine | Survives client failure |
| Use Case | Compute + storage | Disaggregated storage |

## Restrictions

Local bypass is NOT supported for:

1. **Coremask operations**: Requires remote channels
   ```c
   if (disk->access_local) {
       _NW("Coremask not supported for local disk");
       return 0;
   }
   ```

2. **RDDA channels**: Local disks don't allocate RDDA resources
   ```c
   BUG_ON(disk->access_local);  // In resource allocation code
   ```

3. **Multi-client scenarios**: Coordination still uses lock channels

## Debugging Local Bypass

### Verify Local Detection

```bash
# Check disk status
cat /proc/nvmesh/disks/<disk>/status | grep -E "Ldisk|access_local"
```

Output:
```json
"Ldisk": true,
"access_local": true,
```

### Check Local Server Connection

```bash
# Verify local server module loaded
lsmod | grep nvmeibs

# Check client registration
cat /proc/nvmesh/disks/<disk>/status | grep is_cl_reg
```

### Monitor Local I/O Statistics

```bash
# Local command counters
cat /proc/nvmesh/disks/<disk>/status | grep gen_cmds_cntrs_local
```

### Disable Local Bypass (for testing)

```bash
# Runtime disable
echo 0 > /sys/module/nvmeibc/parameters/use_local_bypass

# Requires rediscovery
echo 1 > /proc/nvmesh/disks/<disk>/rediscover
```

## Common Issues

1. **Local server not loaded**: Disk won't be detected as local
   - **Solution**: Ensure `nvmeibs` module loaded before `nvmeibc`

2. **Registration failure**: Can't access local disk
   - **Check**: Server has disk configured
   - **Check**: CID allocation succeeded

3. **Lock channel failure**: Distributed locks don't work
   - **Check**: Lock channel connected successfully
   - **Check**: Lock segments allocated

4. **PRPL allocation failure**: I/O fails with -ENOMEM
   - **Check**: DMA pool sizes (module parameters)
   - **Workaround**: Disable PRPL (`local_io_use_prpl = false`)

## Related Files

- `nvmeibs_*.c`: Local server implementation
- `nvmeibc_disk_local_dma_pools.c`: Local DMA pool management
- `nvmeib_local_disk.h`: Local disk interface definitions
- `nvmeibc_locks_channel.c`: Lock channel (used for local coordination)

