---
layout: post
title:  "76 requests per second?"
date:   2020-03-08 22:27:30
categories: bugs
---

It was a particularly mischievous bug. At the time, we were writing a new
research prototype kernel in our group. Research operating systems are fun
because you typically end up writing a lot of code until you get something very
basic (like print) running. In fact, it's often the case that you end up with a
system or function call which can't be fully implemented at the moment (because
it would require too much code or other things are still missing for it).
Therefore, every so often, you have to cut some corners and do a temporary,
partial implementation to make progress on a goal. 

Unfortunately, if you cut too many corners, you end up forgetting about them.
Then they haunt you as bugs weeks or months later. This post is about such an
instance of a bug I had to deal with -- approximately a year ago.

### The premise

In our prototype OS, we had a kernel that implemented user-space processes,
memory management, files, and device pass-through. We also had a user-space
runtime for fine-grained scheduling (including locks etc.). However, at the time
neither our runtime nor our kernel had full support for POSIX. POSIX is pretty
important if you want to run existing applications, which in turn is important
if you want to evaluate how well your system is doing. Unfortunately,
implementing POSIX is a huge undertaking and our resources were limited.

Fortunately, this problem was solved by a project called [rumpkernels][0] -- a
NetBSD libOS. A libOS is a fancy term for a library that contains an entire
operating system (in this case NetBSD). I'm not going into [too many details][1]
here, but the way it works is that you'll end up implementing a minimal set of
[functions][2] (around 50, called hypercalls in rump) which abstract away the
underlying machine. In return, you'll get all the NetBSD code (kernel and
user-space) running inside your process: So instead of implementing a TCP/IP
stack, disk and network drivers, libpthread and libc, you just have to implement
these ~50, arguably pretty basic, functions which primarily deal with low-level
memory management, timers, scheduling, and locking. It's a great trade-off,
especially since we already had most of that.

### 76 Requests?

So there I was, working for a few weeks on writing the implementation for those
50 hypercall functions. It was also around Christmas, so I ended up taking a
break at some point. Overall, it probably took about 2-3 months to finish it.
Many of the hypercalls were trivial to implement since we already had support
for it, but some required significant additional implementation. After the
implementation was completed, I figured I would test it by compiling a
user-space process, linking the NetBSD code, my hypercall implementation, and
[redis][3] (a single-threaded key--value store). I definitely had some linking
and init issues first, but after some digging, I finally saw a familiar
prompt on my terminal:

```log
[INFO] - vibrio::rumprt::crt::tls: tls area size is: data + bss + extra = 48
[INFO] - vibrio::rumprt::prt: rumprun_makelwp 0x37a10008 0x3e019240 0x3e801000--4194304 64 0x37a10198
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 3.0.6 (ffbeedb9/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 1
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

1:M 02 Feb 02:02:10.540 # Server started, Redis version 3.0.6
1:M 02 Feb 02:02:10.540 * The server is now ready to accept connections on port 6379
```

Redis started, great! Eager to see if it worked, I ran the redis-benchmark
tool and sent GET requests to the network interface of the VM:

```log
$ redis-benchmark -h 172.31.0.10 -n 1000
====== GET ======
  1000 requests completed in 13.00 seconds
  50 parallel clients
  3 bytes payload
  keep alive: 1

30.10% <= 2 milliseconds
80.10% <= 3 milliseconds
85.10% <= 4 milliseconds
90.10% <= 5280 milliseconds
93.40% <= 5281 milliseconds
93.90% <= 5282 milliseconds
95.10% <= 6712 milliseconds
96.30% <= 7684 milliseconds
99.20% <= 7685 milliseconds
100.00% <= 7685 milliseconds
76.92 requests per second
```

76 requests?! That's pretty terrible for most applications, let alone a
key--value store... Just for comparison: a Linux VM with redis can do about 2
million requests per second. So we're off by a factor of **26'000x**, yelp. The
issue was also pretty clear from the histogram: 10% of the requests ended up
just being *really* slow (like 5+ seconds slow). What's going on?

### Debugging

