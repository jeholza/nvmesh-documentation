# TOMA (TOpology MAnager) Integration

## Overview

TOMA is a service that allows management operations to be communicated between the management layer (block/volume layer) and the target server. The client disk layer integrates with TOMA by:

1. **Subscribing**: Registering interest in specific volumes/segments
2. **Sending**: Transmitting management messages to target
3. **Receiving**: Processing management messages from target
4. **Reregistering**: Maintaining subscriptions across rediscovery

## TOMA Architecture

```
Block/Volume Layer
       |
       | (subscribe/send/recv)
       v
  Disk Layer (nvmeibc_disk.c)
       |
       | (admin channel)
       v
  Admin Channel (nvmeibc_ib_admin_channel.c)
       |
       | (RDMA messages)
       v
  Target Server (TOMA service)
```

## Key Data Structures

### TOMA Connection Entry

```c
enum toma_conn_subscribe_state {
    TOMA_NEED_SUBSCRIBED        = 1,  // Needs subscription
    TOMA_ALREADY_SUBSCRIBED     = 2,  // Successfully subscribed
    TOMA_ASYNC_SUBSCRIBE_SENT   = 3,  // Async subscribe pending
};

struct nvmeibc_toma_connection_hash_entry {
    u64 handle;                                     // Hash key (volume handle)
    nvmeibc_disk_toma_recv_req_callback_t *recv_req_cb;  // Receive callback
    u64 arg;                                        // Callback argument
    enum toma_conn_subscribe_state subscribed;      // Subscription state
    struct hlist_node hlist_next;                   // Hash table linkage
};
```

### TOMA Connection Hash

```c
struct nvmeibc_disk {
    DECLARE_HASHTABLE(toma_conn_hash, 8);  // 256 buckets
    rwlock_t toma_subscribe_lock;          // Protects subscription ops
    bool toma_subscribe_freeze;            // Freeze during rediscovery
};
```

### TOMA State in Admin Channel

```c
struct nvmeibc_admin_channel {
    struct {
        bool valid;  // True when subscriptions can be sent
    } toma;
};
```

## TOMA Operations

### 1. Subscribe

Register interest in a volume/segment:

```c
int nvmeibc_disk_subscribe_toma_service(struct nvmeibc_disk *disk, 
                                       u64 handle,
                                       struct nvmeibc_disk_subscription_params *params)
{
    struct nvmeibc_ib_admin_channel *ch = NULL;
    struct nvmeibc_toma_connection_hash_entry *entry;
    unsigned long flags;
    int subscribe_permission;
    int rv;
    
    // 1. Validate callback
    if (params->recv_req_cb == NULL)
        return -EINVAL;
    
    // 2. Acquire subscribe lock (blocks during rediscovery)
    subscribe_permission = disk_toma_subscribe_lock_return_permission(disk, &flags);
    
    // 3. Add to hash (even if can't subscribe yet)
    entry = nvmeibc_disk_toma_conn_hash_add(disk, handle, params);
    if (!entry) {
        rv = -ENOMEM;
        goto unlock;
    }
    
    // 4. Check if in middle of release/rediscovery
    if (subscribe_permission) {
        rv = -EAGAIN;  // Retry after rediscovery
        goto unlock;
    }
    
    // 5. Try to subscribe if admin channel available
    rv = -EAGAIN;  // Default: retry later
    
    if ((ch = get_alive_admin_ch(disk)) && ch->toma.valid) {
        // Send subscription
        if (nvmeibc_disk_use_async_subscribe) {
            entry->subscribed = TOMA_ASYNC_SUBSCRIBE_SENT;
        }
        
        if ((rv = toma_send_subscribe(entry, ch, 
                                      nvmeibc_disk_use_async_subscribe)) >= 0) {
            // Success
            if (!nvmeibc_disk_use_async_subscribe)
                entry->subscribed = TOMA_ALREADY_SUBSCRIBED;
        } else {
            // Failed - will retry on rediscovery
            rv = -EAGAIN;
        }
    }
    
unlock:
    disk_toma_subscribe_unlock(disk, &flags);
    return rv;
}
```

