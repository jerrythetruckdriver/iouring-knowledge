# Read/Write Attributes (PI Metadata)

*Kernel 6.14+*

## Overview

io_uring can now exchange integrity/protection information (PI) metadata with read and write operations. This enables end-to-end data integrity verification through the storage stack without extra syscalls.

## The Interface

SQE fields:
```c
sqe->attr_ptr       // pointer to attribute struct
sqe->attr_type_mask // bitmask of which attributes to use
```

Attribute type flag:
```c
#define IORING_RW_ATTR_FLAG_PI  (1U << 0)
```

PI attribute struct:
```c
struct io_uring_attr_pi {
    __u16 flags;
    __u16 app_tag;
    __u32 len;
    __u64 addr;      // pointer to PI buffer
    __u64 seed;
    __u64 rsvd;
};
```

## Feature Detection

```c
#define IORING_FEAT_RW_ATTR  (1U << 16)
```

Check `io_uring_params.features` after setup.

## How It Works

1. Application prepares data buffer and PI metadata buffer
2. Sets `attr_type_mask = IORING_RW_ATTR_FLAG_PI` on the SQE
3. Points `attr_ptr` to `io_uring_attr_pi` struct
4. Kernel passes PI metadata alongside the I/O through the block layer
5. Hardware verifies/generates checksums based on PI configuration

## Use Cases

- **Database integrity**: end-to-end checksum verification for data pages
- **Storage appliances**: pass-through PI from client to NVMe device
- **Silent corruption detection**: hardware-verified checksums on every I/O

## Design

This is the io_uring counterpart to `preadv2()`/`pwritev2()` with PI extensions. The attribute mechanism is generic — `attr_type_mask` is a bitmask, so future attribute types can be added without changing the SQE layout.

NVMe devices with PI support (Type 1/2/3 protection) can use this to verify data integrity at the drive firmware level. No trust boundary — the checksum travels from application memory to disk platter and back.
