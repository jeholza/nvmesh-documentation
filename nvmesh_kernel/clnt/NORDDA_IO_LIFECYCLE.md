# Nordda Channel IO Request Lifecycle Documentation

## Overview

This document describes the complete lifecycle of an IO request in the nordda (No-RDMA) channel implementation. The nordda channel is used for executing IO operations against remote storage targets using RDMA Send/Recv operations instead of RDMA Read/Write.

## Key Data Structures

### nvmeibc_volume_req_info

The IO request structure (`struct nvmeibc_volume_req_info`) represents a single IO operation. Key fields include:

- `req`: Embedded `nvmeibc_volume_request` containing the actual request data
- `nrch`: Pointer to the owning nordda channel
- `idx`: Request index in the channel's request array
- `version`: Request version for disambiguation
- `release_counter`: Tracks outstanding completions (send + recv)
- `raddr`, `rkey`, `rn_pages`: Remote bounce buffer information
- `wdc`: Watchdog context for timeout detection
- `send_counter`, `send_comp_counter`: Track send operations
- `n_send_comp`, `n_recv_comp`: Completion counters for debugging

## IO Request Lifecycle Phases

### Phase 1: Request Allocation

**Location:** `nvmeibc_ib_nordda_channel_get_io_context()` → `get_req_info()`

**File:** `clnt/nvmeibc_ib_nordda_channel.c`

1. Upper layer calls `nvmeibc_ib_nordda_channel_get_io_context()` to obtain a request context
2. `get_req_info()` acquires channel lock and allocates a request from the free pool
3. Request is removed from `ch->free_reqs` list
4. Channel's `n_used_reqs` counter is incremented
5. Request info reference count is incremented via `nvmeibc_channel_try_use_req_info()`

**State Changes:**
- Request removed from free pool
- Request marked as in-use
- `release_counter` initialized to 0

### Phase 2: IO Submission

**Location:** `nordda_execute_io()` → `nvmeibc_ib_net_nordda_execute_io()` → `execute_io()` or `execute_gen()` or `execute_lock()`

**Files:** 
- `clnt/nvmeibc_ib_nordda_channel.c` (nordda_execute_io)
- `clnt/nvmeibc_ib_net_nordda.c` (execute_io, execute_gen, execute_lock)

**For IO Commands (Read/Write/Discard):**

1. **Link Request to Command:**
   - `req->bcmd = block_cmd` (links request to disk IO command)
   - `req->md.has` set based on metadata presence

2. **Data Mapping:**
   - For reads and writes: `nvmeibc_ib_net_map_data()` maps scatter-gather list to RDMA-capable memory
   - For metadata: `nvmeibc_ib_net_map_md()` maps metadata buffers
   - DMA direction set: `DMA_FROM_DEVICE` for reads, `DMA_TO_DEVICE` for writes

3. **Request Preparation:**
   - `release_counter = 2` (one for send completion, one for recv completion)
   - Request version incremented via `NVMEIB_INC_TAG_VERSION(info->version)`
   - Tag encoded with channel version, request version, and request index

4. **Work Request Building:**
   - **For Writes:** RDMA Write operations created to write data to remote bounce buffer
   - **For Reads:** Only Send operation (remote will RDMA Write data to us)
   - Multiple work requests chained together if needed (data + metadata)
   - Journal metadata piggyback operations added if applicable

5. **Command Submission:**
   - Watchdog started via `nvmeibc_ib_nordda_channel_req_start_wd(info)`
   - Request marked in-use bitmap: `set_bit(info->idx, info->nrch->req_in_use)`
   - Work requests posted to send queue via `nvmeibc_ib_post_send()`
   - `send_counter` incremented
   - Latency measurement initiated

**State Changes:**
- `release_counter = 2`
- Request linked to disk command (`req->bcmd != NULL`)
- Data mapped and pinned
- Request marked in channel's in-use bitmap
- Watchdog armed
- Request posted to RDMA send queue

### Phase 3: Send Completion

**Location:** `nordda_send_completion()` → `send_completion_has_rsp()`

**File:** `clnt/nvmeibc_ib_nordda_channel.c`

**Triggered when:** RDMA send work request completes (local operation complete)

**Processing:**

1. **Validation:**
   - Decode request index and version from work completion ID
   - Verify tag in sent message matches work completion
   - Update last send success timestamp

2. **Latency Measurement:**
   - `nvmeibc_nr_lat_meas_send_comp()` records send completion time

3. **Completion Accounting:**
   - Increment `send_comp_counter`
   - Decrement `release_counter`