**Return values**:
- `0`: Successfully subscribed
- `-EAGAIN`: Deferred (will subscribe on next rediscovery) - NOT an error
- `-ENOMEM`: Fatal error (allocation failure)

**Subscription states**:
1. `TOMA_NEED_SUBSCRIBED`: Entry in hash, not yet subscribed
2. `TOMA_ASYNC_SUBSCRIBE_SENT`: Subscribe sent, awaiting completion
3. `TOMA_ALREADY_SUBSCRIBED`: Successfully subscribed

### 2. Unsubscribe

Unregister interest in a volume/segment:

```c
int nvmeibc_disk_unsubscribe_toma_service(struct nvmeibc_disk *disk, u64 handle)
{
    struct nvmeibc_ib_admin_channel *ch;
    struct nvmeibc_toma_connection_hash_entry *entry;
    int rv = -1;
    
    // 1. Lookup connection
    if (!(entry = nvmeibc_disk_toma_conn_hash_lookup(disk, handle)))
        goto out;
    
    // 2. Send unsubscribe if subscribed
    if (entry->subscribed == TOMA_ALREADY_SUBSCRIBED ||
        entry->subscribed == TOMA_ASYNC_SUBSCRIBE_SENT) {
        
        if (nvmeibc_disk_get_status(disk) == d_online) {
            if ((ch = get_alive_admin_ch(disk))) {
                // Send unsubscribe (async)
                rv = toma_send_cmd(entry, ch, NVMEIB_TOMA_CMD_UNREG, 
                                  NULL, true);
                
                if (rv >= 0) {
                    // Will delete from hash in completion callback
                    return 0;
                }
            }
        }
    }
    
    // 3. Delete from hash immediately if unsubscribe not sent
    nvmeibc_disk_toma_conn_hash_del(disk, handle);
    
out:
    return rv;
}
```

**When unsubscribe is sent**:
- Completion callback deletes hash entry
- Serialized on admin channel work queue

**When unsubscribe is not sent**:
- Hash entry deleted immediately
- Target will eventually timeout subscription

### 3. Send

Send management message to target:

```c
int nvmeibc_disk_toma_send(struct nvmeibc_disk *disk, u64 handle,
                           struct nvmeibc_disk_toma_send_params *params)
{
    struct nvmeibc_ib_admin_channel *ch;
    struct nvmeibc_toma_connection_hash_entry *entry;
    
    // 1. Check disk state
    if (atomic_read(&disk->dying) || atomic_read(&disk->paused))
        return -1;
    
    // 2. Get admin channel
    if (!(ch = get_alive_admin_ch(disk)))
        return -1;
    
    // 3. Verify connection subscribed
    if (!(entry = nvmeibc_disk_toma_conn_hash_lookup(disk, handle)))
        return -1;
    
    if (entry->subscribed == TOMA_NEED_SUBSCRIBED) {
        // Not yet subscribed
        return -1;
    }
    
    // 4. Send message (async)
    return toma_send_cmd(entry, ch, NVMEIB_TOMA_CMD_SEND, params, true);
}
```

**Message parameters**:
```c
struct nvmeibc_disk_toma_send_params {
    void *buf;      // Message buffer
    int len;        // Message length
};
```

### 4. Receive

Process message from target:

```c
void nvmeibc_disk_toma_recv(struct nvmeibc_disk *disk,
                            struct nvmeibc_toma_recv_msg *toma_recv_msg)
{
    struct nvmeibc_toma_connection_hash_entry *toma_conn_hent;
    u8 *buf = toma_recv_msg->buf;
    int len = toma_recv_msg->len;
    
    // 1. Lookup connection by handle
    toma_conn_hent = nvmeibc_disk_toma_conn_hash_lookup(disk, 
                                                        toma_recv_msg->handle);
    if (!toma_conn_hent)
        return;
    
    // 2. Validate callback
    if (toma_conn_hent->recv_req_cb == NULL)
        return;
    
    // 3. Invoke callback
    toma_conn_hent->recv_req_cb(
        (void*)nvmeibc_cinst_get_core_p(disk),  // Context
        toma_conn_hent->arg,                     // Callback arg
        buf,                                      // Message buffer
        len                                       // Message length
    );
}
```

