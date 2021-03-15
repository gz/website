---
layout: post
title:  "Context-switching gone wrong"
date:   2021-03-13 12:11:11
categories: bugs
---

I remember while this bug wasn't that hard to fix (or find), it still had some
"fantastic" potential to it. Because, when it appeared, at first nothing made
sense anymore...

When programming, it happens occasionally that you end up with a dead-end: A
branch in the code that you determine will *never* be taken by the CPU with
absolute certainty. However, due to limitations in the language or the structure
of your algorithm you can't really avoid to account for it in the program.

An example can be a match statement like this one:

```rust
let mut ops = Vec::with_capacity(nop);
for _i in 0..nop {
    let op = orng.gen();
    match op % 2 {
        0 => ops.push(Operation::WriteOperation(OpWr::Pop)),
        1 => ops.push(Operation::WriteOperation(OpWr::Push(orng.gen()))),
        _ => unreachable!("Can only have 0 or 1"),
    }
}
```

Since we end up doing modulo two on our `op` variable, the only values we ever
expect to see is 0 or 1 in that match statement. However, the rust compiler (and
pretty much every other compiler I'm aware of) isn't smart enough to determine
this and still wants us to provide a wild-card pattern (using `_`), which
provides an execution path for every number that is not 0 or 1. Because we
determine this code-path can never be executed we end up putting an
`unreachable!` statement there, which would just abort the program. That's
perfectly fine because it's obvious to see that we never go there.

Another (slightly weaker) form of such paths are "trivial" pre-conditions in the
form of asserts. For example, here is such a pre-condition in C:

```c
void mutex_obj_hold(kmutex_t *lock)
{
    struct kmutexobj *mo = (struct kmutexobj *)lock;

    KASSERTMSG(mo->mo_magic == MUTEX_OBJ_MAGIC,
        "%s: lock %p: mo->mo_magic (%#x) != MUTEX_OBJ_MAGIC (%#x)",
         __func__, mo, mo->mo_magic, MUTEX_OBJ_MAGIC);

    atomic_inc_uint(&mo->mo_refcnt);
}
```

Essentially, the `KASSERTMSG` is saying that in order to call `mutex_obj_hold`
the argument provided must be of type `kmutexobj`, which is validated in C by
checking that the member `mo_magic` is set to the constant `MUTEX_OBJ_MAGIC`.
This is a way to add some type robustness for a "loosely typed" language such as
C. While this pre-condition can certainly be wrong, in practice this is rarely
expected to happen. The only time this supposedly can fail is if someone just
wrote some locking code and invoked this function with a wrong argument (aka a
different type) or forgot to initialize the lock[^1].

### Branching into the unreachable

As you can probably guess, the manifestation of the bug in this post was that
the execution of a program would suddenly end up in such code-paths by taking
these "impossible" branches. Which meant for the first example, that it could
suddenly fail with

```log
thread 'main' panicked at 'internal error: entered unreachable code: Can only be 0 or 1', src/main.rs:6:5
```

And, because that wasn't weird enough, the bug also wasn't deterministic: Most
of the time the program would work "just fine", but every so often the code
would abort due to some random `assert`, `panic`, or `unreachable!` statement.
Upon inspection they always turned out to be unlikely or impossible things which
now suddenly happened at an alarming rate.

To understand this a bit better, we have to refresh some knowledge about CPU
assembly language (we use x86) to see what's really going on under the hood.
It's important to understand that our `match` and `KASSERT` example will
eventually be rewritten using just a bunch of `if` statements by the compiler:

```rust
if op % 2 == 0 {
    ops.push(Operation::WriteOperation(OpWr::Pop));
}
else if op % 2 == 1 {
    ops.push(Operation::WriteOperation(OpWr::Push(orng.gen())));
}
else {
    unreachable!("Can only have 0 or 1");
}
```

Afterwards, the code is further simplified into comparisons (`cmp`) and branch
instructions (`jmp`, `jne` etc.) when converted to x86 assembly. To explain, we
show the assembly code for the first if statement, `if op % 2 == 0`:

```asm
and     rax, 1
cmp     rax, 0
jne     .LBB126_3
```

The first instruction does a bitwise `and` with `op` (in `rax`) and the constant
`1` (calculates `op % 2` by taking the first bit of `op` and stores this in
`rax`), then it compares the `rax` to 0 using `cmp`, and finally if `rax`
contained something not 0, it will jump to another location (e.g., to the label
`.LBB126_3`). However, if it was 0 it will continue with the next instruction
after `jne`.

One thing that's important to understand here is that there is an implicit
dependency between the result of `cmp` and `jne` that is not directly visible:
Since `jne` needs to jump (or not) based on the result of the previous `cmp`
instruction, it needs to access its result somehow. On x86, the CPU has a
special register called [RFLAGS][0] for this purpose. What happens is that the
[`cmp`][1] instruction will subtract the second operand (0) from the first
operand (`rax`) and then set flags in RFLAGS based on the result of the
subtraction. For example, the Zero Flag (which is the one important for `jne`),
is set to 1 if the subtraction performed by `cmp` results in zero. This means
`jne` just needs to inspect the Zero Flag in RFLAGS: if it is set it won't jump
because the compared values were equal, but if it remained unset it will jump to
the specified location.

### Locating the issue

So what went wrong with our code? Why would the CPU suddenly take those odd
branches that led nowhere? Assuming our branch logic in the CPU wasn't broken,
surely the bug must be in the software somewhere?

As always there were a couple of options where this bug could be located: For
example, since we wrote our own ELF parsing and loading library, if something
was messed up in there about how instructions get loaded or relocated it could
definitely become a problem. This seemed unlikely though since the bug only
affected branches and also wasn't deterministic.

What really helped was when I eventually figured out that this bug only happened
on CPU cores were the OS received interrupts during execution. That limited the
scope of the bug to anything that would happen in the interrupt handler. To
isolate further, I downsized the interrupt handler by removing all program logic
in it (aside from restoring the context). When afterwards the bug was still
present, I knew it must be in the context restore logic.

To explain this a bit more: The bread and butter of an operating system is to
give an illusion to a program that it is running without any interruption on a
CPU. In reality tough, an OS will context-switch many times per second to
execute many different programs concurrently. One cause for a context switch can
be interrupts: For example from a device, to notify when a packet is ready, or a
timer to tell us that it might be time to run some other program now. To hold up
the illusion, OS and hardware ideally must capture the exact state of the CPU
when the interruption happens and restore back that exact state to resume
execution of a program later.

There are two predominant ways to resume execution in user-space from a kernel
on modern x86: [sysret][3] to return after a program made a syscall, and
[iretq][2] to return after an interrupt. Both need to restore the CPU context
which can include some or all general-purpose registers, the SIMD and floating
point state, as well as the aforementioned RFLAGS register. Context-switching
usually can be executed only through a series of assembly statements.

Here is a code-snippet that stores the CPU state when an interrupt happens (it
misses a register, can you spot it?). A memory region, accessed through `%rax`,
is used to move the CPU register contents to a process-specific save area.

```asm
// Interrupt execution starts here...

// Save original %rax temporarily on the stack 
// because we will overwrite it to hold a reference to save area
pushq %rax

// First, get the pointer to the register save area
rdgsbase %rax
movq 0x8(%rax), %rax

// Save CPU general-purpose register context
// We don't save %rax yet since we used it to
// reference the save area location
movq %rbx,  1*8(%rax)
movq %rcx,  2*8(%rax)
movq %rdx,  3*8(%rax)
movq %rsi,  4*8(%rax)
movq %rdi,  5*8(%rax)
movq %rbp,  6*8(%rax)
// We don't save %rsp yet since it is overridden by CPU on irq entry
movq %r8,   8*8(%rax)
movq %r9,   9*8(%rax)
movq %r10, 10*8(%rax)
movq %r11, 11*8(%rax)
movq %r12, 12*8(%rax)
movq %r13, 13*8(%rax)
movq %r14, 14*8(%rax)
movq %r15, 15*8(%rax)

// Save original rax, which we pushed on the stack previously
popq %r15
movq %r15, 0*8(%rax)

// Save %rsp of interrupted process
movq 5*8(%rsp), %r15
movq %r15, 7*8(%rax)

// Save RIP were we were at before we got interrupted
// This is at rsp+16 (put there by the hardware):
movq 2*8(%rsp), %r15
movq %r15, 16*8(%rax)

// Saves the fs register
rdfsbase %r15
movq %r15, 19*8(%rax)

// Save vector registers
fxsave 24*8(%rax)

// Move on to high-level language function
callq handle_generic_exception
```

And here is the "opposite" code-snippet that restores the CPU context again
(from the save area, now in `%rdi`) and returns to the interrupted program using
`iretq` (this example doesn't miss anything and is correct):

```asm
// Restore fs and gs registers
swapgs
movq 19*8(%rdi), %rsi
wrfsbase %rsi

// Restore SIMD/vector registers
fxrstor 24*8(%rdi)

// Restore general-purpose CPU registers
movq  0*8(%rdi), %rax
movq  1*8(%rdi), %rbx
movq  2*8(%rdi), %rcx
movq  3*8(%rdi), %rdx
movq  4*8(%rdi), %rsi
// %rdi: Restored last (see below) to preserve `save_area`
movq  6*8(%rdi), %rbp
// %rsp: Restored through `iretq` (see below)
movq  8*8(%rdi), %r8
movq  9*8(%rdi), %r9
movq 10*8(%rdi), %r10
movq 11*8(%rdi), %r11
movq 12*8(%rdi), %r12
movq 13*8(%rdi), %r13
movq 14*8(%rdi), %r14
movq 15*8(%rdi), %r15

// Stack segment register
pushq 18*8(%rdi)
// %rsp stack register
pushq 7*8(%rdi)
// RFLAGS register
pushq 17*8(%rdi)
// Code segment register
pushq 19*8(%rdi)

// Instruction pointer
pushq 16*8(%rdi)

// Restore `rdi` register last, since it was used to reach `save_area`
movq 5*8(%rdi), %rdi

// And back we go:
iretq
```

The bug in this case was that we simply forgot to ever store `RFLAGS` to the
save area in the first snippet. This meant that every time we returned from an
interrupt it would use an outdated `RFLAGS` (the one which remained from the
last time the process did a system call). 

Going back to our branch example from earlier: Imagine an interrupt happens
after the CPU already executed `cmp` but just before it wanted to start
executing `jne`. Instead of resuming execution at `jne` with the up-to-date
`RFLAGS`, now we resume with an outdated `RFLAGS` value that may or may not
correspond to what `cmp` computed one instruction earlier. Without restoring
`RFLAGS` properly, our branching became a seemingly random affair -- depending
on whether interrupts would happen or not and when :).


[^1]: Another way would be through memory corruption, but that's not what was happening with this bug.

[0]: https://en.wikipedia.org/wiki/FLAGS_register "RFlags Register"
[1]: https://www.felixcloutier.com/x86/cmp "CMP Instruction on x86"
[2]: https://www.felixcloutier.com/x86/iret:iretd "IRETQ Instruction on x86"
[3]: https://www.felixcloutier.com/x86/sysret "SYSRET Instruction on x86"