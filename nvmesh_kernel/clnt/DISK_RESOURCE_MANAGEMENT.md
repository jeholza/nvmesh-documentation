# RDDA Resource Management and Lifecycle

## Overview

RDDA (Remote Disk Direct Access) channels require server-side resources including bounce buffers, queue pairs, and metadata buffers. The resource management system tracks these resources and allocates them to RDDA I/O channels on demand.

## Resource Types

### Disk Resources

```c
struct rsc_info {
    int id;              // Resource ID (index in array)
    struct list_head link;  // Link in my_rscs or free_rscs
};

struct nvmeibc_disk_info {
    struct rsc_info *rscs;       // Array of all resources
    int n_rscs;                  // Total resources from server
    int max_client_rscs;         // Max this client can use
    
    struct list_head my_rscs;    // Resources granted to this client
    int mine;                    // Count of granted resources
    
    struct list_head free_rscs;  // Unused resources
    int used;                    // Resources currently in use
};
```

### Channel Resource

```c
struct nvmeibc_disk_channel_rsc {
    u64 id;                      // Unique resource ID
    
    // Bounce buffer (for RDDA data transfer)
    u64 mem_raddr;               // Remote address
    u32 mem_size;                // Size in bytes
    u32 mem_lkey;                // Local key
    u32 mem_rkey;                // Remote key
    u32 mem_n_pages;             // Number of pages
    
    // Metadata buffer
    u64 md_raddr;                // Remote address
    u32 md_size;                 // Size in bytes
    u32 md_lkey;                 // Local key
    u32 md_rkey;                 // Remote key
    
    // Queue structures
    void *sq_shadow;             // Submission Queue shadow
    int sq_size;                 // SQ size in bytes
    int sq_n_entries;            // Number of entries
    
    void *cq_shadow;             // Completion Queue shadow
    int cq_size;                 // CQ size in bytes
    int cq_n_entries;            // Number of entries
    
    // PRPL (Physical Region Page List)
    u64 *prp1_shadow;            // PRP1 shadow
    u64 *prpl_phys;              // Physical page list
    
    // Associated channel
    struct nvmeibc_ib_io_channel *ch;  // NULL if not in use
    
    struct list_head link;       // Link in my_rscs list
};
```

## Resource Lifecycle

### 1. Discovery Phase

#### Request Resources

During discovery, client requests resources from server:

```c
static int request_disks_resources(struct nvmeibc_disk *disk)
{
    struct nvmeibc_admin_rnic *arnic;
    bool use_rdda;
    
    // Check if RDDA should be used
    use_rdda = !disk->is_local &&
               nvmeibc_cinst_get_core_p(disk)->use_rdda &&
               disk->info->max_client_rscs && 
               !disk->is_tcp;
    
    if (!use_rdda)
        return 0;  // No RDDA resources needed
    
    // Find main admin channel
    list_for_each_entry(arnic, &disk->arnics, link) {
        if (!nvmeibc_disk_use_arnic_for_disk(arnic, disk,
            NVMEIBC_ARNIC_ALIVE | NVMEIBC_ARNIC_HAS_CH))
            continue;
        
        // Request resources via admin channel
        rv = nvmeibc_ib_admin_channel_request_disks_resources(
                ac_to_iac(arnic->channel));
        
        if (rv >= 0) {
            // Success - mark as main channel
            arnic->channel->is_main = true;
            break;
        }
    }
    
    return rv;
}
```

#### Create Resource Structures