**Called from**: Admin channel receive path when TOMA message arrives

## Rediscovery Handling

### The Challenge

TOMA subscriptions must survive disk rediscovery:
- Rediscovery destroys and recreates admin channel
- All TOMA subscriptions must be re-registered
- Upper layer (volume/block) should not be aware

### Subscribe Lock

Coordinates subscription with discovery/release:

```c
rwlock_t toma_subscribe_lock;
bool toma_subscribe_freeze;
```

**Lock protocol**:
```c
// Subscription path
read_lock(&disk->toma_subscribe_lock);
if (!disk->toma_subscribe_freeze) {
    // Can subscribe
} else {
    // Rediscovery in progress - defer
}
read_unlock(&disk->toma_subscribe_lock);

// Discovery/Release path
write_lock(&disk->toma_subscribe_lock);
disk->toma_subscribe_freeze = true;
// Perform discovery/release
disk->toma_subscribe_freeze = false;
write_unlock(&disk->toma_subscribe_lock);
```

### Reregistration Flow

After successful discovery:

```c
static int discover(struct nvmeibc_disk *disk, bool is_rediscover)
{
    // ... discovery stages ...
    
    // After admin channel connected
    ch->toma.valid = true;
    
    // ... more discovery ...
    
    // Before completing discovery
    if (is_rediscover) {
        toma_rereg_disk_handles(disk, ch, nvmeibc_disk_use_async_subscribe);
    }
    
    return 0;
}
```

### Reregistration Implementation

```c
static int toma_rereg_disk_handles(struct nvmeibc_disk *disk,
                                  struct nvmeibc_ib_admin_channel *ch,
                                  bool is_async)
{
    int bucket;
    struct hlist_node *h_node;
    struct nvmeibc_toma_connection_hash_entry *h_curr;
    int n_reg = 0, n_rereg = 0, n_async_reg = 0;
    
    // Iterate through all hash entries
    __hash_for_each_safe__(disk->toma_conn_hash, bucket, _, h_node, h_curr, hlist_next) {
        switch (h_curr->subscribed) {
        case TOMA_NEED_SUBSCRIBED:
            // New subscription, never sent before
            if (is_async)
                h_curr->subscribed = TOMA_ASYNC_SUBSCRIBE_SENT;
            
            rv = toma_send_subscribe(h_curr, ch, is_async);
            if (rv >= 0) {
                n_reg++;
                if (!is_async)
                    h_curr->subscribed = TOMA_ALREADY_SUBSCRIBED;
                else
                    n_async_reg++;
            }
            break;
            
        case TOMA_ALREADY_SUBSCRIBED:
        case TOMA_ASYNC_SUBSCRIBE_SENT:
            // Resubscribe after rediscovery
            if (is_async)
                h_curr->subscribed = TOMA_ASYNC_SUBSCRIBE_SENT;
            
            rv = toma_send_subscribe(h_curr, ch, is_async);
            if (rv >= 0) {
                n_rereg++;
                if (!is_async)
                    h_curr->subscribed = TOMA_ALREADY_SUBSCRIBED;
                else
                    n_async_reg++;
            } else {
                // Failed resubscribe - trigger release
                _NE(..., "Failed to resubscribe");
                nvmeibc_disk_start_release(disk, 
                    NVMEIBC_DISK_RELEASE_TOMA_SUBSCRIBE_FAILURE);
                return -1;
            }
            break;
        }
    }
    
    _NT(..., "TOMA rereg: new=%d, rereg=%d, async=%d", 
        n_reg, n_rereg, n_async_reg);
    
    return 0;
}
```

**Critical**: If reregistration fails, disk release is triggered to retry discovery.

