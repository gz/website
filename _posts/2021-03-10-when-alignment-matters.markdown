---
layout: post
title:  "When alignment matters"
date:   2021-03-11 02:22:30
categories: bugs
---

Some bugs are hard to find because when they manifest, they do so in a very
misleading way. This one is about fairly unexpected consequences when things
aren't properly aligned. Most of the time alignment problems lead to "just" bad
performance. In some other cases, hardware architectures may generate a fault on
misaligned accesses (but this is easy to pin-point and fix). However, in our
case the problem was a little more subtle.

We were trying to run some more applications on our new research OS. If you
haven't read the [previous post][0]: We had a new kernel that we built from
scratch and our user-space processes ran with a small libOS that we wrote from
scratch too. Basically, we had a lot of code which was fairly new, and bugs had
to be expected anywhere.

For example, our user-space runtime -- the victim of the bug in this post --
contained a custom built memory allocator, user-space scheduler and a system
call abstraction for the kernel. In addition, it provided the lowest layer (the
hypercalls) that allowed a process to also link with [rumpkernel][1]. rumpkernel
is a library OS built on NetBSD that gave a lot of POSIX support to a process.
Most importantly for this post, it came with an implementation of libpthread.
The way it worked was that libpthread provided an implementation of all the
pthread APIs, but underneath it would call into our own custom scheduler
(through the hypercalls), to create new threads and control which threads would
be put in and out of the run queues etc.

### Locking gone wrong

[Last time][0] we were in the process of porting redis to our OS. This time I
was trying to port a second application, namely [memcached][3]. It was a good
test because in comparison to redis, memcached was multi-threaded. So it
exercised a lot more code-paths than redis.

When trying to launch memcached, it became apparent that the process very often
ended up stuck somewhere during initialization and didn't make progress anymore.
Unfortunately, it wasn't really deterministic: Every time it would get stuck in
a slightly different function (and loop). And sometimes it even worked:
memcached started and waited for incoming connections -- great!

In such a scenario, a systems programmer will immediately think of two potential
classes of bugs: (1) a memory corruption: for example a stack corruption or a
buffer overflow could cause the program to wait in some loop forever. (2) a
locking issue: e.g., a deadlock or a race due to improper locking.

After getting stuck a few times, I started to notice that most of the times when
execution came to a halt, it was in one of the pthread locking functions.
Therefore, I expected a problem with locking. However, I trusted that memcached
would use pthread locks correctly and that libpthread (which came from NetBSD)
was correct too. What I certainly didn't have much faith in was my scheduler or
the integration of libpthread with it. Memory corruption wasn't high on my list
because most of our code was written in Rust and so we experienced very few
memory bugs, if at all.

For libpthread to work with our scheduler, I had to provide a [series][3]
[of][4] [low-level][5] [APIs][6]. These functions were prefixed with `_lwp`,
which stands for light-weight process. The `_lwp` APIs can be a fairly complex
affair to get right: For example [_lwp_park][7] and [_lwp_unpark][8] are used to
add threads and remove threads from the scheduler run-queue. In addition to
`_lwp_park` having many different parameters, one must also be aware that in
multi-threaded code, unpark can potentially arrive before park has even
completed... It isn't surprising that I spent a lot of time trying to find the
bug somewhere in my implementation of those APIs. A problem there could easily
lead to threads not being woken up or put to sleep incorrectly  (which in turn
meant someone else might end up waiting on a lock forever). However, as much as
I wanted to find the bug there, it just wasn't.

### Tagged pointers

Through the debug process, I became more familiar with the locking APIs in
NetBSD. I especially learned more about how pthreads worked internally than I
really wanted to know. But, this proved to be crucial for figuring out what was
wrong. What helped getting me on the right path was the following comment in
[pthread_int.h][9]:

```comment
Bits in the owner field of the lock that indicate lock state.  If the
WRITE_LOCKED bit is clear, then the owner field is actually a count of
the number of readers.
```

What this comment meant to convey is that in case of a reader-writer lock the
"owner field" (a 64 bit value) uses a bunch of bits (the first four) to indicate
things about the lock's state (`wait`, `wrlock` and `wrwant`). If `wrlock` is
set to 1, bits 4 to 63 point to an address where the owner struct resides, if
`wrlock` is set to 0 the bits contain the amount of readers instead.

```comment
 N                              4        3        2        1        0
 +------------------------------------------------------------------+
 | owner or read count          | <free> | wrlock | wrwant |  wait  |
 +------------------------------------------------------------------+
```

This is a form of tagged pointers, which encodes a few bits of meta-data along
with an address or count. The reason this works is because everything allocated
by malloc, such as the owner struct, is expected to have (at least) a 16 byte
alignment on our platform (so the first 4 bits of any pointer handed out by
`malloc` are guaranteed to be zero). If one wants to dereference the owner
pointer, libpthread just has to make sure to always mask out (set to zero) the
lower 4 bits before dereferencing it.