```c
int nvmeibc_disk_create_remote(struct nvmeibc_ib_admin_channel *ach,
                               int n_rscs, int max_client_rscs)
{
    struct nvmeibc_disk *disk = ach->base.base.disk;
    struct nvmeibc_disk_info *info;
    struct rsc_info *rscs;
    
    // Allocate resource array
    rscs = n_rscs ? kzalloc(sizeof(*rscs) * n_rscs, GFP_KERNEL) : NULL;
    
    // Initialize disk info
    info->rscs = rscs;
    info->n_rscs = n_rscs;
    info->max_client_rscs = max_client_rscs;
    info->mine = 0;
    info->used = 0;
    
    INIT_LIST_HEAD(&info->my_rscs);
    INIT_LIST_HEAD(&info->free_rscs);
    
    // Note: Resources not yet added to my_rscs
    // That happens via nvmeibc_disk_add_rsc()
}
```

#### Add Resource

For each resource, server sends details:

```c
int nvmeibc_disk_add_rsc(struct nvmeibc_ib_admin_channel *ach,
                         struct nvmeibs_disk_description *d,
                         struct lock_seg_info *lsi)
{
    struct nvmeibc_disk *disk = ach->base.base.disk;
    struct nvmeibc_disk_info *info = disk->info;
    int i = d->seq_num;
    struct nvmeibc_disk_channel_rsc *r;
    
    // Initialize resource structure
    r = &info->hcaa[0].channel_rscs[i];
    init_disk_rsc(i, r, d);
    
    // Allocate local structures (shadows, PRPL, etc.)
    if ((rv = alloc_disk_rsc(ach, i, r, lsi)) < 0)
        return rv;
    
    // Add to free list (not in use yet)
    list_add_tail(&r->link, &info->my_rscs);
    
    return 0;
}
```

### 2. Resource Allocation

#### Locate Resource

Server tells client which resources it can use:

```c
int nvmeibc_disk_locate_resource(struct nvmeibc_disk *disk, u64 id)
{
    unsigned long flags;
    unsigned i = id;
    
    spin_lock_irqsave(&disk->spinlock, flags);
    
    if (i < disk->info->n_rscs) {
        // Add to my_rscs list
        add_resource(disk, &disk->info->rscs[i]);
    } else {
        // Invalid resource ID
        rv = -1;
    }
    
    spin_unlock_irqrestore(&disk->spinlock, flags);
    return rv;
}

static void add_resource(struct nvmeibc_disk *disk, struct rsc_info *rsc)
{
    BUG_ON(!list_empty(&rsc->link));
    list_add_tail(&rsc->link, &disk->info->my_rscs);
    ++disk->info->mine;  // Increment available count
}
```

#### Get Resource (for channel)

When connecting an I/O channel:

```c
static struct rsc_info *get_resource(struct nvmeibc_disk *disk)
{
    struct rsc_info *rsc;
    
    // Get first available resource
    rsc = list_first_entry_or_null(&disk->info->my_rscs, struct rsc_info, link);
    if (rsc) {
        list_del_init(&rsc->link);
        ++disk->info->used;  // Mark as in use
    }
    
    return rsc;
}
```

#### Assign to Channel

```c
static struct nvmeibc_disk_channel_rsc *get_ch_rsc(
    struct nvmeibc_ib_admin_channel *ch,
    struct nvmeibc_ib_io_channel *ioch)
{
    struct nvmeibc_disk *disk = ch->base.base.disk;
    struct nvmeibc_disk_info *info = disk->info;
    struct nvmeibc_disk_channel_rsc *r = NULL;
    struct rsc_info *rsc;
    unsigned long flags;
    
    spin_lock_irqsave(&disk->spinlock, flags);
    
    // Get available resource
    rsc = get_resource(disk);
    if (!rsc) {
        // No resources available
        spin_unlock_irqrestore(&disk->spinlock, flags);
        return NULL;
    }
    
    // Map rsc_info to channel_rsc
    r = &info->hcaa[0].channel_rscs[rsc->id];
    
    // Associate with channel
    r->ch = ioch;
    ioch->rsc = r;
    
    spin_unlock_irqrestore(&disk->spinlock, flags);
    
    // Set up local mappings for this HCA
    set_local_keys(r, ch);
    
    return r;
}
```

### 3. Resource Usage

While channel is active:

```c
struct nvmeibc_ib_io_channel {
    struct nvmeibc_disk_channel_rsc *rsc;  // Assigned resource
    enum ib_io_channel_state state;         // IN_USE, ALLOCATED, etc.
};

// Resource is "in use" while channel state == IN_USE
// Resource remains allocated even when channel state == ALLOCATED
```

### 4. Resource Release

#### Put Resource (channel completes I/O)

```c
static void put_resource(struct nvmeibc_disk *disk, struct rsc_info *rsc)
{
    BUG_ON(!list_empty(&rsc->link));
    list_add_tail(&rsc->link, &disk->info->my_rscs);
    --disk->info->used;  // No longer in use
}
```

#### Kill Channel

When channel is disconnected:

```c
static void kill_channel_(struct nvmeibc_disk *disk,
                         struct nvmeibc_ib_io_channel *ch)
{
    struct nvmeibc_disk_info *info = disk->info;
    struct nvmeibc_disk_channel_rsc *r;
    
    r = ch->rsc;
    ch->rsc = NULL;
    
    if (r) {
        r->ch = NULL;
        
        // Return resource to available pool
        put_resource(disk, &info->rscs[get_r_index(info, r)]);
        
        // Remove channel from available list
        if (!list_empty(&ch->base.link))
            list_del_init(&ch->base.link);
    }
    
    if (ch->done)
        complete(ch->done);
}
```

### 5. Resource Deallocation

During disk release:

```c
static void free_disc_rscs(struct nvmeibc_ib_admin_channel *ch,
                          struct nvmeibc_disk_info *info)
{
    int i, j;
    
    // Free all resource sets
    for (j = 0; j < info->n_rscs_sets; ++j) {
        for (i = 0; i < info->n_rscs; ++i) {
            free_disc_rsc(ch, &info->hcaa[j].channel_rscs[i]);
        }
        kfree(info->hcaa[j].channel_rscs);
        kfree(info->hcaa[j].lsi);
    }
    kfree(info->rscs);
}

static void free_disc_rsc(struct nvmeibc_ib_admin_channel *ch,
                         struct nvmeibc_disk_channel_rsc *r)
{
    // Unmap DMA
    nvmeibc_dma_unmap(r);
    
    // Free allocated structures
    kfree(r->prp1_shadow);
    vfree(r->prpl_phys);
    kfree(r->sq_shadow);
    kfree(r->cq_shadow);
}
```

## Resource State Diagram

```
[Server Advertises Resources]
         |
         v
[Client Allocates Structures] (nvmeibc_disk_create_remote)
         |
         v
[Server Grants Resources] (nvmeibc_disk_locate_resource)
         |
         v
[Resources in my_rscs] (mine++)
         |
         |<--------------------+
         v                     |
[Get Resource] --------> [Resource Used]
(mine--, used++)         (associated with channel)
         ^                     |
         |                     v
         |              [Put Resource]
         +-----------------(mine++, used--)
         
[Disk Release]
         |
         v
[Free Resources] (free_disc_rscs)
```

## Resource Counters

### Key Counters

```c
struct nvmeibc_disk_info {
    int mine;    // Resources available for use
    int used;    // Resources currently assigned to channels
};

// Invariant: mine + used <= max_client_rscs
```

### Counter Updates

| Operation | mine | used | Notes |
|-----------|------|------|-------|
| locate_resource() | +1 | 0 | Server grants resource |
| get_resource() | -1 | +1 | Assign to channel |
| put_resource() | +1 | -1 | Channel releases resource |
| kill_channel() | +1 | -1 | Channel disconnected |

## Watermark System

Resources control RDDA vs NoRDDA channel selection:

```c
#define NVMEIBC_WATERMARK_GOTO_NORDDA  0

if (info->mine > NVMEIBC_WATERMARK_GOTO_NORDDA) {
    // Enough RDDA resources - try RDDA first
    try_rdda_channel();
} else {
    // Low on RDDA resources - use NoRDDA
    try_nordda_channel();
}
```

