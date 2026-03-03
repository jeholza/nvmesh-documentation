# Client-Side Admin Channel Documentation

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Data Structures](#data-structures)
4. [Connection Lifecycle](#connection-lifecycle)
5. [Message Types and Opcodes](#message-types-and-opcodes)
6. [Configuration Commands](#configuration-commands)
7. [Keep-Alive Mechanism](#keep-alive-mechanism)
8. [TOMA Integration](#toma-integration)
9. [Resource Management](#resource-management)
10. [Error Handling](#error-handling)

---

## Overview

The **Admin Channel** is a critical control-plane component in the NVMesh client architecture. It serves as the primary communication channel between a client node and a target's admin NIC (RANIC - Remote Admin NIC). The admin channel is responsible for:

- **Initial login and authentication** with target nodes
- **Resource discovery and allocation** (IO NICs, disks, memory regions)
- **Configuration exchange** (shared constants, RDMA info, journal ranges)
- **TOMA (Topology Manager) communication** for volume management and recovery operations
- **Keep-alive monitoring** to detect connection failures
- **Coordination of IO channels** (both RDDA and No-RDDA)
- **Lock management** for segment locking

### Key Characteristics

- **One admin channel per disk**: Each client maintains one "main" admin channel per remote disk
- **Transport**: RDMA over InfiniBand or TCP
- **Message limit**: Up to 128 concurrent admin messages (configurable via `NVMEIBC_CHANNEL_MAX_MAIN_ADMIN_MSGS`)
- **Work queue**: Dedicated work queue for admin channel operations with high/low priority support

---

## Architecture

### Component Hierarchy

```
nvmeibc_admin_channel (Base)
    ├── nvmeibc_ib_admin_channel (InfiniBand-specific)
    │   ├── nvmeibc_ib_net_admin (Network layer)
    │   ├── nvmeibc_ib_admin_channel_toma (TOMA subsystem)
    │   └── Keep-Alive subsystem
    └── Remote IO NICs (rionics)
        └── Local IO NICs (lionics)
            ├── RDDA IO Channels
            └── No-RDDA Channels
```

### File Organization

| File | Purpose |
|------|---------|
| `nvmeibc_admin_channel.h/c` | Base admin channel implementation |
| `nvmeibc_ib_admin_channel.h/c` | InfiniBand-specific admin channel |
| `nvmeibc_ib_net_admin.h/c` | Admin network layer |
| `nvmeibc_toma.h/c` | TOMA message handling |
| `nvmeibc_msgs_shared.h` | Shared message definitions |

---

## Data Structures

### struct nvmeibc_admin_channel

The base admin channel structure (defined in `nvmeibc_admin_channel.h`):

```c
struct nvmeibc_admin_channel {
    /* Base channel */
    struct nvmeibc_channel base;
    
    /* Remote admin NIC */
    struct nvmeibc_admin_rnic *arnic;
    
    /* Version information */
    union nvmeib_version link_version;
    u64 cid;  /* Controller cookie for us */
    
    /* Remote system info */
    int cntr_page_size;
    int n_disks;
    
    /* Remote IO NICs and disks */
    struct list_head rionics;  /* List of struct nvmeibc_io_rnic */
    int n_rionics_used;
    
    /* Main channel flag */
    bool is_main;  /* True if we allocated resources through it */
    
    /* Work queue for channel operations */
    struct workq_struct *remove_wq;
    atomic_t wq_high_pri_cnt;
    
    /* Segment locks */
    struct nvmeibc_disk_segments_locks segments_locks_remote;
    
    /* Periodic callbacks (e.g., keep-alive) */
    spinlock_t periodics_guard;
    struct list_head periodics;
    
    /* Version extension operations */
    const struct vex_ops *vex_ops[vex_ach_ops_num];
    const struct vex_ops *vex_nrio_ops[vex_nrch_ops_num];
};
```

### struct nvmeibc_ib_admin_channel

InfiniBand-specific admin channel (defined in `nvmeibc_ib_admin_channel.h`):

```c
struct nvmeibc_ib_admin_channel {
    /* Base channel */
    struct nvmeibc_admin_channel base;
    
    /* Remote admin NIC identification */
    char ranic_guid[GUID_SIZE];
    union ib_gid ranic_gid;
    
    /* Network layer */
    struct nvmeibc_ib_net_admin net;
    
    /* Transmit ring */
    int tx_ring_size;  /* Default: 128 */
    struct nvmeib_iu **tx_ring;
    struct list_head free_tx;
    struct list_head uncomp_tx;
    
    /* Send completion synchronization */
    struct completion send_done;
    spinlock_t guard;
    
    /* Receive handling */
    struct list_head recv_ioctx;
    struct nvmeib_iu *send_ioctx;
    struct nvmeib_recvq *recv_q;
    
    /* Message areas */
    void *msg_area;
    void *msg_area_end;
    struct nvmeib_alloc_n_map msg_area_map;
    
    /* Keep-alive */
    spinlock_t ka_spinlock;
    struct nvmeib_hdr *ka_msg_area;
    u64 ka_msg_dma_addr;
    bool ka_sent;
    bool ka_running;
    TIMER_LIST_INSTANCE(ka_timer);
    unsigned long ka_start_time;
    
    /* Controller message buffer info */
    u64 cmsg_buffer_raddr;
    u32 cmsg_buffer_pages;
    u32 cmsg_buffer_rkey;
    u32 cmsg_buffer_page_shift;
    
    /* Lock segment info */
    struct lock_seg_info *lsi;
    bool lsi_alloc;
    
    /* TOMA subsystem */
    struct nvmeibc_ib_admin_channel_toma toma;
    
    /* Logout handling */
    struct completion logout_comp;
    bool logout;
    enum nvmeibc_disk_release_reason release_reason;
};
```

### struct nvmeibc_ib_admin_channel_toma

TOMA (Topology Manager) message handling subsystem:

```c
struct nvmeibc_ib_admin_channel_toma {
    /* Valid flag - prevents sending before control sequence completes */
    bool valid;
    
    /* Callbacks */
    nvmeibc_disk_async_subscribe_toma_comp_callback *async_subscribe_comp_cb;
    nvmeibc_disk_unsubscribe_toma_comp_callback_t *unsubscribe_comp_cb;
    
    /* Receive message reassembly */
    struct nvmeibc_toma_recv_msg recv_msg;
};
```

---

## Connection Lifecycle

### 1. Creation and Initialization

**Entry Point**: `nvmeibc_ib_admin_channel_create()`

```c
struct nvmeibc_ib_admin_channel *nvmeibc_ib_admin_channel_create(
    const struct nvmeibc_cinst_params_core *p, 
    struct nvmeibc_admin_rnic *arnic);
```

**Steps**:
1. Allocate `struct nvmeibc_ib_admin_channel`
2. Initialize from `arnic` (remote admin NIC info):
   - Copy remote GID
   - Set service ID and port
   - Set destination GID for path
3. Call `nvmeibc_admin_channel_init()`:
   - Initialize base channel
   - Create dedicated work queue for admin operations
   - Initialize periodic callbacks list
   - Initialize segment locks

### 2. Connection Establishment

**Entry Point**: `nvmeibc_ib_admin_channel_connect()`

```c
int nvmeibc_ib_admin_channel_connect(
    struct nvmeibc_ib_admin_channel *ch,
    struct nvmeibc_disk *disk, 
    struct nvmeibc_admin_rnic *arnic);
```

**Flow**:

```
1. initialize_net()
   └── Set local/remote keys for RDMA

2. login()
   ├── Prepare login request
   │   ├── Set channel type: NVMEIBC_ADMIN_CHANNEL
   │   ├── Set max IU length (message size)
   │   ├── Include client UUID
   │   └── Set version info
   │
   ├── nvmeibc_ib_net_admin_alloc()
   │   ├── Create QP (Queue Pair)
   │   ├── Setup send/receive queues
   │   │   ├── Send queue: 128 entries
   │   │   ├── Receive queue: 0 (uses SRQ) or configured
   │   │   └── Max outstanding reads: 16
   │   ├── Register callbacks:
   │   │   ├── on_login() → on_login_admin_ch()
   │   │   ├── on_reject() → on_reject_admin_ch()
   │   │   ├── on_disconnect() → on_disconnect_admin_ch()
   │   │   ├── send_completion() handler
   │   │   └── recv_completion() handler
   │   └── Connect to target
   │
   ├── alloc_msg_area()
   │   ├── Allocate control message area
   │   └── Allocate keep-alive message area
   │
   ├── alloc_iu_bufs()
   │   └── Allocate transmit IU buffers (128)
   │
   └── keep_alive_start()
       └── Start periodic keep-alive timer

3. share_config() [if not access_local]
   └── Exchange shared configuration constants

4. read_io() [if not access_local]
   ├── Request IO NIC information
   ├── Allocate IO resources
   └── Setup IO channels
```

### 3. Login Callback: on_login_admin_ch()

Called when login response is received from target:

```
1. Parse login response
   ├── Extract controller ID (cid)
   ├── Extract link version
   ├── Extract message buffer info:
   │   ├── Remote address (cmsg_buffer_raddr)
   │   ├── Remote key (cmsg_buffer_rkey)
   │   ├── Number of pages
   │   └── Page shift
   └── Extract controller page size

2. Setup message areas
   └── Map remote message buffer for RDMA access

3. Setup keep-alive
   └── Configure keep-alive RDMA write parameters

4. Initialize TOMA subsystem
   └── nvmeibc_toma_init()
```

### 4. Disconnection

**Entry Point**: `nvmeibc_ib_admin_channel_disconnect()`

```
1. Stop keep-alive timer

2. nvmeibc_ib_net_disconnect()
   ├── Set dying flag
   ├── Complete pending send operations
   └── Close QP connection

3. Wait for IO channels to drain
   ├── Wait for RDDA channels
   └── Wait for No-RDDA channels

4. Free admin channel resources
   ├── Drain work queue
   ├── Free network resources
   ├── Free message areas
   └── Free TOMA resources
```

---

## Message Types and Opcodes

### High-Level Message Categories

The admin channel handles several categories of messages:

1. **Login/Logout** - Connection establishment
2. **Configuration** - Resource and topology exchange
3. **TOMA** - Topology manager communication
4. **Keep-Alive** - Connection monitoring
5. **Commands** - Server-initiated requests
6. **Responses** - Replies to client requests
7. **Debug** - Debugging and diagnostics

### Message Opcodes (from nvmeib_types.h)

#### Receive Message Types (hdr->opcode)

Messages received by the client from the target:

```c
switch (hdr->opcode) {
    case NVMEIB_RSP:
        /* Response to a client request */
        process_rsp(ch, iu);
        break;
        
    case NVMEIB_CMD:
        /* Command from server to client */
        process_req(ch, iu);
        break;
        
    case NVMEIB_TOMA_REQ:
        /* TOMA management request */
        process_toma_req(ch, iu);
        break;
        
    case NVMEIB_H_LOGOUT:
        /* Target-initiated logout */
        break;
        
    case NVMEIB_KEEP_ALIVE:
        /* Keep-alive message */
        nvmeibc_admin_channel_exec_periodic(&ch->base, jiffies);
        keep_alive_process_req(ch, hdr->tag);
        break;
        
    case NVMEIB_DBG_CMD:
        /* Debug command */
        break;
}
```

#### Server Commands (NVMEIB_CMD sub-types)

Server-initiated commands that clients must handle:

```c
enum nvmeibs_cmd_opcode {
    NVMEIBS_GET_RSC,          /* Request to return resources */
    NVMEIBS_PUT_RSC,          /* Server provides resources */
    NVMEIBS_JAM_ABND2FREE,    /* Abort abandoned resource */
    NVMEIBS_RGID_CHANGE,      /* Remote GID change notification */
    NVMEIBS_IOCH_DRAINED,     /* IO channel drained notification */
    /* ... others ... */
};
```

#### Send Message Types (wr_opcode)

Work request opcodes for messages sent by client:

```c
enum nvmeib_wr_opcode {
    NVMEIB_SEND_CFG,          /* Configuration message */
    NVMEIB_TOMA_SEND_REQ,     /* TOMA request to server */
    NVMEIB_TOMA_SEND_RSP,     /* TOMA response to server */
    NVMEIB_RDMA_GET_JMDC_REQ, /* Get journal metadata cache */
    NVMEIB_WR_DBG_CMD,        /* Debug command */
    NVMEIB_KEEP_ALIVE_REQ,    /* Keep-alive request */
    /* ... others ... */
};
```

### Message Send Flow

**Entry Point**: `nvmeibc_ib_admin_channel_prepare_n_send_msg()`

```c
int nvmeibc_ib_admin_channel_prepare_n_send_msg(
    struct nvmeibc_ib_admin_channel *ch,
    ssize_t (*f)(void *p, void *buf, const void *buf_end),  /* Message builder */
    void *p,                /* Context for builder */
    int wr_opcode,          /* Work request opcode */
    struct nvmeib_iu *iu);  /* IO unit buffer */
```

**Flow**:
```
1. Get TX IU buffer
   └── nvmeibc_ib_admin_channel_get_tx_iu()

2. Sync for CPU access
   └── ib_dma_sync_single_for_cpu()

3. Build message
   └── Call builder function f()

4. Initialize send completion
   └── ach_send_done_on_init()
       ├── Reinit completion
       └── Check not dying

5. Sync for device access
   └── ib_dma_sync_single_for_device()

6. Send message
   └── send_msg()
       └── Post send work request

7. Wait for completion
   └── wait_for_completion_interruptible_timeout()
       └── Timeout: NVMEIB_WAIT_FOR_ADMIN_SEND_COMP (30 sec default)

8. Check status
   └── Return success/error based on IU status
```

---

## Configuration Commands

Configuration commands are sent via `NVMEIB_CONFIG` messages. The opcode field specifies the type of configuration operation.

### Configuration Operation Types

```c
enum nvmeibc_config_ops {
    NVMEIBC_MA_SHARE_CONFIG  = 0x00,  /* Share configuration constants */
    NVMEIBC_MA_GET_IO        = 0x01,  /* Get IO NIC information */
    NVMEIBC_MA_GET_ACCESS    = 0x02,  /* Get access map */
    NVMEIBC_MA_ALLOC_IO_NET  = 0x03,  /* Allocate IO network resources */
    NVMEIBC_MA_LOCATE_RSC    = 0x04,  /* Locate/allocate disk resources */
    NVMEIBC_MA_RESET_RSC     = 0x05,  /* Reset/free resources */
    NVMEIBC_MA_GET_DISK_MEMS = 0x07,  /* Get disk memory regions */
    NVMEIBC_MA_ALLOC_NR_NET  = 0x08,  /* Allocate No-RDDA network */
    NVMEIBC_MA_GET_LOCK_GIDS = 0x09,  /* Get lock channel GIDs */
    NVMEIBC_MA_GET_JRANGE    = 0x10,  /* Get journal range info */
};
```

### 1. Share Configuration (NVMEIBC_MA_SHARE_CONFIG)

**Purpose**: Exchange shared configuration constants between client and target.

**Function**: `share_config()` in `nvmeibc_ib_admin_channel.c`

**Message Structure**:
```c
struct volume_client_config_share {
    struct volume_client_config_share_base {
        __be64 version;
        struct nvmeib_container share_const_ctnr;
        struct volume_client_config_share_const_elem share_consts[];
    } base;
};
```

**Response**: Server replies with its shared constants.

### 2. Get IO NIC Info (NVMEIBC_MA_GET_IO)

**Purpose**: Discover remote IO NICs and their network information.

**Function**: `nvmeibc_ib_admin_channel_access_iornics()`

**Message Structure**:
```c
struct volume_client_config_ma_get_io {
    struct volume_client_config_ma_get_io_base {
        char client_name[NVMEIB_BASE_CLIENT_NAME_SIZE];
        char disk_name[NVMEIB_DISK_MAX_NVMEXPRESS_ID_SIZE];
        struct volume_client_config_rdma_info rdma;
        __be64 keep_alive_raddr;
        /* ... */
    } base;
};
```

**Response**: Server provides:
- List of remote IO NICs (RIONICs)
- GID information for each NIC
- Network topology

### 3. Locate Resources (NVMEIBC_MA_LOCATE_RSC)

**Purpose**: Request allocation of disk resources (for IO operations).

**Function**: `nvmeibc_ib_admin_channel_request_disks_resources()`

**Flow**:
```
1. Send LOCATE_RSC request
   ├── Specify disk name
   └── Include client identification

2. Server allocates resources
   ├── Assign resource IDs
   └── Reserve IO slots

3. Client receives resource list
   └── nvmeibc_disk_locate_resource()
       └── Store resource IDs for future use
```

**Key Detail**: 
- Max resources per client: `NVMEIB_MAX_DISK_RESOURCES_PER_CLIENT` (128 by default)
- Resources are used for RDDA and No-RDDA IO operations

### 4. Get Journal Range (NVMEIBC_MA_GET_JRANGE)

**Purpose**: Retrieve journal range information for erasure coding.

**Function**: `nvmeibc_ib_admin_channel_get_journal_range()`

**Message Structure**:
```c
struct volume_client_config_get_jrange {
    struct volume_client_config_get_jrange_base {
        char client_name[NVMEIB_BASE_CLIENT_NAME_SIZE];
        char disk_name[NVMEIB_DISK_MAX_NVMEXPRESS_ID_SIZE];
        struct volume_client_config_rdma_info rdma;
        uuid_be client_uuid;
    } base;
    struct volume_client_config_get_jrange_ext1 ext1;
    struct volume_client_config_get_jrange_ext2 ext2;
    struct volume_client_config_get_jrange_ext3 ext3;
};
```

**Response**: Contains journal range metadata, entry information, and RDMA access details.

### 5. Allocate IO Network (NVMEIBC_MA_ALLOC_IO_NET)

**Purpose**: Allocate network resources for IO channels (RDDA).

**Function**: `nvmeibc_ib_admin_channel_init_io_channel()`

**Parameters**:
- Disk resource ID
- MSI-X table address and payload (for interrupts)
- QP information

### 6. Get Lock GIDs (NVMEIBC_MA_GET_LOCK_GIDS)

**Purpose**: Retrieve GID information for lock-manager channels.

**Function**: `nvmeibc_ib_admin_channel_connect_lock_lb_channel()`

**Used For**: Setting up segment locking channels for coordinated access to disk regions.

---

## Keep-Alive Mechanism

The keep-alive mechanism ensures the admin channel connection remains healthy and detects failures quickly.

### Configuration

```c
/* Keep-alive timeout (from nvmeib_types.h) */
#ifndef LOW_MEM
#define NVMEIB_KEEP_ALIVE_TO (6 * HZ)   /* 6 seconds */
#else
#define NVMEIB_KEEP_ALIVE_TO (40 * HZ)  /* 40 seconds for low memory */
#endif
```

### Keep-Alive Structure

```c
struct nvmeib_keep_alive {
    u8 value;       /* Heartbeat value */
    u32 lkey;       /* Local key */
    u64 raddr;      /* Remote address */
    u32 size;       /* Buffer size */
    u32 rkey;       /* Remote key */
};
```

### Keep-Alive Flow

#### Initialization: `keep_alive_start()`

```
1. Allocate keep-alive message buffer
   └── ch->ka_msg_area

2. Map for DMA
   └── ch->ka_msg_dma_addr

3. Setup RDMA parameters
   ├── Remote address (from login response)
   ├── Remote key
   └── Size

4. Register periodic callback
   └── nvmeibc_admin_channel_add_periodic()
       └── Called by keep-alive timer

5. Start timer
   └── Fires every NVMEIB_KEEP_ALIVE_TO
```

#### Keep-Alive Send: `keep_alive_send_req()`

```
1. Check if already in flight
   └── Return if ka_sent == true

2. Prepare keep-alive message
   ├── Set opcode: NVMEIB_KEEP_ALIVE
   ├── Set tag
   └── Increment value

3. Send via RDMA WRITE
   └── Posts RDMA_WRITE work request
       ├── Source: ch->ka_msg_dma_addr
       ├── Destination: controller msg buffer
       └── Opcode: NVMEIB_KEEP_ALIVE_REQ

4. Mark as sent
   └── ch->ka_sent = true

5. Start timeout timer
   └── If no response, disconnect
```

#### Keep-Alive Response Processing

```
Server sends NVMEIB_KEEP_ALIVE message back
    └── handle_recv() processes it
        ├── Execute periodic callbacks
        │   └── nvmeibc_admin_channel_exec_periodic()
        └── keep_alive_process_req()
            ├── Mark ka_sent = false
            ├── Cancel timeout timer
            └── Update last_response_time
```

#### Timeout Handling

```
If keep-alive timer expires:
    └── keep_alive_timeout()
        ├── Log error
        ├── Mark channel as dying
        └── Initiate disconnect
            └── nvmeibc_ib_net_disconnect()
```

### Periodic Callback Registration

The admin channel supports registering periodic callbacks that are invoked during keep-alive processing:

```c
struct admin_periodic {
    /* Callback function */
    int (*on_periodic)(void *arg, unsigned long t);
    void *on_periodic_arg;
    
    /* Removal function */
    void (*remove_periodic)(struct nvmeibc_admin_channel *ch,
                           struct admin_periodic *p);
    
    /* Linked list */
    struct nvmeibc_admin_channel *ch;
    struct list_head link;
};

/* Registration */
void nvmeibc_admin_channel_add_periodic(
    struct nvmeibc_admin_channel *ch,
    struct admin_periodic *p);
```

**Use Cases**:
- IO channel health monitoring
- Resource cleanup
- Statistics collection

---

## TOMA Integration

TOMA (Topology Manager) is the volume management and orchestration component. The admin channel provides the transport for TOMA messages.

### TOMA Message Types

```c
enum nvmeib_toma_cmd_type {
    NVMEIB_TOMA_CMD_SEND,        /* Send request to TOMA */
    NVMEIB_TOMA_CMD_SUBSCRIBE,   /* Subscribe to events */
    NVMEIB_TOMA_CMD_UNSUBSCRIBE, /* Unsubscribe from events */
};
```

### TOMA Message Handling

#### Sending TOMA Requests

**Function**: `nvmeibc_toma_send_req()`

```
1. Check message size
   └── Max: NVMEIB_TOMA_REQ_MAX_LEN (3248 bytes)

2. Get TX IU buffer
   └── nvmeibc_ib_admin_channel_get_tx_iu()

3. Build TOMA message
   └── prp_toma_req()
       ├── Set opcode: NVMEIB_TOMA_REQ
       ├── Set command type
       ├── Copy payload
       └── Set tag

4. Send message
   └── nvmeibc_ib_admin_channel_prepare_n_send_msg()
       └── wr_opcode: NVMEIB_TOMA_SEND_REQ

5. Invoke send completion callback
   └── toma_cmd->send_params->send_comp_cb()

6. Wait for response
   └── Poll for NVMEIB_TOMA_REQ response
```

#### Receiving TOMA Messages

**Function**: `process_toma_req()` and `nvmeibc_toma_recv_req()`

```
1. Receive TOMA message fragment
   └── handle_recv() → process_toma_req()

2. Reassemble fragments
   └── nvmeibc_toma_recv_req()
       ├── Check handle for multi-part message
       ├── Allocate reassembly buffer if needed
       ├── Append fragment to buffer
       └── Mark complete when all parts received

3. Process complete message
   └── Invoke TOMA handler
       └── Volume topology updates
       └── Recovery commands
       └── Configuration changes

4. Send response
   └── nvmeibc_ib_admin_send_rsp()
       └── wr_opcode: NVMEIB_TOMA_SEND_RSP
```

### TOMA Subscription

Clients can subscribe to TOMA events (e.g., topology changes):

```c
/* Subscribe callback */
ch->toma.async_subscribe_comp_cb = disk_subscribe_callback;

/* Unsubscribe callback */
ch->toma.unsubscribe_comp_cb = disk_unsubscribe_callback;
```

### TOMA Message Fragmentation

Large TOMA messages are fragmented:

```c
struct nvmeibc_toma_recv_msg {
    u64 handle;         /* Message handle for reassembly */
    u8 *buf;            /* Reassembly buffer */
    int len;            /* Current length */
    bool complete;      /* All fragments received */
    
    /* Response */
    struct volume_client_rsp *rsp;
    size_t rsp_sz;
    u64 rsp_dma_addr;
};
```

**Max Fragment Size**: Determined by `NVMEIBS_MAX_ADMIN_MSG_SIZE`

---

## Resource Management

The admin channel manages allocation and tracking of various resources.

### Resource Types

1. **Admin Resources**
   - Message buffers (IU - IO Units)
   - RDMA memory regions
   - Keep-alive buffers

2. **IO Resources** (managed via admin channel)
   - Disk resource IDs
   - IO channel slots
   - Journal ranges (for EC)

3. **Network Resources**
   - Remote IO NICs (RIONICs)
   - Local IO NICs (LIONICs)
   - QP connections

### IU (IO Unit) Management

#### TX (Transmit) IUs

```c
/* Get transmit IU */
struct nvmeib_iu *nvmeibc_ib_admin_channel_get_tx_iu(
    struct nvmeibc_ib_admin_channel *ch);

/* Return transmit IU */
void nvmeibc_ib_admin_channel_put_tx_iu(
    struct nvmeibc_ib_admin_channel *ch,
    struct nvmeib_iu *iu);
```

**Pool Size**: `NVMEIBC_CHANNEL_MAX_MAIN_ADMIN_MSGS` (128)

**Management**:
- Free list: `ch->free_tx`
- Uncompleted list: `ch->uncomp_tx` (for error cleanup)
- Protected by: `ch->guard` spinlock

#### RX (Receive) IUs

```c
/* Get receive IU by index */
struct nvmeib_iu *nvmeibc_ib_admin_channel_get_rx_iu(
    struct nvmeibc_ib_admin_channel *ch, 
    int index);

/* Return receive IU (repost to receive queue) */
int nvmeibc_ib_admin_channel_put_rx_iu(
    struct nvmeibc_ib_admin_channel *ch,
    struct nvmeib_iu *iu);
```

**Sources**:
- Shared Receive Queue (SRQ) - preferred
- Private receive queue - fallback

### RIONIC/LIONIC Management

**RIONIC** (Remote IO NIC): Represents a remote (target-side) IO NIC.

**LIONIC** (Local IO NIC): Represents a local (client-side) IO NIC connected to a RIONIC.

```
Admin Channel
    └── rionics (list)
        └── RIONIC
            ├── lionics (list)
            │   └── LIONIC
            │       ├── io_channels[] (RDDA)
            │       └── nr_channels[] (No-RDDA)
            └── disk_link (to disk)
```

**Reference Counting**:
- `n_rionics_used`: Count of RIONICs in use
- If count reaches 0, admin channel can be closed

### Work Queue Management

The admin channel has a dedicated work queue for operations:

```c
/* High priority work */
int nvmeibc_admin_channel_add_work(
    struct nvmeibc_admin_channel *ch,
    struct workqe_struct *work);

/* Low priority work */
int nvmeibc_admin_channel_add_low_pri_work(
    struct nvmeibc_admin_channel *ch,
    struct workqe_struct *work);
```

**Priority Handling**:
- High priority work is always scheduled
- Low priority work is deferred if high priority work is pending
- Counter: `ch->wq_high_pri_cnt`

**Work Types**:
- Resource allocation/deallocation
- Channel setup/teardown
- TOMA message processing
- Dirty bit fetching
- Lock management

---

## Error Handling

### Connection Errors

#### QP Errors

```c
static void handle_qp_err(
    u64 wr_id,
    enum ib_wc_status wc_status,
    bool send_err,
    struct nvmeibc_ib_admin_channel *ch);
```

**Handled Errors**:
- `IB_WC_WR_FLUSH_ERR` - Work request flushed
- `IB_WC_RETRY_EXC_ERR` - Retry limit exceeded
- `IB_WC_RNR_RETRY_EXC_ERR` - RNR retry exceeded
- `IB_WC_RESP_TIMEOUT_ERR` - Response timeout
- `IB_WC_LOC_QP_OP_ERR` - Local QP operation error

**Action**: Disconnect admin channel

#### Login Rejection

Callback: `on_reject_admin_ch()`

**Common Reasons**:
- Version mismatch
- Resource exhaustion on target
- Authentication failure
- Disk not available

**Action**: Report error and clean up resources

#### Disconnect

Callback: `on_disconnect_admin_ch()`

```
1. Stop keep-alive

2. Mark as dying
   └── atomic_set(&ch->net.base.dying, 1)

3. Complete pending operations
   └── complete_all(&ch->send_done)

4. Notify disk layer
   └── nvmeibc_disk_on_admin_channel_disconnect()

5. Clean up resources
   └── Free IUs, message areas, etc.
```

### Message Errors

#### Send Timeouts

```c
/* Default timeout */
#define NVMEIB_WAIT_FOR_ADMIN_SEND_COMP (30 * HZ)  /* 30 seconds */
```

**Handling**:
```
1. Log timeout error

2. Check if dying
   └── May have disconnected during send

3. Disconnect channel
   └── nvmeibc_ib_net_disconnect()

4. Return error to caller
   └── -ETIMEDOUT
```

#### Receive Errors

- NULL IU: Log error, don't repost
- Sync errors: Log warning, repost IU
- Unhandled opcode: Log trace, repost IU

### Recovery

Admin channel failures typically trigger:

1. **Disk Rediscovery**
   - Retry connection to same target
   - Try alternate paths
   - Eventually mark disk as unavailable

2. **IO Failover** (for redundant configurations)
   - Redirect IO to replica
   - Continue operations

3. **Resource Cleanup**
   - Free allocated resources
   - Drain work queues
   - Release IUs and memory regions

---

## Debug and Diagnostics

### Tracing

The admin channel uses extensive tracing via the `_ND`, `_NT`, `_NE` macros:

```c
_ND(trace_name, "format", args...);  /* Debug */
_NT(trace_name, "format", args...);  /* Trace */
_NE(error_name, "format", args...);  /* Error */
```

**Key Trace Points**:
- Connection establishment/teardown
- Message send/receive
- Resource allocation/deallocation
- Keep-alive events
- TOMA message handling

### Debug Commands

The admin channel supports debug commands via `NVMEIB_DBG_CMD`:

```c
enum nvmeibc_dbg_cmd_opcode {
    NVMEIBC_DBG_PLEASE_KILL_YOURSELF,  /* Request target to fail */
    /* ... others ... */
};
```

**Example**: `prp_dbg_please_kill_yourself()` - Used for testing failure scenarios.

### Statistics and Monitoring

Admin channel statistics (available via disk structures):

- **Message counters**: Sent/received messages by type
- **Error counters**: Timeouts, retries, disconnects
- **Latency**: Message round-trip times
- **Keep-alive**: Last successful time, timeout count

---

## API Summary

### Creation and Connection

```c
/* Create admin channel */
struct nvmeibc_ib_admin_channel *nvmeibc_ib_admin_channel_create(
    const struct nvmeibc_cinst_params_core *p,
    struct nvmeibc_admin_rnic *arnic);

/* Connect admin channel */
int nvmeibc_ib_admin_channel_connect(
    struct nvmeibc_ib_admin_channel *ch,
    struct nvmeibc_disk *disk,
    struct nvmeibc_admin_rnic *arnic);

/* Disconnect admin channel */
void nvmeibc_ib_admin_channel_disconnect(
    struct nvmeibc_ib_admin_channel *ch);

/* Free admin channel */
void nvmeibc_ib_admin_channel_free(
    struct nvmeibc_ib_admin_channel *ch);
```

### Configuration

```c
/* Access IO NICs */
int nvmeibc_ib_admin_channel_access_iornics(
    struct nvmeibc_ib_admin_channel *ch,
    struct nvmeib_rdma_ib_port_gid *port_gids,
    u16 port_pkey,
    struct nvmeibc_ib_port *port);

/* Request disk resources */
int nvmeibc_ib_admin_channel_request_disks_resources(
    struct nvmeibc_ib_admin_channel *ch);

/* Get journal range */
int nvmeibc_ib_admin_channel_get_journal_range(
    struct nvmeibc_ib_admin_channel *ch);
```

### IO Channel Management

```c
/* Connect IO channel */
int nvmeibc_ib_admin_channel_connect_io_channel(
    struct nvmeibc_ib_admin_channel *ch,
    struct nvmeibc_ib_io_channel *ioch);

/* Initialize IO channel */
int nvmeibc_ib_admin_channel_init_io_channel(
    struct nvmeibc_ib_admin_channel *ch,
    struct nvmeibc_ib_io_channel *ioch,
    u64 disk_rsc_id,
    u64 msix_table_addr,
    u64 msix_address,
    u32 msix_payload);

/* Connect No-RDDA channel */
int nvmeibc_ib_admin_channel_connect_nordda_channel(
    struct nvmeibc_ib_admin_channel *ch,
    struct nvmeibc_ib_nordda_channel *nrch);
```

### Lock Management

```c
/* Connect lock channel */
struct nvmeibc_locks_channel *nvmeibc_ib_admin_channel_connect_lock_lb_channel(
    struct nvmeibc_ib_admin_channel *ch);

/* Remove lock channel */
void nvmeibc_ib_admin_channel_rm_locks_ch(
    struct nvmeibc_locks_channel *lch);
```

### TOMA

```c
/* Send TOMA command (synchronous) */
int nvmeibc_ib_admin_send_toma_cmd(
    struct nvmeibc_ib_admin_channel *ch,
    struct nvmeibc_disk_toma_cmd *toma_cmd);

/* Send TOMA command (asynchronous) */
int nvmeibc_ib_admin_send_toma_cmd_async(
    struct nvmeibc_ib_admin_channel *ch,
    struct nvmeibc_disk_toma_cmd *toma_cmd);

/* Send TOMA response */
int nvmeibc_ib_admin_send_rsp(
    struct nvmeibc_ib_admin_channel *ch,
    u64 hdr_tag,
    struct volume_client_rsp *rsp,
    int rsp_opcode,
    int wr_opcode,
    int rsp_len);
```

### Message Handling

```c
/* Get transmit IU */
struct nvmeib_iu *nvmeibc_ib_admin_channel_get_tx_iu(
    struct nvmeibc_ib_admin_channel *ch);

/* Return transmit IU */
void nvmeibc_ib_admin_channel_put_tx_iu(
    struct nvmeibc_ib_admin_channel *ch,
    struct nvmeib_iu *iu);

/* Prepare and send message */
int nvmeibc_ib_admin_channel_prepare_n_send_msg(
    struct nvmeibc_ib_admin_channel *ch,
    ssize_t (*f)(void *p, void *buf, const void *buf_end),
    void *p,
    int wr_opcode,
    struct nvmeib_iu *iu);
```

---

## Related Documentation

- **NORDDA_IO_LIFECYCLE.md** - No-RDDA IO channel lifecycle
- **RDMA_COMPLETION_MODES.md** - RDMA completion handling
- **RDMA_COMPLETION_MODES_DIAGRAMS.md** - Visual diagrams of completion flows

---

## Glossary

| Term | Definition |
|------|------------|
| **Admin Channel** | Control-plane channel for management operations |
| **ARNIC** | Admin Remote NIC - target's admin NIC |
| **IU** | IO Unit - message buffer |
| **LIONIC** | Local IO NIC - client-side IO NIC |
| **No-RDDA** | Non-RDMA Direct Data Access (send/recv-based IO) |
| **RDDA** | RDMA Direct Data Access (RDMA read/write-based IO) |
| **RIONIC** | Remote IO NIC - target-side IO NIC |
| **SRQ** | Shared Receive Queue |
| **TOMA** | Topology Manager (volume orchestration) |

---

**Document Version**: 1.0  
**Last Updated**: 2026-01-27  
**Author**: Auto-generated from codebase analysis