4. **Completion Handling:**
   ```c
   if (--req->release_counter == 0) {
       // Both send and recv completed
       nvmeibc_ib_nordda_channel_req_stop_wd(req);  // Stop watchdog
       nvmeibc_ib_net_free_req(&ch->net.base, &req->req);  // Unmap data
       clear_bit(req->idx, req->nrch->req_in_use);  // Clear in-use bit
       process_rsp_finalize(ch, req);  // Complete and return to pool
   }
   ```

5. **Special Case - Reused Bounce Buffers:**
   - If `req->reused_bb_wait_send_comp` is set, the request was reused before send completed
   - Send completion triggers pending IO processing to reuse the request

**State Changes:**
- `send_comp_counter` incremented
- `release_counter` decremented
- If `release_counter == 0`: Request unmapped and returned to pool

### Phase 4: Receive Completion (Standard Mode)

**Location:** `nordda_recv_completion()` → `nordda_handle_recv()` → `process_rsp()`

**File:** `clnt/nvmeibc_ib_nordda_channel.c`

**Triggered when:** Response message received from remote target

**Standard Mode** (`wait_release_zero_before_cb = false`):

1. **Response Processing:**
   - Decode response tag to identify request
   - Verify request index and version match
   - Update last received timestamp
   - Mark `recv_comp_arrived = true`

2. **Type-Specific Processing:**

   **For IO Commands** (`process_io_rsp`):
   - Extract completion code
   - Process piggyback lock reads if present
   - Record latency measurement
   - **Callback to datapath:** `nvmeibc_ib_net_complete_iocmd_reuse()` - **This completes to upper layer**
   - Check for bounce buffer reuse scenario

   **For Gen Commands** (`process_gen_rsp`):
   - Decode generic command response
   - Extract response data
   - Complete to upper layer via `nvmeibc_ib_net_nordda_complete_gen_cmd()`

   **For Lock Commands** (`process_lock_rsp`):
   - Decode lock operation result
   - Complete to upper layer via `nvmeibc_ib_net_nordda_complete_lock_cmd()`

3. **Release Counter Handling:**
   ```c
   if (--req->release_counter == 0) {
       // Both recv and send completed
       nvmeibc_ib_nordda_channel_req_stop_wd(req);  // Stop watchdog
       nvmeibc_ib_net_free_req(&ch->net.base, &req->req);  // Unmap data
       clear_bit(req->idx, req->nrch->req_in_use);  // Clear in-use bit
       process_rsp_finalize(ch, req);  // Return to pool and process pending
   }
   ```

**Key Point:** In standard mode, the datapath callback (completion to upper layer) occurs in the receive completion handler, regardless of whether send completion has arrived.

**State Changes:**
- `comp_code` set from response
- `recv_comp_arrived = true`
- `n_recv_comp` incremented
- **Datapath callback invoked** (upper layer notified)
- `release_counter` decremented
- If `release_counter == 0`: Watchdog stopped, data unmapped, request returned to pool

### Phase 5: Finalization and Pool Return

**Location:** `process_rsp_finalize()`

**File:** `clnt/nvmeibc_ib_nordda_channel.c`

**Called when:** `release_counter` reaches 0 (both send and recv completed)

**Processing:**

1. **Wait Mode Handling:**
   - If `wait_release_zero_before_cb = true`, datapath callback is invoked here
   - Otherwise, callback was already invoked in recv completion

2. **Bounce Buffer Reuse Check:**
   - If reuse conditions met, request is saved for later data write
   - Otherwise, proceeds to return request to pool

3. **Pool Return:**
   ```c
   if (nvmeibc_channel_try_use_req_info(&ch->base)) {
       nordda_pending_io(ch->base.disk, &ch->base, req, false, 0);
   }
   ```
   - `nordda_pending_io()` attempts to get pending disk command
   - If pending command available, request is immediately reused
   - Otherwise, `put_req_info()` is called to return to free pool

**State Changes:**
- Request unlinked from disk command (`req->dcmd = NULL`)
- Request either reused for pending IO or returned to free pool

### Phase 6: Request Pool Return

**Location:** `put_req_info()`

**File:** `clnt/nvmeibc_ib_nordda_channel.c`

**Processing:**

1. **Cleanup:**
   - `release_counter = 0`
   - Decrement `ch->n_used_reqs`

2. **Return to Pool:**
   - Add request back to `ch->free_reqs` list
   - Request now available for new IO operations

3. **Reference Counting:**
   - Call `nvmeibc_channel_end_use_req_info()` to decrement channel reference count
   - This must be last operation as channel may be freed after

**State Changes:**
- Request added to free pool
- Request available for allocation
- Channel reference count decremented

## Completion Ordering