**Strategy**: When RDDA resources are scarce, switch to NoRDDA to avoid resource exhaustion.

## DMA Mapping

Resources include DMA-mapped buffers:

```c
static int set_local_keys(struct nvmeibc_disk_channel_rsc *r,
                         struct nvmeibc_ib_admin_channel *ch)
{
    struct nvmeib_mem_map map;
    
    // Map bounce buffer for this HCA
    map.dev = ch->net.base.port->nic_dev->dev;
    map.pd = ch->net.base.port->nic_dev->dev->pd;
    map.vaddr = kmalloc(r->mem_size, GFP_KERNEL);
    map.size = r->mem_size;
    map.access_flags = IB_ACCESS_LOCAL_WRITE |
                      IB_ACCESS_REMOTE_READ |
                      IB_ACCESS_REMOTE_WRITE;
    
    rv = nvmeib_mem_alloc_n_map(&map);
    
    r->mem_laddr = map.ioaddr;
    r->mem_lkey = map.lkey;
    // r->mem_rkey already set by server
    
    return rv;
}
```

## Resource Cleanup

### Normal Cleanup

During graceful disk release:

```c
// 1. Disconnect all RDDA channels
for_each_ioch(disk, ch) {
    kill_channel(disk, ch);  // Returns resources
}

// 2. Free resource structures
free_disc_rscs(ch, disk->info);

// 3. Free disk info
kfree(disk->info);
disk->info = NULL;
```

### Error Cleanup

If resource allocation fails:

```c
if (alloc_disk_rsc(...) < 0) {
    // Free partially allocated resources
    for (i = 0; i < allocated_count; i++) {
        free_disc_rsc(ch, &resources[i]);
    }
    return -ENOMEM;
}
```

## Resource Limits

### Server-Side Limits

```c
max_client_rscs;  // Maximum resources this client can use
```

Set by server based on:
- Available server memory
- Number of connected clients
- Client priority/QoS

### Client-Side Limits

```c
// Maximum RDDA channels per disk
// Limited by available resources
int max_rdda_channels = min(max_client_rscs, 
                           nr_max_channels_per_disk);
```

## Debugging

### Check Resource State

```bash
cat /proc/nvmesh/disks/<disk>/status
```

Output:
```json
{
  "info": {
    "tot_d_rscs": 16,
    "max_c_rscs": 8,
    "mine": 5,
    "used": 3,
    "available_channels": 5
  }
}
```

**Interpretation**:
- `tot_d_rscs=16`: Server has 16 total resources
- `max_c_rscs=8`: This client can use up to 8
- `mine=5`: 5 resources currently granted and available
- `used=3`: 3 resources assigned to channels
- `available_channels=5`: 5 RDDA channels available (some may share resources)

### Resource Exhaustion

```
[nvmesh] Disk vol1: No resources available for RDDA channel
```

**Causes**:
- All `max_client_rscs` resources in use
- Other clients using resources
- Server resource exhaustion

**Solution**:
- Use NoRDDA channels (automatic fallback)
- Request more resources from server
- Disconnect unused RDDA channels

## Local Disks

Local disks do NOT use RDDA resources:

```c
if (disk->access_local) {
    // No resources needed
    disk->info = NULL;
    return 0;
}
```

**Reason**: Local I/O uses direct NVMe submission, no bounce buffers needed.

## Related Documentation

- `CHANNEL_MANAGEMENT.md`: How resources affect channel selection
- `DISK_DISCOVERY_REDISCOVERY.md`: Resource request during discovery
- `LOCAL_BYPASS.md`: Why local disks don't need resources

## Related Files

- `nvmeibc_disk.c`: Resource management implementation
- `nvmeibc_ib_io_channel.c`: RDDA channel using resources
- `nvmeibc_ib_admin_channel.c`: Resource request protocol

