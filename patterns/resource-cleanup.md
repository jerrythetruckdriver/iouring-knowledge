# Resource Cleanup and Ring Teardown

## The Problem

io_uring resources (rings, registered files, registered buffers, provided buffer rings) must be cleaned up properly. Leaks are subtle: a forgotten registered buffer pin keeps physical pages locked, eating RLIMIT_MEMLOCK. A forgotten ring fd leaks kernel memory.

## Ring Teardown

```c
io_uring_queue_exit(&ring);
```

This calls `close()` on the ring fd and unmaps the shared memory. The kernel:

1. Cancels all in-flight requests (best-effort)
2. Waits for uncancelable requests to complete
3. Unregisters all resources (files, buffers, eventfd)
4. Frees the ring memory
5. Unpins registered buffer pages

### In-Flight Requests

Requests in-flight at teardown get canceled. But some operations (like disk writes already submitted to hardware) can't be canceled — the kernel waits for them. `io_uring_queue_exit()` may block briefly.

For clean shutdown:

```c
// 1. Cancel all pending requests
io_uring_prep_cancel(sqe, NULL, IORING_ASYNC_CANCEL_ANY | IORING_ASYNC_CANCEL_ALL);
io_uring_submit(&ring);

// 2. Drain CQEs until all cancel confirmations received
while (pending > 0) {
    io_uring_wait_cqe(&ring, &cqe);
    // Process cancel CQEs
    pending--;
    io_uring_cqe_seen(&ring, cqe);
}

// 3. Now safe to exit
io_uring_queue_exit(&ring);
```

## File Descriptor Lifecycle

### Normal FDs

Accepted or opened via io_uring — these are in the process fd table. Close them with `IORING_OP_CLOSE` or `close()`. If you forget, they survive ring teardown (they're process resources, not ring resources).

### Direct Descriptors (Fixed Files)

Allocated into the ring's fixed file table via `IORING_FILE_INDEX_ALLOC`. These are ring resources — they're freed on ring teardown. But relying on teardown for cleanup is sloppy. Close them explicitly:

```c
io_uring_prep_close_direct(sqe, fixed_slot);
```

### The Leak Pattern

```c
// Bug: accept into process fd table, then register as fixed file
int fd = accept(...);  // fd in process table
io_uring_register_files_update(&ring, slot, &fd, 1);  // also in fixed table
// Now fd exists in TWO places
// io_uring_queue_exit() cleans up the fixed table entry
// but the process fd table entry leaks
```

Fix: use direct accept (`IORING_FILE_INDEX_ALLOC`) to avoid double registration.

## Registered Buffer Cleanup

```c
// Unregister explicitly
io_uring_unregister_buffers(&ring);

// Or just exit the ring — kernel unpins automatically
io_uring_queue_exit(&ring);
```

If you cloned buffers to multiple rings, unregister from each ring or exit each ring. The physical pages are unpinned when the last reference is dropped.

## Provided Buffer Rings

```c
// Unregister
struct io_uring_buf_reg reg = { .bgid = my_group };
io_uring_unregister_buf_ring(&ring, &reg);

// If kernel-allocated (IOU_PBUF_RING_MMAP), munmap the region
munmap(pbuf_ring_ptr, size);
```

If application-allocated, free the memory after unregistering.

## Eventfd

```c
io_uring_unregister_eventfd(&ring);
close(eventfd);
```

Or just exit the ring — eventfd is unregistered automatically. But still close the eventfd itself.

## SQPOLL Thread

SQPOLL thread is stopped when the ring is closed. If the ring fd is leaked (not closed), the SQPOLL thread runs forever, consuming CPU.

## Personality

```c
io_uring_unregister_personality(&ring, personality_id);
```

Or freed on ring exit. Minor resource — just a credential reference.

## Signal Handling

Register a cleanup handler for SIGTERM/SIGINT:

```c
volatile sig_atomic_t shutdown_requested = 0;

void signal_handler(int sig) {
    shutdown_requested = 1;
}

// In event loop
if (shutdown_requested) {
    // Cancel all, drain, exit
    graceful_shutdown(&ring);
}
```

Or use `IORING_OP_WAITID` / pidfd polling for child process cleanup.

## Graceful Shutdown Pattern

```c
void graceful_shutdown(struct io_uring *ring) {
    // Phase 1: Stop accepting new connections
    // Cancel multishot accept
    io_uring_prep_cancel(sqe, accept_user_data, IORING_ASYNC_CANCEL_USERDATA);
    
    // Phase 2: Drain existing connections
    // Cancel all multishot recvs
    io_uring_prep_cancel(sqe, NULL, IORING_ASYNC_CANCEL_ANY | IORING_ASYNC_CANCEL_ALL);
    io_uring_submit(ring);
    
    // Phase 3: Wait for cancel confirmations
    int outstanding = count_pending_ops();
    while (outstanding > 0) {
        io_uring_wait_cqe(ring, &cqe);
        outstanding--;
        io_uring_cqe_seen(ring, cqe);
    }
    
    // Phase 4: Close accepted connections
    for (int i = 0; i < max_conns; i++) {
        if (conn_active[i]) {
            io_uring_prep_close_direct(sqe, i);
        }
    }
    io_uring_submit_and_wait(ring, active_count);
    
    // Phase 5: Unregister resources (optional, exit does this)
    io_uring_unregister_buffers(ring);
    io_uring_unregister_files(ring);
    
    // Phase 6: Exit ring
    io_uring_queue_exit(ring);
}
```

## Common Leaks

| Resource | Symptom | Detection |
|----------|---------|-----------|
| Ring fd | Kernel memory leak, /proc/*/fdinfo entries | lsof, /proc/self/fd |
| Process fd from accept | fd table grows, EMFILE | /proc/self/fd count |
| Registered buffers | VmLck grows in /proc/self/status | `grep VmLck /proc/self/status` |
| SQPOLL thread | CPU usage on idle | `ps -eLf \| grep io_uring` |
| Provided buffer memory | RSS grows | Application allocator stats |

## Rule of Thumb

1. Direct descriptors > normal fds (ring manages lifecycle)
2. Cancel before exit (don't rely on kernel cleanup for correctness)
3. Track outstanding operations (can't drain what you don't count)
4. Ring exit cleans up ring resources, not process resources
5. Test with valgrind/ASAN — buffer leaks in provided buffer rings are easy to miss
