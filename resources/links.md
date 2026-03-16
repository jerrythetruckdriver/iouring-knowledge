# Resources

## Primary Sources

- **Jens Axboe's io_uring paper**: https://kernel.dk/io_uring.pdf — the original design document
- **liburing**: https://github.com/axboe/liburing — reference C library
- **io_uring header**: `include/uapi/linux/io_uring.h` in the kernel tree
- **Jens Axboe's kernel tree**: https://git.kernel.dk/cgit/linux/ — bleeding edge io_uring patches

## Documentation

- **Lord of the io_uring**: https://unixism.net/loti/ — best tutorial-level documentation
- **io_uring man pages**: https://man7.org/linux/man-pages/man7/io_uring.7.html
- **Kernel docs**: https://docs.kernel.org/filesystems/io_uring.html

## Articles & Blog Posts

- **Efficient IO with io_uring** (Jens Axboe, 2019) — https://kernel.dk/io_uring.pdf
- **An Introduction to io_uring** (LWN, 2020) — https://lwn.net/Articles/776703/
- **io_uring and networking in 2023** (LWN) — multishot, zero-copy send
- **The State of io_uring** (Jens Axboe, various Linux Plumbers) — annual state-of-the-art talks
- **io_uring is not an event system** (without.boats) — Rust perspective on completion I/O
- **Glommio design notes** (Glauber Costa) — thread-per-core + io_uring

## Conference Talks

- **Linux Plumbers Conference** — Jens Axboe presents io_uring updates annually
- **Kernel Recipes** — io_uring deep dives
- **FOSDEM** — io_uring in production talks
- **P99 CONF** — performance-focused io_uring content (TigerBeetle, etc.)

## Framework Source Code

Worth reading for io_uring usage patterns:

| Project | Language | Source |
|---------|----------|--------|
| TigerBeetle | Zig | https://github.com/tigerbeetle/tigerbeetle |
| Glommio | Rust | https://github.com/DataDog/glommio |
| Monoio | Rust | https://github.com/bytedance/monoio |
| tokio-uring | Rust | https://github.com/tokio-rs/tokio-uring |
| io-uring crate | Rust | https://github.com/tokio-rs/io-uring |
| Seastar | C++ | https://github.com/scylladb/seastar |
| liburing | C | https://github.com/axboe/liburing |

## Benchmarking Tools

- **fio** — the standard storage benchmark, io_uring ioengine built-in
- **t/io_uring** — Jens Axboe's standalone io_uring benchmark in liburing
- **io_uring-bench** — various community benchmarks

## Mailing Lists & Discussion

- **io-uring mailing list**: io-uring@vger.kernel.org
- **LWN.net**: Regular io_uring coverage
- **Kernel Git log**: `git log --oneline io_uring/` — the real changelog