## malloc() and free()

As you can probably guess by now, the real culprit ended up being our malloc
aligning everything to 8 instead of 16 bytes. This meant that our address to the
owner would use the 4th bit, which was reserved for use by libpthread. Even
though the bit wasn't really in use, the pthread code would still clear the bit
before a dereference, and thus change the base address for the owner struct
slightly -- which turned out to be quite confusing for the pthread code.

One last detail about this bug was that our underlying memory allocator was
actually doing all the alignment correctly. However, our user-space runtime
including the memory allocator was written in Rust. The bug was introduced when
I wrote the wrapper code that converted the `malloc` and `free` calls (as
required by rumpkernel) to Rust's [alloc][10] and [dealloc][11] interface.

The [dealloc][11] function -- Rust's equivalent of `free(ptr)` -- takes a
pointer *and* a Layout (which encodes size and alignment information for the
pointer). With pure Rust code, this is fine since the Rust compiler will insert
the correct layout argument on deallocations. Hence, Rust allocators technically
don't need to manually keep track of the size of pointers handed out by `alloc`
(nice!). However to support `malloc` and `free` for C code, we need to at be
able to find the size of a pointer when it is given back to `free` (which gets
just the pointer as a single argument with no extra size or alignment). Memory
allocators written for C often solve this problem by allocating a few bytes more
than needed and storing the size of the block in the first few bytes of the
newly allocated memory.

Here is our original (buggy) code, which prepends every block with an 8 byte
header on `malloc` to store the size, and then reads the size again during free.
As you can see `HEADER_SIZE` is set to 8 bytes, which means our pointers we hand
out are now aligned to 8 bytes (even though we requested a 16 byte alignment
from the underlying allocator). The fix is to change `HEADER_SIZE` and/or
`ALIGNMENT` accordingly.

```rust
pub const HEADER_SIZE: usize = 8;
pub const ALIGNMENT: usize = 16;

/// Implementes malloc using the `alloc::alloc` interface.
///
/// We need to add a header to store the size for the
/// `free` and `realloc` implementation.
#[no_mangle]
pub unsafe extern "C" fn malloc(size: c_size_t) -> *mut u8 {
    trace!("malloc {}", size);

    let allocation_size: u64 = (size + HEADER_SIZE) as u64;
    let ptr = alloc::alloc::alloc(Layout::from_size_align_unchecked(
        allocation_size as usize,
        ALIGNMENT,
    ));

    if ptr != ptr::null_mut() {
        *(ptr as *mut u64) = allocation_size;
        ptr.offset(HEADER_SIZE as isize)
    } else {
        ptr::null_mut()
    }
}

/// Implements `free` through the rust `dealloc` interface.
///
/// Recovers the size of the block (needed by dealloc) by reading the
/// pre-pended header first.
#[no_mangle]
pub unsafe extern "C" fn free(ptr: *mut u8) {
    if ptr == ptr::null_mut() {
        return;
    }

    let allocation_size: u64 = *(ptr.offset(-(HEADER_SIZE as isize)) as *mut u64);
    alloc::alloc::dealloc(
        ptr.offset(-(HEADER_SIZE as isize)),
        Layout::from_size_align_unchecked(allocation_size as usize, ALIGNMENT),
    );
}
```

## Afterthoughts

I can think of two things that could've been done better here to prevent this
bug:

- A post conditions that asserts proper alignment for `ptr` (unfortunately I
  doubt I was consciously aware of the alignment restriction when I wrote that
  `malloc` function anyways)

- libpthread would be more robust if it used the high bits in the pointer for
  tagging rather than the first four (this would prevent alignment issues, but
  might cause other bugs more easily)

[0]: https://gerdzellweger.com/bugs/2020/03/08/76-requests-per-second.html "76 requests per second"
[1]: https://aaltodoc.aalto.fi/bitstream/handle/123456789/6318/isbn9789526049175.pdf "Rumpkernel PhD thesis"
[2]: https://memcached.org/ "memcached key--value server"
[3]: https://man.netbsd.org/_lwp_create.2 "_lwp_create man page"
[4]: https://man.netbsd.org/_lwp_unpark_all.2 "_lwp_unpark_all man page"
[5]: https://man.netbsd.org/_lwp_self.2 "_lwp_self man page"
[6]: https://man.netbsd.org/_lwp_ctl.2 "_lwp_ctl man page"
[7]: https://man.netbsd.org/_lwp_park.2 "_lwp_park man page"
[8]: https://man.netbsd.org/_lwp_unpark.2 "_lwp_unpark man page"
[9]: https://github.com/NetBSD/src/blob/ba908cc6df5309a877172334358b7f3103294e18/lib/libpthread/pthread_int.h#L314 "pthread_int.h"
[10]: https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html#tymethod.alloc "GlobalAlloc::alloc()"
[11]: https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html#tymethod.dealloc "GlobalAlloc::dealloc()"