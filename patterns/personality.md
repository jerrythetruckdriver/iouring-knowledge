# Personality / Credentials Support

Run io_uring operations with different credentials than the submitting thread.

## What It Is

A "personality" is a saved set of credentials (uid, gid, capabilities, security labels) registered with the ring. When submitting an SQE, you can set `sqe->personality` to execute that operation under different credentials.

## Registration

```c
// Register current thread's credentials as a personality
__u16 id = io_uring_register_personality(&ring);
// id > 0 on success, use in sqe->personality

// Unregister when done
io_uring_unregister_personality(&ring, id);
```

`IORING_REGISTER_PERSONALITY` (opcode 9) snapshots the calling thread's `struct cred` at registration time. The kernel holds a reference to this credential set.

## Usage

```c
// Submit operation with different personality
io_uring_prep_openat(sqe, AT_FDCWD, "/restricted/file", O_RDONLY, 0);
sqe->personality = saved_personality_id;
```

The open happens with the registered credentials, not the submitter's current credentials. Useful for:

- **Privilege separation**: Register a high-privilege personality, drop privileges, use the personality only for specific operations.
- **Multi-tenant**: Different personalities for different users sharing one ring.
- **Setuid-like behavior**: Specific operations need elevated access without keeping the whole process privileged.

## How It Works

1. `io_uring_register_personality` calls `get_current_cred()` and stores it
2. Returns a 16-bit ID (fits in `sqe->personality` field)
3. On submission, kernel overrides `current->cred` for that request
4. After completion, credentials are restored
5. Works with io-wq workers too — worker adopts the personality's credentials

## Limitations

- **16-bit ID space**: Max 65535 personalities per ring. In practice, you need far fewer.
- **Snapshot semantics**: Credentials are captured at registration time. Changes to the thread's credentials after registration don't affect the personality.
- **Security context**: LSM (SELinux/AppArmor) labels are included in the snapshot. This can be powerful or dangerous depending on your threat model.
- **No per-personality quotas**: All personalities share the ring's resource limits.

## Security Considerations

- Registering a personality requires having those credentials at registration time — you can't escalate
- But a ring with registered high-privilege personalities is a target. Protect the ring fd.
- Combine with `IORING_REGISTER_RESTRICTIONS` to limit which opcodes can use which personalities
- In container environments, personalities interact with user namespaces — test carefully

## Pattern: Privilege Drop with Escape Hatch

```c
// While still root:
__u16 root_personality = io_uring_register_personality(&ring);

// Drop to unprivileged user
setuid(nobody_uid);

// Can still do privileged ops through the personality
io_uring_prep_openat(sqe, AT_FDCWD, "/etc/shadow", O_RDONLY, 0);
sqe->personality = root_personality;
```

This is powerful but dangerous. Lock it down with restrictions.

## When to Use

Honestly? Rarely. Most applications don't need credential switching at the io_uring level. If you're building:

- A file server serving multiple users
- A database with per-tenant isolation
- A privileged daemon that needs to act as different users

Then personalities make sense. Otherwise, keep it simple.