### Standard Mode (`wait_release_zero_before_cb = false`)

This is the typical mode for performance:

1. **Recv Completion First:**
   - Datapath callback invoked immediately
   - Upper layer notified of IO completion
   - `release_counter` decremented to 1
   - Request still in use (waiting for send completion)
   - When send completion arrives: `release_counter` → 0, request returned to pool

2. **Send Completion First:**
   - No callback to upper layer yet
   - `release_counter` decremented to 1
   - Request remains in use
   - When recv completion arrives: Datapath callback invoked, `release_counter` → 0, request returned to pool

**Key Point:** Upper layer sees completion as soon as response arrives, but request isn't returned to pool until both completions arrive.

### Wait Mode (`wait_release_zero_before_cb = true`)

Used for IOMMU or specific HW (e.g., SIW):

- Datapath callback is deferred until both send and recv completions arrive
- Callback happens in `process_rsp_finalize()` when `release_counter` reaches 0
- More conservative but can avoid issues with reusing memory before local operations complete

## Watchdog Protection

**Purpose:** Detect stuck requests due to network issues or remote target failures

**Mechanism:**

1. **Start:** Watchdog armed in `nvmeibc_post_io()` via `nvmeibc_ib_nordda_channel_req_start_wd()`
2. **Timeout Detection:** `handle_watchdog_event_nordda()` checks for various timeout conditions:
   - QP timeout since send
   - QP timeout since last receive
   - Channel timeout (disk-specific)
3. **Action:** On timeout, `nordda_handle_error()` is called and channel disconnection initiated
4. **Stop:** Watchdog disarmed when both completions arrive (or in recv completion in standard mode)

## Error Handling

**Send Completion Error:**
- `nordda_handle_qp_err()` called with send error indication
- Transport errors trigger path disconnection
- Request error handling via `nordda_handle_error()`

**Recv Completion Error:**
- Similar error handling path
- Request tracked but completion deferred to disconnect handling

**Timeout:**
- Watchdog triggers error handling
- Channel disconnect initiated
- All pending requests completed with error in `nordda_channel_free_volume_reqs()`

## Bounce Buffer Reuse Optimization

For journal writes with EC volumes:

1. **First Request (Journal Write):**
   - Data written to bounce buffer
   - Recv completion arrives, data write acknowledged
   - Send completion may not have arrived yet

2. **Reuse for Data Write:**
   - Upper layer returns reused bounce buffer cookie
   - Same request reused with different command
   - Original request's FR descriptors stored aside
   - New version prevents confusion with late send completion

3. **Send Completion Handling:**
   - If send comp arrives after reuse: `reused_bb_wait_send_comp` mechanism tracks
   - Send comp releases original mapping
   - Request then available for pending IO

## Pending IO Processing

**Trigger Points:**
- After request finalization (`process_rsp_finalize`)
- After send completion if release counter reaches 0
- From upper layer via `nordda_pending_io`

**Processing:**
1. Check for pending disk commands in disk's pending queue
2. If available, reuse just-freed request immediately
3. If no pending commands, request returned to free pool

## Summary: Standard Mode Lifecycle

```
[Allocation]
    ↓
[Submit IO] ← request linked to cmd, release_counter=2, watchdog armed
    ↓
    ├─→ [Send Completion] ← release_counter--
    │        ↓
    │   (release_counter == 1)
    │        ↓
    │   [Wait for Recv]
    │
    └─→ [Recv Completion] ← datapath callback, release_counter--
             ↓
        (release_counter == 0)
             ↓
        [Stop Watchdog]
             ↓
        [Unmap Data]
             ↓
        [Process Finalize]
             ↓
        [Try Pending IO] → Found? → [Reuse Request]
             ↓
            No
             ↓
        [Return to Pool]
```

## Key Files

- **clnt/nvmeibc_ib_nordda_channel.c** - Channel management, completion handlers, lifecycle
- **clnt/nvmeibc_ib_net_nordda.c** - IO submission, request encoding, response decoding
- **clnt/nvmeibc_ib_nordda_channel.h** - Data structure definitions
- **clnt/nvmeibc_disk.c** - Pending IO management, upper layer integration

## Completion Mode Summary

| Aspect | Standard Mode | Wait Mode |
|--------|--------------|-----------|
| `wait_release_zero_before_cb` | `false` | `true` |
| Callback Timing | At recv completion | After both completions |
| Upper Layer Notification | Immediate | Deferred |
| Use Case | Performance (default) | IOMMU, specific HW |
| Request Pool Return | After both completions | After both completions |

Both modes return the request to the pool only after both send and recv completions arrive, ensuring all RDMA operations are complete before reuse.