## Synchronous vs Asynchronous Subscribe

### Synchronous (Legacy)

```c
nvmeibc_disk_use_async_subscribe = false;
```

- Subscribe waits for completion
- Blocks admin channel work queue
- Slower discovery
- Simpler state machine

### Asynchronous (Default)

```c
nvmeibc_disk_use_async_subscribe = true;
```

- Subscribe returns immediately
- Completion handled in callback
- Faster discovery
- More complex state machine

**Async state transitions**:
```
TOMA_NEED_SUBSCRIBED
    |
    v (send subscribe)
TOMA_ASYNC_SUBSCRIBE_SENT
    |
    v (completion callback)
TOMA_ALREADY_SUBSCRIBED
```

**Completion callback**:
```c
static void nvmeibc_disk_async_subscribe_toma_comp(struct nvmeibc_disk *disk, 
                                                   u64 handle, int status)
{
    struct nvmeibc_toma_connection_hash_entry *h;
    
    spin_lock(&disk->spinlock);
    
    h = nvmeibc_disk_toma_conn_hash_lookup_nolock(disk, handle);
    if (!h)
        goto out;
    
    if (!status) {
        // Success
        h->subscribed = TOMA_ALREADY_SUBSCRIBED;
    } else {
        // Failed - mark for retry
        h->subscribed = TOMA_NEED_SUBSCRIBED;
    }
    
out:
    spin_unlock(&disk->spinlock);
}
```

## TOMA Command Types

```c
enum nvmeib_toma_cmd_type {
    NVMEIB_TOMA_CMD_REG,      // Register/Subscribe
    NVMEIB_TOMA_CMD_UNREG,    // Unregister/Unsubscribe
    NVMEIB_TOMA_CMD_SEND,     // Send message
};
```

## Hash Table Management

### Add Connection

```c
static struct nvmeibc_toma_connection_hash_entry *
nvmeibc_disk_toma_conn_hash_add(struct nvmeibc_disk *disk, u64 handle,
                               struct nvmeibc_disk_subscription_params *params)
{
    struct nvmeibc_toma_connection_hash_entry *toma_conn_hent;
    unsigned long flags;
    
    spin_lock_irqsave(&disk->spinlock, flags);
    
    // Check if already exists
    if (nvmeibc_disk_toma_conn_hash_lookup_nolock(disk, handle)) {
        spin_unlock_irqrestore(&disk->spinlock, flags);
        return NULL;
    }
    
    // Allocate entry
    toma_conn_hent = kzalloc(sizeof(*toma_conn_hent), GFP_ATOMIC);
    if (!toma_conn_hent) {
        spin_unlock_irqrestore(&disk->spinlock, flags);
        return NULL;
    }
    
    // Initialize
    toma_conn_hent->handle = handle;
    toma_conn_hent->recv_req_cb = params->recv_req_cb;
    toma_conn_hent->arg = params->arg;
    toma_conn_hent->subscribed = TOMA_NEED_SUBSCRIBED;
    
    // Add to hash
    hash_add(disk->toma_conn_hash, &toma_conn_hent->hlist_next, handle);
    
    spin_unlock_irqrestore(&disk->spinlock, flags);
    return toma_conn_hent;
}
```

### Lookup Connection

```c
static struct nvmeibc_toma_connection_hash_entry *
nvmeibc_disk_toma_conn_hash_lookup(struct nvmeibc_disk *disk, u64 handle)
{
    struct nvmeibc_toma_connection_hash_entry *toma_conn_hent;
    unsigned long flags;
    
    spin_lock_irqsave(&disk->spinlock, flags);
    toma_conn_hent = nvmeibc_disk_toma_conn_hash_lookup_nolock(disk, handle);
    spin_unlock_irqrestore(&disk->spinlock, flags);
    
    return toma_conn_hent;
}
```

### Delete Connection