With prototype OSes there is usually little support for debugging aside from
printing and guessing. And its even harder if you're debugging performance.
However, with 76 requests per second, there ought to be something seriously
wrong here so this should be easy? I remember being puzzled for a while though:

At first, I figured surely it must be slow because I am compiling all this code
in debug mode without optimizations. So I quickly tried it with compiler
optimizations turned on: No change at all, still 76 requests. This was
confusing, there should have been at least some impact from the compiler?

My next suspicion was that I must have messed up some of the locking
implementation. I instrumented most of them but to no avail. They didn't take up
any significant amount of time during the execution.

I also carefully inspected the scheduler and it's run queues. Maybe the right
threads just weren't running because of a bug in the scheduling algorithm? I
didn't find anything wrong with it tough, the system was happily running all
kinds of different threads at different times.

Interrupts were another potential culprit: Maybe they didn't arrive or they
didn't work properly. I dismissed the theory when I verified that the interrupt
handler did indeed get invoked properly by pinging the system.

Finally, I blamed rump itself (always good to blame others when desperate). But
running the original rumpkernel as a unikernel in a VM revealed it does indeed
work as intended with millions of GETs per second.

What got me on the right track eventually was the fact that pinging the
interface also showed a similar anomaly with latency (although with less total
variance).

```
$ ping 172.31.0.10
PING 172.31.0.10 (172.31.0.10) 56(84) bytes of data.
64 bytes from 172.31.0.10: icmp_seq=1 ttl=255 time=5.78 ms
64 bytes from 172.31.0.10: icmp_seq=2 ttl=255 time=5.32 ms
64 bytes from 172.31.0.10: icmp_seq=3 ttl=255 time=2.50 ms
64 bytes from 172.31.0.10: icmp_seq=4 ttl=255 time=0.940 ms
64 bytes from 172.31.0.10: icmp_seq=5 ttl=255 time=8.81 ms
```

Ping uses ICMP requests, which are handled quite differently from TCP or UDP
packets. I figured the issue must be somewhere deep down in the device driver,
or device itself.

At one point, I didn't quickly kill the system and let `ping` run for a longer
time. That's when I saw some concerning errors in the serial log (and at the
same time the ping messages suddenly stopped arriving):

```
wm0: device timeout (txfree 4032 txsfree 0 txnext 832)
wm0: device timeout (txfree 4032 txsfree 0 txnext 832)
wm0: device timeout (txfree 4032 txsfree 0 txnext 832)
```

`wm0` is our network device (a e1000 PCIe NIC emulated by QEMU here). The
device timeout message is due to a [watchdog timer expiring in the NetBSD driver
code][4]. At this stage, I was still confused. Why would the device not work
correctly for me? It definitely works with any other OS. The bug must be
somewhere in my implementation, but where?


### rumpcomp_pci_dmalloc

I started to piece things together when I finally noticed the following message
in the log: 
`[ERROR] - vibrio::rumprt::dev: rumpcomp_pci_dmalloc size=65536 alignment=4096`

rumpcomp_pci_dmalloc is one of the required hypercalls. It's used by the NetBSD
kernel (and its device drivers) to allocate DMA memory which can be read and
written to by device drivers *and* the devices. This is important because the
device eventually needs to write incoming packets somewhere into memory or read
outgoing packets from memory (using [DMA][5]). DMA memory is a bit special
because we need to know the underlying physical address of the allocated buffer.
This is because a device does not use virtual addresses (like our process), and
ends up referring to memory by physical addresses[^1].

Looking at our rumpcomp_pci_dmalloc function, nothing is obviously wrong with
it:

```rust
#[no_mangle]
pub unsafe extern "C" fn rumpcomp_pci_dmalloc(
    size: usize,
    alignment: usize,
    pptr: *mut c_ulong,
    vptr: *mut c_ulong,
) -> c_int {
    let layout = Layout::from_size_align_unchecked(size, alignment);
    if size > 4096 {
        error!("rumpcomp_pci_dmalloc size={} alignment={}", size, alignment);
    }

    let mut p = crate::mem::PAGER.lock();
    let r = (*p).allocate(layout);
    match r {
        Ok((vaddr, paddr)) => {
            *vptr = vaddr.as_u64();
            *pptr = paddr.as_u64();
            0
        }
        Err(_e) => 1,
    }
}
```

