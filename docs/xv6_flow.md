# How xv6 works: flow diagrams

These diagrams follow the code in [xv6/xv6/](../xv6/xv6/), the xv6-riscv tree
described by `book/book-riscv-rev5.pdf`, not older xv6 releases. Two places
where this tree differs from what you may have read elsewhere:

- **No `initcode`.** `userinit()` only allocates a process. The first time any
  process runs `forkret()`, it calls `fsinit()` and then `kexec("/init")`
  directly ([proc.c:513-544](../xv6/xv6/kernel/proc.c#L513-L544)).
- **Sleep API.** Waiting is `sleep_prepare(chan)`, release your own lock, then
  `sleep()`. There is no `sleep(chan, lk)` ([proc.c:548-596](../xv6/xv6/kernel/proc.c#L548-L596)).

To view the diagrams, open the VS Code Markdown preview (the Markdown Preview
Mermaid Support extension renders them) or view the file on GitHub.

## Contents

1. [The whole system on one page](#1-the-whole-system-on-one-page)
2. [Build: from source to kernel and fs.img](#2-build-from-source-to-kernel-and-fsimg)
3. [Boot: power-on to the shell prompt](#3-boot-power-on-to-the-shell-prompt)
4. [Memory: address spaces, page tables, allocator, sbrk](#4-memory-address-spaces-page-tables-allocator-sbrk)
5. [Processes: states, scheduler, context switch](#5-processes-states-scheduler-context-switch)
6. [Traps and interrupts](#6-traps-and-interrupts)
7. [System calls](#7-system-calls)
8. [fork, exec, exit, wait: the shell runs a command](#8-fork-exec-exit-wait-the-shell-runs-a-command)
9. [Locks, sleep and wakeup](#9-locks-sleep-and-wakeup)
10. [File system](#10-file-system)
11. [Pipes](#11-pipes)
12. [Console and UART](#12-console-and-uart)
13. [Where to read the code](#13-where-to-read-the-code)

---

## 1. The whole system on one page

Three layers: user programs, the kernel, and the QEMU `virt` hardware. User
code enters the kernel only through the trampoline page, by a system call
(`ecall`), an interrupt, or an exception.

```mermaid
flowchart TB
    subgraph USER["User space: U-mode, one page table per process"]
        INIT["init<br/>opens console, forks sh,<br/>reaps orphans forever"]
        SH["sh<br/>read line, fork, exec, wait"]
        PROGS["ls, cat, grep, wc, echo, mkdir, rm,<br/>usertests, forktest, grind, ..."]
        ULIB["ulib.c start(), printf, malloc<br/>usys.S stubs: li a7, SYS_x then ecall"]
    end

    subgraph KERNEL["Kernel: S-mode, one kernel page table shared by all harts"]
        TRAMP["trampoline page, same VA in every page table<br/>uservec: save user regs, switch to kernel satp<br/>userret: switch to user satp, restore regs, sret"]
        TRAP["trap.c<br/>usertrap, kerneltrap, devintr, clockintr"]
        SYSC["syscall.c<br/>syscalls[a7]() table, 22 calls"]
        PROC["proc.c + swtch.S<br/>proc[64], fork / exec / exit / wait / kill<br/>per-CPU scheduler, sleep / wakeup"]
        VM["vm.c + kalloc.c<br/>Sv39 page tables, copyin / copyout,<br/>lazy sbrk faults, free-page list"]
        FS["sysfile.c, file.c, pipe.c, fs.c, log.c, bio.c<br/>fds, pipes, paths, inodes, log, buffer cache"]
        DRV["console.c, uart.c, virtio_disk.c, plic.c"]
    end

    subgraph HW["QEMU virt machine"]
        HARTS["3 RISC-V harts (CPUS=3)<br/>per-hart stimecmp timer"]
        RAM["128 MB RAM<br/>0x80000000 to 0x88000000"]
        PLIC["PLIC"]
        UART["UART 16550<br/>IRQ 10"]
        DISK[("virtio-blk disk<br/>IRQ 1, backed by fs.img")]
    end

    INIT --> ULIB
    SH --> ULIB
    PROGS --> ULIB
    ULIB -->|"ecall"| TRAMP
    TRAMP --> TRAP
    TRAP -->|"scause 8: syscall"| SYSC
    SYSC --> PROC
    SYSC --> VM
    SYSC --> FS
    FS --> DRV
    TRAP -->|"page fault"| VM
    TRAP -->|"timer: yield()"| PROC
    TRAP -->|"return to user"| TRAMP
    TRAMP -->|"sret"| ULIB
    PROC --> VM
    DRV <-->|"MMIO"| UART
    DRV <-->|"MMIO + DMA"| DISK
    UART --> PLIC
    DISK --> PLIC
    PLIC -->|"external interrupt"| TRAP
    HARTS -->|"timer interrupt"| TRAP
    VM --- RAM
```

---

## 2. Build: from source to kernel and fs.img

`make qemu` builds two artifacts: the kernel ELF, which QEMU loads at
`0x80000000`, and `fs.img`, a 2000-block file system that holds `README` and
the 20 user programs.

```mermaid
flowchart LR
    QEMU["qemu-system-riscv64<br/>-machine virt -bios none -m 128M -smp 3 -nographic<br/>-kernel kernel/kernel<br/>-drive fs.img as virtio-blk-device on virtio-mmio-bus.0"]

    subgraph KB["Kernel"]
        KSRC["kernel/*.c, kernel/*.S"] -->|"riscv64 gcc<br/>-march=rv64gc -mcmodel=medany"| KOBJ["kernel/*.o"]
        KOBJ -->|"ld -T kernel.ld<br/>_entry first, at 0x80000000<br/>trampoline on its own page"| KELF["kernel/kernel"]
    end

    subgraph UB["User programs"]
        USYSPL["user/usys.pl"] -->|"perl"| USYSS["user/usys.S<br/>one stub per syscall"]
        USYSS --> ULIBO["ULIB<br/>ulib.o usys.o printf.o umalloc.o"]
        USRC["user/cat.c, sh.c, init.c, ..."] --> UOBJ["user/*.o"]
        UOBJ -->|"ld -T user.ld + ULIB"| UBIN["user/_cat, _sh, _init, ...<br/>20 programs"]
        ULIBO --> UBIN
    end

    subgraph FSB["Disk image"]
        MKFS["mkfs/mkfs<br/>built with host gcc"]
        README["README"]
        IMG[("fs.img<br/>2000 blocks of 1 KB")]
        MKFS --> IMG
    end

    UBIN --> MKFS
    README --> MKFS
    KELF --> QEMU
    IMG --> QEMU
```

---

## 3. Boot: power-on to the shell prompt

Every hart starts at the same instruction. Hart 0 does all one-time setup
while the others spin on `started`. Then every hart enters its own
`scheduler()` loop, and the first process boots the file system from process
context, because disk I/O has to sleep.

```mermaid
sequenceDiagram
    autonumber
    participant ROM as QEMU reset ROM (0x1000)
    participant H0 as hart 0
    participant HN as harts 1 and 2
    participant P1 as first process (pid 1)
    participant DISK as virtio disk

    ROM->>H0: jump to _entry at 0x80000000 in M-mode, paging off
    ROM->>HN: same jump, all harts at once
    Note over H0,HN: entry.S: sp = stack0 + (hartid + 1) * 4096, call start()
    H0->>H0: start(): mstatus.MPP = S, mepc = main, satp = 0
    H0->>H0: medeleg = mideleg = 0xffff (all traps go to S-mode)<br/>sie: external + timer, PMP: S-mode may access all RAM
    H0->>H0: timerinit(): enable Sstc, stimecmp = time + 1000000
    H0->>H0: tp = mhartid, then mret drops to S-mode in main()
    HN->>HN: same start() path, then main() spins until started == 1
    Note over H0: main() on hart 0 runs every init step once
    H0->>H0: consoleinit (uartinit, devsw[CONSOLE]), printkinit
    H0->>H0: kinit: freerange(end, PHYSTOP) fills the free-page list
    H0->>H0: kvminit: kvmmake maps UART, virtio, PLIC, kernel text RX,<br/>data and RAM RW, trampoline, 64 kernel stacks
    H0->>H0: kvminithart: satp = kernel page table, sfence.vma (paging on)
    H0->>H0: procinit, trapinit, trapinithart (stvec = kernelvec)
    H0->>H0: plicinit (IRQ priorities), plicinithart (enable UART + disk for hart 0)
    H0->>H0: binit (buffer cache), iinit (inode table), fileinit (file table)
    H0->>DISK: virtio_disk_init: reset, negotiate features, set up queue 0
    H0->>P1: userinit: allocproc() (trapframe, page table, context.ra = forkret)<br/>cwd = root inode (iget only, no disk read), state = RUNNABLE
    H0->>HN: started = 1 (release store)
    HN->>HN: kvminithart, trapinithart, plicinithart
    par every hart
        H0->>H0: scheduler()
    and
        HN->>HN: scheduler()
    end
    Note over P1: whichever hart sees P1 RUNNABLE first switches to it
    H0->>P1: that hart's scheduler swtches to P1, which starts in forkret()
    P1->>P1: forkret: release p->lock, first == 1
    P1->>DISK: fsinit(ROOTDEV): read superblock, initlog replays any committed log,<br/>ireclaim frees orphaned inodes
    P1->>DISK: kexec("/init"): load ELF into a new page table, build stack
    P1->>P1: prepare_return, jump to userret in the trampoline with user satp
    P1->>P1: sret to user mode at ulib start(), which calls init main()
    Note over P1: init: open("console") or mknod it first, dup(0) twice for fds 1 and 2
    P1->>P1: fork(), the child exec("sh"), init loops in wait() forever
```

---

## 4. Memory: address spaces, page tables, allocator, sbrk

### 4.1 Physical memory and the two kinds of virtual address space

The kernel identity-maps devices and RAM, so kernel code uses physical
addresses directly. Each process has its own page table. The trampoline page
appears at the same virtual address (`MAXVA - 4096`) in both kinds of page
table, so the trap code keeps running across the `satp` switch.

```mermaid
flowchart LR
    subgraph PHYS["Physical addresses (QEMU virt)"]
        direction TB
        P1["0x00001000 boot ROM"]
        P2["0x02000000 CLINT<br/>(unused: timer is stimecmp)"]
        P3["0x0C000000 PLIC"]
        P4["0x10000000 UART0"]
        P5["0x10001000 virtio disk"]
        P6["0x80000000 kernel text<br/>entry.S first, trampoline page, then etext"]
        P7["kernel data + bss<br/>stack0, cpus[], proc[], bcache, itable, ...<br/>ends at symbol end"]
        P8["free RAM: end to PHYSTOP (0x88000000)<br/>every page handed out by kalloc()"]
        P1 ~~~ P2 ~~~ P3 ~~~ P4 ~~~ P5 ~~~ P6 ~~~ P7 ~~~ P8
    end

    subgraph KVA["Kernel virtual (kernel_pagetable, all harts)"]
        direction TB
        K1["MAXVA - 4096: TRAMPOLINE (R X)<br/>maps the trampoline page in kernel text"]
        K2["KSTACK(0..63): 1 page each (R W)<br/>with an unmapped guard page between stacks"]
        K3["... unmapped ..."]
        K4["0x80000000 to etext: kernel text (R X)<br/>etext to PHYSTOP: data + free RAM (R W)<br/>identity mapped"]
        K5["PLIC, UART0, virtio<br/>identity mapped (R W)"]
        K1 ~~~ K2 ~~~ K3 ~~~ K4 ~~~ K5
    end

    subgraph UVA["User virtual (p->pagetable, one per process)"]
        direction TB
        U1["MAXVA - 4096: TRAMPOLINE (R X, no U)<br/>same physical page as the kernel's"]
        U2["TRAPFRAME (R W, no U)<br/>this process's p->trapframe page"]
        U3["... unmapped ..."]
        U4["heap up to p->sz<br/>sbrk maps now, sbrklazy maps on first touch"]
        U5["user stack: USERSTACK = 1 page (R W U)"]
        U6["guard page (PTE_U cleared by uvmclear)"]
        U7["data + bss (R W U)"]
        U8["0x0: text (R X U)<br/>entry point = ulib.c start()"]
        U1 ~~~ U2 ~~~ U3 ~~~ U4 ~~~ U5 ~~~ U6 ~~~ U7 ~~~ U8
    end
```

### 4.2 Sv39 address translation

The hardware walks this tree on every TLB miss. `walk()` in vm.c does the same
walk in software when the kernel builds or reads page tables. `MAXVA` is
`2^38`, one bit below the Sv39 limit, so addresses never need sign extension.

```mermaid
flowchart LR
    SATP["satp<br/>mode 8 (Sv39) + root page PPN"] --> L2["level-2 page<br/>512 PTEs<br/>index = VA bits 38-30"]
    L2 -->|"valid, no R/W/X:<br/>points to next level"| L1["level-1 page<br/>index = VA bits 29-21"]
    L1 -->|"valid, no R/W/X"| L0["level-0 page<br/>index = VA bits 20-12"]
    L0 -->|"leaf PTE<br/>PPN + flags V R W X U"| PA["physical page<br/>+ VA bits 11-0 as offset"]
    L2 -.->|"walk(alloc=1) creates missing<br/>table pages with kalloc()"| L1
```

### 4.3 Physical page allocator

There is one free list of 4 KB pages, protected by one spinlock. Every
dynamic kernel allocation is a whole page.

```mermaid
flowchart TB
    subgraph KALLOC["kalloc.c"]
        KINIT["kinit()<br/>freerange(end, PHYSTOP): kfree every page once at boot"]
        KF["kfree(pa)<br/>panic unless page-aligned and in end..PHYSTOP<br/>fill with 1s, push on list"]
        FL[("kmem.freelist<br/>struct run list, kmem.lock")]
        KA["kalloc()<br/>pop head, fill with 5s<br/>returns 0 when out of memory"]
        KINIT --> KF --> FL --> KA
    end
    KA --> PT["page-table pages<br/>walk(alloc=1), uvmcreate, kvmmake"]
    KA --> UP["user pages<br/>uvmalloc: exec, eager sbrk<br/>uvmcopy: fork copies every page<br/>vmfault: lazy sbrk"]
    KA --> TF["one trapframe page per process<br/>allocproc"]
    KA --> KS["64 kernel stacks at boot<br/>proc_mapstacks"]
    KA --> MISC["pipe buffers, exec argv copies,<br/>virtio desc / avail / used rings"]
```

### 4.4 Growing memory: eager and lazy sbrk

```mermaid
flowchart TD
    CALL["user: sbrk(n) calls sys_sbrk(n, SBRK_EAGER)<br/>sbrklazy(n) calls sys_sbrk(n, SBRK_LAZY)"] --> D1{"eager, or n negative?"}
    D1 -->|"yes"| GP["growproc(n)<br/>n positive: uvmalloc maps zeroed pages now<br/>n negative: uvmdealloc unmaps and kfrees"]
    D1 -->|"no: lazy"| LZ["p->sz += n only<br/>no pages mapped yet"]
    LZ --> T1["user load or store in the new range"]
    T1 --> PF["page fault, scause 13 (load) or 15 (store)"]
    LZ --> T2["kernel copyin / copyout / copyinstr<br/>into that range, e.g. read() into a buffer"]
    PF --> VF["vmfault(pagetable, p->sz, va)<br/>va below p->sz and not mapped?<br/>kalloc, zero, map R W U"]
    T2 --> VF
    VF -->|"fails: va beyond sz or no memory"| KILL["user fault: process killed<br/>kernel copy: syscall returns -1"]
```

---

## 5. Processes: states, scheduler, context switch

### 5.1 Process states

`p->state` is protected by `p->lock`. A process is killed by setting
`p->killed`. It exits the next time it reaches a check point in `usertrap()`
or in a sleep loop that tests `killed()`.

```mermaid
stateDiagram-v2
    [*] --> UNUSED: procinit()
    UNUSED --> USED: allocproc() from userinit() or kfork()
    USED --> UNUSED: allocation failed, freeproc()
    USED --> RUNNABLE: setup finished
    RUNNABLE --> RUNNING: scheduler() picks it and calls swtch()
    RUNNING --> RUNNABLE: yield() after a timer interrupt
    RUNNING --> SLEEPING: sleep() while chan is still set
    SLEEPING --> RUNNABLE: wakeup(chan) or kkill(pid)
    RUNNING --> ZOMBIE: kexit(status)
    ZOMBIE --> UNUSED: parent kwait() calls freeproc()
    note right of ZOMBIE: keeps its slot and xstate until the parent reaps it
```

### 5.2 The scheduler and swtch

Each CPU runs `scheduler()` on its boot stack. A process never switches
directly to another process: it switches to its CPU's scheduler thread, which
switches to the next one. `p->lock` is held across the switch in both
directions. `swtch` saves only `ra`, `sp` and `s0`-`s11`; the caller-saved
registers were already saved by the C calling convention.

```mermaid
sequenceDiagram
    participant A as process A kernel thread
    participant S as scheduler() on this CPU
    participant B as process B kernel thread

    Note over A: timer interrupt, usertrap or kerneltrap calls yield()
    A->>A: acquire(A->lock), A->state = RUNNABLE
    A->>A: sched(): panic unless lock held, noff == 1,<br/>state not RUNNING, interrupts off
    A->>S: swtch(A->context, cpu->context)
    Note over S: resumes just after its own earlier swtch call
    S->>S: cpu->proc = 0, release(A->lock)
    loop forever
        S->>S: intr_on() then intr_off(), so pending interrupts are taken
        S->>S: for each p in proc[64]: acquire(p->lock)
        alt p->state == RUNNABLE
            S->>S: p->state = RUNNING, cpu->proc = p
            S->>B: swtch(cpu->context, B->context)
            Note over B: resumes inside its own sched() call,<br/>or in forkret() if it is a new process
            B->>B: release(B->lock) in yield, sleep or forkret
            B->>B: run, then return to user through prepare_return and userret
        else not RUNNABLE
            S->>S: release(p->lock), try the next slot
        end
        S->>S: nothing RUNNABLE in the whole scan: wfi until an interrupt
    end
```

---

## 6. Traps and interrupts

### 6.1 From user space into the kernel and back

`stvec` points at `uservec` while user code runs and at `kernelvec` while
kernel code runs. `usertrap()` changes it on entry and `prepare_return()`
changes it back.

```mermaid
flowchart TD
    EV(["User code: ecall, interrupt, or exception"]) --> HW["hardware<br/>sepc = pc, scause, stval<br/>SPP = U, SPIE = SIE, SIE = 0<br/>pc = stvec"]
    HW --> UV["trampoline.S uservec (user page table)<br/>save 31 user regs into TRAPFRAME<br/>sp = kernel_sp, tp = kernel_hartid<br/>satp = kernel_satp, sfence.vma<br/>jump to kernel_trap = usertrap"]
    UV --> UT["trap.c usertrap()<br/>stvec = kernelvec<br/>trapframe->epc = sepc"]
    UT --> C1{"scause"}
    C1 -->|"8: ecall"| SYS["if killed: kexit(-1)<br/>epc += 4<br/>intr_on()<br/>syscall()"]
    C1 -->|"interrupt"| DEV["devintr()<br/>1 = device, 2 = timer"]
    C1 -->|"13 or 15: page fault"| VMF{"vmfault(): lazy page<br/>below p->sz?"}
    VMF -->|"yes"| MAP["map a zeroed page"]
    VMF -->|"no"| BAD
    C1 -->|"anything else"| BAD["print scause, sepc, stval<br/>setkilled(p)"]
    SYS --> K1
    DEV --> K1
    MAP --> K1
    BAD --> K1
    K1{"killed(p)?"} -->|"yes"| EXIT["kexit(-1), never returns"]
    K1 -->|"no"| T1{"timer interrupt?"}
    T1 -->|"yes"| Y["yield(): other processes run,<br/>execution resumes here later"]
    T1 -->|"no"| PR
    Y --> PR["prepare_return()<br/>intr_off, stvec = uservec<br/>trapframe: kernel_satp, kernel_sp,<br/>kernel_trap, kernel_hartid<br/>sstatus: SPP = U, SPIE = 1, sepc = epc"]
    PR --> RET["usertrap returns user satp in a0<br/>to userret in the trampoline"]
    RET --> UR["userret: fence.i, satp = user page table,<br/>sfence.vma, restore regs from TRAPFRAME"]
    UR --> SRET(["sret: user mode at epc, interrupts on"])
```

### 6.2 Traps taken while already in the kernel

Kernel code never page-faults on purpose, so any exception in S-mode is a
panic. Only device and timer interrupts are expected.

```mermaid
flowchart TD
    KEV(["interrupt while kernel code runs<br/>stvec = kernelvec"]) --> KV["kernelvec.S<br/>push 256-byte frame of caller-saved regs<br/>on the current kernel stack"]
    KV --> KT["kerneltrap()<br/>save sepc and sstatus<br/>panic if the trap came from U-mode or interrupts are on"]
    KT --> KD{"devintr()"}
    KD -->|"0: exception"| KP["panic: kerneltrap"]
    KD -->|"1: device"| KR
    KD -->|"2: timer"| KY{"a process is running<br/>on this CPU?"}
    KY -->|"yes"| KYI["yield()"]
    KY -->|"no: scheduler"| KR
    KYI --> KR["restore sepc and sstatus<br/>(yield may have changed them)"]
    KR --> KV2["kernelvec: pop regs, sret"]
```

### 6.3 Interrupt sources and devintr()

```mermaid
flowchart LR
    TIMER["per-hart timer<br/>time reaches stimecmp"] -->|"scause: interrupt 5"| DT["devintr(): timer"]
    UARTI["UART IRQ 10<br/>byte received or TX idle"] --> PLIC["PLIC<br/>priority 1 for IRQ 1 and 10<br/>enabled for every hart's S-mode"]
    DISKI["virtio IRQ 1<br/>request finished"] --> PLIC
    PLIC -->|"scause: interrupt 9"| DE["devintr(): external"]
    DE --> CL["plic_claim()<br/>read this hart's claim register"]
    CL -->|"irq 10"| UI["uartintr()<br/>ack ISR, wakeup(tx_chan) if TX idle,<br/>consoleintr(c) for each received byte"]
    CL -->|"irq 1"| VI["virtio_disk_intr()<br/>ack, walk used ring,<br/>b->disk = 0, wakeup(b)"]
    UI --> CO["plic_complete(irq)<br/>return 1"]
    VI --> CO
    DT --> CK["clockintr()<br/>hart 0 only: ticks++, wakeup(ticks)<br/>every hart: stimecmp = time + 1000000 (about 0.1 s)"]
    CK --> R2["return 2: caller calls yield()"]
```

---

## 7. System calls

A system call is an `ecall` with the call number in `a7` and arguments in
`a0`-`a5`. The return value goes back in `a0`. Kernel code never dereferences
user pointers directly: it uses `copyin` and `copyout`, which walk the user
page table and check addresses against `p->sz`.

```mermaid
flowchart LR
    U["user: write(fd, buf, n)"] --> STUB["usys.S stub<br/>li a7, SYS_write (16)<br/>ecall"]
    STUB --> TR["uservec, then usertrap()"]
    TR --> SC["syscall()<br/>num = trapframe->a7<br/>trapframe->a0 = syscalls[num]()<br/>unknown num: print, a0 = -1"]
    SC --> SP["sysproc.c<br/>fork 1, exit 2, wait 3, kill 6, getpid 11,<br/>sbrk 12, pause 13, uptime 14"]
    SC --> SF["sysfile.c<br/>pipe 4, read 5, exec 7, fstat 8, chdir 9,<br/>dup 10, open 15, write 16, mknod 17,<br/>unlink 18, link 19, mkdir 20, close 21"]
    SC --> SL["log.c<br/>sync 22"]
    SP --> AR["argument helpers<br/>argint, argaddr: read trapframe a0..a5<br/>argstr, fetchaddr: copyin / copyinstr<br/>argfd: index p->ofile[]"]
    SF --> AR
    SC -.->|"a0 restored by userret"| U
```

---

## 8. fork, exec, exit, wait: the shell runs a command

What happens when you type `ls` at the `$` prompt. `fork` copies all user
memory eagerly. `exec` builds the new image in a fresh page table and only
discards the old one once nothing can fail.

```mermaid
sequenceDiagram
    autonumber
    participant SH as sh (parent)
    participant K as kernel
    participant C as child
    participant FS as file system

    SH->>K: fork(): ecall with a7 = SYS_fork
    K->>K: kfork: allocproc() takes an UNUSED slot, new pid,<br/>trapframe page, page table with trampoline and trapframe,<br/>context.ra = forkret, context.sp = top of kstack
    K->>K: uvmcopy: kalloc and copy every mapped user page
    K->>K: copy trapframe, child a0 = 0,<br/>filedup each open file, idup(cwd), copy name
    K->>K: np->parent = p (under wait_lock), np->state = RUNNABLE
    K-->>SH: return child pid in a0
    SH->>K: wait(0)
    K->>K: kwait: scan proc[] for children, none is ZOMBIE yet
    K->>K: sleep_prepare(p), release wait_lock, sleep()
    Note over C: some CPU's scheduler picks the child
    C->>C: forkret: release p->lock, prepare_return, userret
    C->>C: fork() returns 0, runcmd(parsecmd("ls")), exec("ls", argv)
    C->>K: sys_exec: copy path, and each argv string into its own kalloc page
    K->>FS: kexec: begin_op, namei(path), ilock, read ELF header, check magic
    K->>K: proc_pagetable(): new, empty user page table
    loop each ELF program header of type LOAD
        K->>K: uvmalloc(sz, vaddr + memsz, flags2perm(flags))
        K->>FS: loadseg: walkaddr, then readi the file bytes into those pages
    end
    K->>FS: iunlockput, end_op
    K->>K: map guard page + 1 stack page, clear PTE_U on the guard,<br/>push argv strings, then the argv[] pointer array
    K->>K: commit: p->pagetable = new, p->sz, epc = elf.entry,<br/>sp = stack pointer, free the old page table and memory
    K-->>C: return argc in a0 (argv pointer in a1)
    C->>C: ulib start() calls main(argc, argv), ls reads the directory and writes to fd 1
    C->>K: main returns, start() calls exit(0)
    K->>FS: kexit: fileclose every open fd, then begin_op, iput(cwd), end_op
    K->>K: acquire wait_lock, reparent children to init, wakeup(parent)
    K->>K: acquire p->lock, xstate = status, state = ZOMBIE,<br/>release wait_lock, sched() never returns
    Note over SH: wakeup set the parent RUNNABLE, sleep() returns in kwait
    K->>K: kwait finds the ZOMBIE child: copyout xstate (skipped for wait(0)),<br/>freeproc frees trapframe, page table and user memory
    K-->>SH: wait returns the child pid, sh prints the next prompt
```

---

## 9. Locks, sleep and wakeup

### 9.1 Spinlocks

`acquire()` disables interrupts before it spins, so an interrupt handler on
the same CPU can never deadlock against the code it interrupted. `push_off()`
and `pop_off()` nest, and interrupts come back on only when the outermost
lock is released.

```mermaid
flowchart LR
    A1["acquire(lk)"] --> A2["push_off()<br/>clear SIE, noff++<br/>save previous SIE in intena<br/>when noff was 0"]
    A2 --> A3{"amoswap.w.aq<br/>old value of locked<br/>was 0?"}
    A3 -->|"no: spin"| A3
    A3 -->|"yes"| A4["lk->cpu = mycpu()<br/>critical section runs"]
    A4 --> R1["release(lk)<br/>lk->cpu = 0<br/>fence rw,w then store 0"]
    R1 --> R2["pop_off()<br/>noff--, turn interrupts on<br/>if noff == 0 and intena"]
```

Sleep locks (`sleeplock.c`) are a spinlock plus a `locked` flag. A waiter
sleeps on the lock's address, so they can be held across disk I/O. Inodes,
buffers and the UART transmitter use them.

| Lock | Kind | Protects | File |
|---|---|---|---|
| `kmem.lock` | spin | free page list | kalloc.c |
| `pid_lock` | spin | `nextpid` | proc.c |
| `wait_lock` | spin | every `p->parent`, the exit/wait handoff. Acquire before any `p->lock` | proc.c |
| `p->lock` | spin | `state`, `chan`, `killed`, `xstate`, `pid`. Held across `swtch` | proc.c |
| `tickslock` | spin | `ticks` | trap.c |
| `cons.lock` | spin | console input buffer | console.c |
| `tx_lock` | sleep | one writer at a time in `uartwrite` | uart.c |
| `pr.lock` | spin | keeps concurrent `printk` output from interleaving | printk.c |
| `ftable.lock` | spin | `struct file` reference counts | file.c |
| `pi->lock` | spin | one pipe's buffer and counters | pipe.c |
| `itable.lock` | spin | inode table slots: `ref`, `dev`, `inum` | fs.c |
| `ip->lock` | sleep | one inode's contents: `valid`, `type`, `size`, `addrs` | fs.c |
| `log.lock` | spin | log header, `outstanding`, `committing` | log.c |
| `bcache.lock` | spin | buffer list, `refcnt`, which block a buffer holds | bio.c |
| `b->lock` | sleep | one buffer's data | bio.c |
| `disk.vdisk_lock` | spin | virtio descriptors, rings, `info[]` | virtio_disk.c |

### 9.2 sleep_prepare, sleep, wakeup: why no wakeup is lost

The waiter registers its channel before it drops the lock that protects the
condition. `wakeup()` clears `p->chan`, so a wakeup that arrives between the
release and `sleep()` makes `sleep()` return at once and is not lost. The
example uses a pipe, but disk I/O, `wait`, `pause`, the console and the log
all follow the same pattern.

```mermaid
sequenceDiagram
    participant W as waiter (piperead)
    participant CL as condition lock (pi->lock)
    participant PL as waiter p->lock
    participant WK as waker (pipewrite)

    W->>CL: acquire, test condition: pipe is empty
    W->>PL: sleep_prepare(nread address): p->chan = chan
    W->>CL: release
    alt waker runs before sleep()
        WK->>CL: acquire, add bytes
        WK->>PL: wakeup(chan): p->chan matches, set p->chan = 0
        WK->>CL: release
        W->>PL: sleep(): p->chan is 0, return at once
    else waker runs after sleep()
        W->>PL: sleep(): p->chan still set, state = SLEEPING, sched()
        WK->>CL: acquire, add bytes
        WK->>PL: wakeup(chan): p->chan = 0, SLEEPING becomes RUNNABLE
        WK->>CL: release
        Note over W: a scheduler runs W later, sleep() returns
    end
    W->>CL: acquire again and re-test the condition in a while loop
```

---

## 10. File system

### 10.1 Layers

Each layer uses only the one below it. Every system call that modifies the
disk wraps its work in `begin_op()` and `end_op()`, so the whole call commits
atomically.

```mermaid
flowchart TB
    S1["system calls (sysfile.c)<br/>open, read, write, close, dup, pipe, fstat,<br/>link, unlink, mkdir, mknod, chdir, exec"]
    FDT["file descriptors (file.c)<br/>p->ofile[16] points into ftable.file[100]<br/>struct file: type, ref, off, readable, writable"]
    FT{"f->type"}
    PIPE["pipe.c<br/>512-byte ring in a kalloc page"]
    DEVSW["devsw[major]<br/>major 1 = CONSOLE: consoleread, consolewrite"]
    NAMEI["pathnames (fs.c namex)<br/>start at root inode or p->cwd,<br/>skipelem one component at a time"]
    DIR["directories (fs.c)<br/>dirlookup, dirlink<br/>dirent = 2-byte inum + 14-byte name"]
    INO["inodes (fs.c)<br/>itable[50]: iget, ilock, iunlock, iput<br/>readi, writei, bmap, itrunc, ialloc, iupdate<br/>12 direct + 256 indirect = 268 KB max file"]
    LOGL["log (log.c)<br/>begin_op, log_write, end_op<br/>commit: write_log, write_head, install_trans"]
    BUF["buffer cache (bio.c)<br/>30 bufs, LRU list, a sleeplock per buf<br/>bread, bwrite, brelse, bpin, bunpin"]
    VD["virtio driver (virtio_disk.c)<br/>3-descriptor request, notify,<br/>sleep until the completion IRQ"]
    DISK[("fs.img")]

    S1 --> FDT --> FT
    FT -->|"FD_PIPE"| PIPE
    FT -->|"FD_DEVICE"| DEVSW
    FT -->|"FD_INODE"| INO
    S1 -->|"open, create, link, unlink,<br/>chdir, exec"| NAMEI
    NAMEI --> DIR --> INO
    INO -->|"bread, brelse"| BUF
    INO -->|"log_write instead of bwrite"| LOGL
    LOGL -->|"bwrite at commit"| BUF
    BUF --> VD --> DISK
```

### 10.2 On-disk layout

Built by `mkfs` from `param.h` and `fs.h`: `FSSIZE = 2000` blocks of 1 KB.
`mkfs` prints `nmeta 47 (boot, super, log blocks 31, inode blocks 13, bitmap
blocks 1) blocks 1953 total 2000`.

```mermaid
flowchart LR
    B0["block 0<br/>boot, unused"] --- B1["block 1<br/>superblock<br/>magic 0x10203040"] --- LH["block 2<br/>log header<br/>n + block numbers"] --- LD["blocks 3-32<br/>30 log blocks"] --- IN["blocks 33-45<br/>200 inodes<br/>64 bytes each, 16 per block"] --- BM["block 46<br/>free-block bitmap"] --- DATA["blocks 47-1999<br/>1953 data blocks"]
```

### 10.3 Reading a file: `read(fd, buf, n)` with a cache miss

```mermaid
sequenceDiagram
    autonumber
    participant U as user process
    participant F as file.c
    participant I as fs.c inode layer
    participant B as bio.c buffer cache
    participant V as virtio_disk.c
    participant D as virtio disk (QEMU)
    participant IRQ as trap.c + PLIC (any hart)

    U->>F: read(): ecall, sys_read, argfd, fileread
    F->>I: ilock(ip): acquiresleep, load dinode with bread if not valid
    F->>I: readi(ip, user_dst = 1, buf, f->off, n)
    loop each 1 KB block in the range
        I->>I: bmap(ip, block index): addrs[0..11] or the indirect block
        I->>B: bread(dev, blockno)
        B->>B: bget: cached? refcnt++ and lock it.<br/>else recycle the least recently used buf with refcnt 0
        alt buf not valid (cache miss)
            B->>V: virtio_disk_rw(b, write = 0)
            V->>V: acquire vdisk_lock, alloc 3 descriptors:<br/>request header, b->data, status byte
            V->>V: b->disk = 1, record b in info[]
            V->>D: add head to avail ring, write QUEUE_NOTIFY
            V->>V: while b->disk == 1: sleep on b (vdisk_lock released)
            D->>D: DMA the block into b->data, add to used ring
            D->>IRQ: raise IRQ 1
            IRQ->>V: devintr, plic_claim, virtio_disk_intr()
            V->>V: ack, b->disk = 0, wakeup(b)
            IRQ->>IRQ: plic_complete(1)
            V->>V: reader wakes, frees the descriptor chain
            V-->>B: return, b->valid = 1
        end
        B-->>I: locked buf
        I->>U: either_copyout: copyout through the user page table
        I->>B: brelse: unlock, move to the MRU end when refcnt reaches 0
    end
    F->>F: f->off += bytes read, iunlock(ip)
    F-->>U: byte count in a0
```

### 10.4 Writing a file and committing the log

`filewrite()` splits a large write into transactions of at most 3072 bytes.
A non-aligned chunk can touch 4 data blocks, and with the inode, indirect and
bitmap blocks it still fits in one op's 10-block budget. `begin_op()`
reserves `MAXOPBLOCKS = 10` log blocks for each running FS call, so at most
three calls run in a transaction at once.

```mermaid
sequenceDiagram
    autonumber
    participant P as process
    participant LG as log.c
    participant I as fs.c inode layer
    participant B as buffer cache
    participant D as disk

    P->>LG: begin_op()
    alt committing, or the log could overflow
        LG->>LG: sleep on log, retry when woken
    else room left
        LG->>LG: outstanding += 1
    end
    P->>I: ilock(ip), writei(ip, user_src = 1, src, off, n)
    loop each block touched
        I->>B: bmap (may balloc: set a bitmap bit, zero the new block), bread
        I->>I: either_copyin user bytes into b->data
        I->>LG: log_write(b): add blockno to lh.block[] unless already there,<br/>bpin(b) so the cache cannot evict it
        I->>B: brelse(b): still dirty in memory, nothing written to disk yet
    end
    I->>LG: iupdate(ip): log_write the inode block
    P->>LG: iunlock(ip), end_op()
    alt other FS calls still outstanding
        LG->>LG: outstanding -= 1, wakeup(log)
    else this was the last one
        LG->>LG: committing = 1, commit() with no locks held
        LG->>D: write_log(): copy each logged block from the cache into log blocks 3 and up
        LG->>D: write_head(): header with n and block numbers. This is the commit point
        LG->>D: install_trans(): copy each block to its home location, bunpin
        LG->>D: write_head() with n = 0: transaction erased
        LG->>LG: committing = 0, ncommit += 1, wakeup(log)
    end
    Note over LG,D: At boot, recover_from_log() replays any header with n above 0, then clears it.<br/>A crash before the commit point loses the whole call. A crash after it is replayed.
```

---

## 11. Pipes

A pipe is two `struct file`s, one read end and one write end, that share one
`kalloc` page holding the counters and a 512-byte ring. `nread` and `nwrite`
only increase, so the buffer is full when `nwrite == nread + 512`.

```mermaid
flowchart LR
    subgraph WR["pipewrite(addr, n)"]
        W0{"read end closed<br/>or killed?"} -->|"yes"| WX["return -1"]
        W0 -->|"no"| W1{"buffer full?"}
        W1 -->|"yes"| W2["wakeup(nread)<br/>sleep on nwrite"]
        W2 --> W0
        W1 -->|"no"| W3["copyin 1 byte<br/>data[nwrite++ mod 512]"]
        W3 --> W4{"more bytes?"}
        W4 -->|"yes"| W0
        W4 -->|"no"| W5["wakeup(nread)<br/>return n"]
    end
    subgraph RD["piperead(addr, n)"]
        R1{"empty and<br/>write end open?"} -->|"yes"| R2["sleep on nread<br/>(return -1 if killed)"]
        R2 --> R1
        R1 -->|"no"| R3["copyout up to n bytes"]
        R3 --> R4["wakeup(nwrite)<br/>return count, 0 means EOF"]
    end
```

`pipeclose()` clears `readopen` or `writeopen` and wakes the other side. The
page is freed when both ends are closed.

---

## 12. Console and UART

Input is interrupt-driven and line-buffered. Output from `write()` sleeps
while the UART is busy. Kernel `printk` bypasses both paths: it calls
`uartputc_sync()`, which spins with interrupts off, so it works inside
interrupt handlers and during a panic.

```mermaid
sequenceDiagram
    participant KB as terminal
    participant UART as UART 16550
    participant ISR as uartintr + consoleintr
    participant RD as reader (sh in consoleread)
    participant WR as writer (printf in consolewrite)

    KB->>UART: byte arrives in RHR
    UART->>ISR: IRQ 10 through the PLIC, devintr
    ISR->>ISR: uartgetc loop, consoleintr(c) under cons.lock
    ISR->>UART: echo with consputc and uartputc_sync
    Note over ISR: ctrl-H or DEL erases, ctrl-U kills the line,<br/>ctrl-P prints the process list, ctrl-D is end of file
    ISR->>RD: on newline, ctrl-D or a full 128-byte buffer:<br/>cons.w = cons.e, wakeup(cons.r)
    RD->>RD: copy bytes from cons.buf to user memory up to the newline
    WR->>WR: either_copyin 32 bytes at a time, uartwrite takes tx_lock
    loop each byte
        alt LSR reports TX idle
            WR->>UART: write the byte to THR
        else UART busy
            WR->>WR: sleep on tx_chan
            UART->>ISR: TX-idle interrupt
            ISR->>WR: wakeup(tx_chan)
        end
    end
```

---

## 13. Where to read the code

| Topic | Functions | Code |
|---|---|---|
| Boot | `_entry`, `start`, `timerinit`, `main` | [entry.S:7](../xv6/xv6/kernel/entry.S#L7), [start.c:15](../xv6/xv6/kernel/start.c#L15), [main.c:11](../xv6/xv6/kernel/main.c#L11) |
| First process | `userinit`, `forkret`, init `main` | [proc.c:219](../xv6/xv6/kernel/proc.c#L219), [proc.c:513](../xv6/xv6/kernel/proc.c#L513), [init.c:15](../xv6/xv6/user/init.c#L15) |
| Memory layout | constants | [memlayout.h](../xv6/xv6/kernel/memlayout.h), [param.h](../xv6/xv6/kernel/param.h) |
| Page tables | `kvmmake`, `walk`, `mappages`, `uvmalloc`, `uvmcopy` | [vm.c:22](../xv6/xv6/kernel/vm.c#L22), [vm.c:99](../xv6/xv6/kernel/vm.c#L99), [vm.c:147](../xv6/xv6/kernel/vm.c#L147), [vm.c:218](../xv6/xv6/kernel/vm.c#L218), [vm.c:299](../xv6/xv6/kernel/vm.c#L299) |
| User copies, lazy faults | `copyout`, `copyin`, `vmfault` | [vm.c:345](../xv6/xv6/kernel/vm.c#L345), [vm.c:383](../xv6/xv6/kernel/vm.c#L383), [vm.c:459](../xv6/xv6/kernel/vm.c#L459) |
| Allocator | `kinit`, `kfree`, `kalloc` | [kalloc.c:27](../xv6/xv6/kernel/kalloc.c#L27) |
| sbrk | `sys_sbrk`, `growproc` | [sysproc.c:40](../xv6/xv6/kernel/sysproc.c#L40), [proc.c:236](../xv6/xv6/kernel/proc.c#L236) |
| Process table | `struct proc`, `allocproc`, `freeproc` | [proc.h:82](../xv6/xv6/kernel/proc.h#L82), [proc.c:110](../xv6/xv6/kernel/proc.c#L110), [proc.c:156](../xv6/xv6/kernel/proc.c#L156) |
| Scheduling | `scheduler`, `sched`, `yield`, `swtch` | [proc.c:429](../xv6/xv6/kernel/proc.c#L429), [proc.c:480](../xv6/xv6/kernel/proc.c#L480), [proc.c:501](../xv6/xv6/kernel/proc.c#L501), [swtch.S:9](../xv6/xv6/kernel/swtch.S#L9) |
| fork, exit, wait, kill | `kfork`, `kexit`, `kwait`, `kkill` | [proc.c:259](../xv6/xv6/kernel/proc.c#L259), [proc.c:326](../xv6/xv6/kernel/proc.c#L326), [proc.c:370](../xv6/xv6/kernel/proc.c#L370), [proc.c:602](../xv6/xv6/kernel/proc.c#L602) |
| exec | `kexec`, `loadseg` | [exec.c:28](../xv6/xv6/kernel/exec.c#L28), [exec.c:157](../xv6/xv6/kernel/exec.c#L157) |
| User traps | `uservec`, `userret`, `usertrap`, `prepare_return` | [trampoline.S:22](../xv6/xv6/kernel/trampoline.S#L22), [trampoline.S:101](../xv6/xv6/kernel/trampoline.S#L101), [trap.c:38](../xv6/xv6/kernel/trap.c#L38), [trap.c:101](../xv6/xv6/kernel/trap.c#L101) |
| Kernel traps, interrupts | `kernelvec`, `kerneltrap`, `devintr`, `clockintr`, PLIC | [kernelvec.S:12](../xv6/xv6/kernel/kernelvec.S#L12), [trap.c:137](../xv6/xv6/kernel/trap.c#L137), [trap.c:188](../xv6/xv6/kernel/trap.c#L188), [trap.c:167](../xv6/xv6/kernel/trap.c#L167), [plic.c](../xv6/xv6/kernel/plic.c) |
| System calls | stubs, `syscall`, argument helpers | [usys.pl](../xv6/xv6/user/usys.pl), [syscall.c:137](../xv6/xv6/kernel/syscall.c#L137), [syscall.c:35](../xv6/xv6/kernel/syscall.c#L35) |
| Locks | `acquire`, `release`, `push_off`, sleep locks | [spinlock.c:22](../xv6/xv6/kernel/spinlock.c#L22), [spinlock.c:93](../xv6/xv6/kernel/spinlock.c#L93), [sleeplock.c:22](../xv6/xv6/kernel/sleeplock.c#L22) |
| Sleep and wakeup | `sleep_prepare`, `sleep`, `wakeup` | [proc.c:548](../xv6/xv6/kernel/proc.c#L548), [proc.c:563](../xv6/xv6/kernel/proc.c#L563), [proc.c:577](../xv6/xv6/kernel/proc.c#L577) |
| File syscalls | `sys_open`, `create`, `sys_exec`, `sys_pipe` | [sysfile.c:329](../xv6/xv6/kernel/sysfile.c#L329), [sysfile.c:259](../xv6/xv6/kernel/sysfile.c#L259), [sysfile.c:459](../xv6/xv6/kernel/sysfile.c#L459), [sysfile.c:502](../xv6/xv6/kernel/sysfile.c#L502) |
| File table | `fileread`, `filewrite`, `fileclose` | [file.c:107](../xv6/xv6/kernel/file.c#L107), [file.c:135](../xv6/xv6/kernel/file.c#L135), [file.c:60](../xv6/xv6/kernel/file.c#L60) |
| Inodes, directories, paths | `ilock`, `iput`, `bmap`, `readi`, `writei`, `dirlookup`, `namex` | [fs.c:295](../xv6/xv6/kernel/fs.c#L295), [fs.c:350](../xv6/xv6/kernel/fs.c#L350), [fs.c:421](../xv6/xv6/kernel/fs.c#L421), [fs.c:510](../xv6/xv6/kernel/fs.c#L510), [fs.c:544](../xv6/xv6/kernel/fs.c#L544), [fs.c:592](../xv6/xv6/kernel/fs.c#L592), [fs.c:692](../xv6/xv6/kernel/fs.c#L692) |
| Log | `begin_op`, `end_op`, `commit`, `log_write`, recovery | [log.c:129](../xv6/xv6/kernel/log.c#L129), [log.c:155](../xv6/xv6/kernel/log.c#L155), [log.c:203](../xv6/xv6/kernel/log.c#L203), [log.c:224](../xv6/xv6/kernel/log.c#L224), [log.c:119](../xv6/xv6/kernel/log.c#L119) |
| Buffer cache | `bget`, `bread`, `brelse` | [bio.c:58](../xv6/xv6/kernel/bio.c#L58), [bio.c:92](../xv6/xv6/kernel/bio.c#L92), [bio.c:117](../xv6/xv6/kernel/bio.c#L117) |
| Disk driver | `virtio_disk_rw`, `virtio_disk_intr` | [virtio_disk.c:216](../xv6/xv6/kernel/virtio_disk.c#L216), [virtio_disk.c:301](../xv6/xv6/kernel/virtio_disk.c#L301) |
| Disk image | `mkfs` | [mkfs.c](../xv6/xv6/mkfs/mkfs.c) |
| Pipes | `pipewrite`, `piperead`, `pipeclose` | [pipe.c:77](../xv6/xv6/kernel/pipe.c#L77), [pipe.c:112](../xv6/xv6/kernel/pipe.c#L112), [pipe.c:59](../xv6/xv6/kernel/pipe.c#L59) |
| Console, UART | `consoleintr`, `consoleread`, `uartwrite`, `uartintr` | [console.c:147](../xv6/xv6/kernel/console.c#L147), [console.c:88](../xv6/xv6/kernel/console.c#L88), [uart.c:80](../xv6/xv6/kernel/uart.c#L80), [uart.c:139](../xv6/xv6/kernel/uart.c#L139) |
| Shell | `main`, `runcmd`, `parsecmd` | [sh.c:146](../xv6/xv6/user/sh.c#L146), [sh.c:59](../xv6/xv6/user/sh.c#L59), [sh.c:334](../xv6/xv6/user/sh.c#L334) |