```c
static int nvmeibc_disk_toma_conn_hash_del(struct nvmeibc_disk *disk, u64 handle)
{
    struct nvmeibc_toma_connection_hash_entry *h_curr;
    unsigned long flags;
    bool found = false;
    
    spin_lock_irqsave(&disk->spinlock, flags);
    
    __hash_for_each_possible_safe__(disk->toma_conn_hash, h_curr, _, _, 
                                    hlist_next, handle) {
        if (h_curr->handle == handle) {
            hash_del(&h_curr->hlist_next);
            kfree(h_curr);
            found = true;
            break;
        }
    }
    
    spin_unlock_irqrestore(&disk->spinlock, flags);
    
    return found ? 0 : -1;
}
```

## Initialization and Cleanup

### Initialization

```c
static int nvmeibc_disk_toma_create(struct nvmeibc_disk *disk, bool is_rediscover)
{
    if (!is_rediscover) {
        // First discovery - initialize hash
        hash_init(disk->toma_conn_hash);
        rwlock_init(&disk->toma_subscribe_lock);
        disk->toma_subscribe_freeze = false;
    }
    
    return 0;
}
```

### Cleanup

```c
static void nvmeibc_disk_toma_free(struct nvmeibc_disk *disk)
{
    int bucket;
    struct hlist_node *h_node;
    struct nvmeibc_toma_connection_hash_entry *h_curr;
    
    // Free all hash entries
    __hash_for_each_safe__(disk->toma_conn_hash, bucket, _, h_node, 
                          h_curr, hlist_next) {
        hash_del(&h_curr->hlist_next);
        kfree(h_curr);
    }
}
```

## Debugging

### Enable TOMA Tracing

```bash
# Enable all TOMA-related traces
echo 1 > /sys/kernel/debug/tracing/events/nvmesh/*toma*/enable

# Watch TOMA operations
cat /sys/kernel/debug/tracing/trace_pipe | grep toma
```

### Check TOMA Status

```bash
cat /proc/nvmesh/disks/<disk>/status
```

Output includes:
```json
{
  "toma_connections": [
    {"handle": 12345, "state": "SUBSCRIBED"},
    {"handle": 12346, "state": "ASYNC_SUBSCRIBE_SENT"}
  ]
}
```

### Common Issues

#### 1. Subscribe Returns -EAGAIN

**Symptom**: Subscribe consistently returns `-EAGAIN`

**Causes**:
- Admin channel not yet connected
- Discovery in progress
- `ch->toma.valid` not set

**Solution**: Wait for discovery to complete, retry will happen automatically

#### 2. Messages Not Received

**Symptom**: Send succeeds but target doesn't respond

**Causes**:
- Not subscribed (state != TOMA_ALREADY_SUBSCRIBED)
- Receive callback not set
- Handle mismatch

**Solution**: Verify subscription state, check callback registration

#### 3. Subscription Lost After Rediscovery

**Symptom**: Messages stop working after rediscovery

**Causes**:
- Reregistration failed
- Hash entry lost

**Solution**: Check for release trigger due to reregistration failure

## Performance Considerations

### Hash Table Size

```c
DECLARE_HASHTABLE(toma_conn_hash, 8);  // 256 buckets
```

- Supports thousands of concurrent subscriptions
- O(1) lookup/add/delete for typical loads

### Asynchronous Operations

All TOMA operations are asynchronous to avoid blocking:
- Subscribe: Async (with async flag)
- Unsubscribe: Always async
- Send: Always async
- Receive: Callback from admin channel

## Module Parameters

```bash
# Enable async subscribe (default: true)
use_async_subscribe=true
```

## Related Documentation

- `DISK_DISCOVERY_REDISCOVERY.md`: TOMA initialization during discovery
- `DISK_UPDATES.md`: TOMA interaction with disk updates
- `nvmeibc_ib_admin_channel.c`: TOMA command transmission

## Related Files

- `nvmeibc_disk.c`: TOMA integration (this file)
- `nvmeibc_toma.h`: TOMA interface definitions
- `nvmeibc_ib_admin_channel.c`: TOMA protocol implementation
- Volume/block layer: TOMA API consumers