In short, the function takes a size and alignment argument to allocate a buffer
that is *physically and virtually contiguous*. It then expects us to write the
corresponding physical and virtual start address of the new buffer into `pptr`
and `vptr`.

Notice the innocent error message that gets printed if size is >4096? Remember
the shortcuts I talked about in the beginning? Well here is one that was taken
early on and then forgotten about:

The interesting logic happens in `(*p).allocate(layout)`. Eventually, this
function calls into our kernel, which has to allocate memory, map it into our
address space and return the virtual (`vaddr`) and physical (`paddr`) address to
it. When this function was first implemented, our kernel used a [buddy
allocator][6] to allocate physical memory. The buddy allocator would always
return a physically consecutive region of memory, which the kernel mapped  as a
virtually consecutive region by writing the page-table entries accordingly.

The bug got introduced when sometime in the middle of implementing the
hypercalls, the underlying kernel memory allocator changed: Rather than using a
buddy allocator (which is fairly slow), an optimization got added that held a
bunch of pages as 4 KiB frames on a stack to allocate from them quickly.
However, the problem now was that these frames were no longer necessarily
physically consecutive. For buffers larger than 4 KiB the kernel would happily
pop off frames from the stack, add them together in virtual space, and then just
return the physical address of the first page that got mapped as `paddr`.

This meant that if the size of the buffer was exactly 4 KiB, everything was
virtually and physically consecutive. For larger memory areas like the one
allocated by the NIC driver (64 KiB), the returned `paddr` would only match for
the first 4 KiB of the buffer. Afterwards all bets were off[^3]. The driver
would then take this large buffer and slice it up into smaller packet buffers.
The physical address of those individual buffers was calculated by taking the
original `paddr` and adding a offset to it. What this meant for the system was
that only a small percentage of the packet buffers were actually working. For
the rest, the device would write to them using some physical address but it
didn't match with the corresponding virtual address in the driver code.

What was surprising here is that even though only a fraction of the packet
buffers were working (e.g., were correctly visible in both device and driver),
the system still "worked", thanks to the all robustness (retransmission, error
checking etc.) built into TCP/IP. It just had really terrible performance.

### What went wrong?

The contract between the user-space allocator and the kernel allocator clearly
broke when the kernel allocator got optimized. So, when I initially changed the
allocation scheme in the kernel, I added the error message in
`rumpcomp_pci_dmalloc` to indicate that this should be fixed later for buffers
larger than 4 KiB (cutting a corner).

However, fixing it wasn't a top priority at the time: we wanted to bring up the
system without drivers anyways first. So instead of using an assert, logging an
error was a better way to make progress on other parts of the system.
Unfortunately, after a while the error message just got forgotten, only to be
rediscovered later in a pile of other log messages.

Once I fixed the bug, redis jumped from 76 to 24k GET requests per second. That
was a quick 325x improvement! It took getting rid of 2-3 other bugs to go past a
million, but those are a story for another time.

[^1]: For simplicity, we ignore the IOMMU here.
[^3]: Given how our stack based allocator worked it would, with high chance, often have some other frames in there that would align but there were no guarantees for that.

[0]: https://github.com/rumpkernel "Rump Kernels"
[1]: https://aaltodoc.aalto.fi/bitstream/handle/123456789/6318/isbn9789526049175.pdf "Rumpkernel PhD thesis"
[2]: https://man.netbsd.org/NetBSD-8.0/i386/rumpuser.3 "rump kernel hypercall interface"
[3]: https://redis.io/ "Redis key--value server"
[4]: https://github.com/NetBSD/src/blob/1a75eb3d30e516d310e1482ba34fd1e65930c97b/sys/dev/pci/if_wm.c#L3274
[5]: https://en.wikipedia.org/wiki/Direct_memory_access "Direct memory access"
[6]: https://en.wikipedia.org/wiki/Buddy_memory_allocation "Buddy allocator"