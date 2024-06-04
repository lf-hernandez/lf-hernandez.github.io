---
title: "Debugging Kernel Oops"
date: 2024-06-01T19:33:02-04:00
---
## Introduction

When writing or using software, it's not uncommon to run into bad states. Sometimes things break and we're met with a cryptic stack trace. In a web server, this may look like an HTTP 500 response along with a helpful error message hinting at where the issue may lie - **if** you were diligent about adding proper error handling, that is - in the best case. Other times you'll be faced with a vague error message without much to go on. In either case, it requires some sleuthing on the programmer's behalf to get to the bottom of what went wrong. In Linux, there are kernel Oopses. This is a non-fatal, recoverable, error state. As opposed to a kernel panic, which is fatal and non-recoverable.

In this post I will go over how to debug a kernel oops, much of what is here is gathered from sources listed in the reference section.

## Terminology

**Kernel Panic**: This is a state from which the kernel cannot recover. In the event of a kernel panic, the operating system will force a reboot or hang; This is done to prevent data loss. This is triggered by invoking the `panic()` function.

**Kernel Oops**: This is a state from which the kernel can recover. The system detects a fault and kills the culprit process followed by printing a log message with a stack trace. Sometimes an Oops will lead to a panic, but this is not always the case. Unlike a panic, there is no function to invoke this state, it is a consequence of an uncaught exception.

**System.map**: This is a symbol-to-address lookup table. It provides a way for developers to debug kernel panics and oops. Modern kernels (post 2.6* you'll notice some docs and articles mention ksysmoops as a requirement, this is no longer the case) have a config option to enable the translation so you'll want to ensure your config has the following option set:

```bash
CONFIG_KALLSYMS=y
```

In general, an oops log will contain the following:

Error Message and Type: This indicates the nature of the error.
Register Dump: This shows the state of the CPU registers.
Stack Trace: This provides a snapshot of the call stack at the point in which it was triggered.

## Anatomy

Example from Sharma:

```bash
[67.994624] Internal error: Oops: 5 [#1] PREEMPT SMP ARM
```

**timestamp** [ 67.994624] - seconds after boot

**error description** Internal error

**oops code** Oops: 5 - this signals it is an oops report and a code that hints at the nature of the error

**number of occurrences** [#1] - this is the number of occurrences of the oops report

**configuration/architecture** - preempt this is a kernel feature that allows higher priority tasks to interrupt lower priority tasks. smp indicates the kernel is capable of utilizing multi-CPUs. arm indicates the system is running on an ARM processor.

Example from Prabhakar:

```bash
BUG: unable to handle kernel NULL pointer dereference at (null) 
IP: [<ffffffffa03e1012>] my_oops_init+0x12/0x21 [oops] 
PGD 7a719067 PUD 7b2b3067 PMD 0 
Oops: 0002 [#1] SMP
```

In this example, we have:

**error description** NULL pointer derference

**instruction pointer** the address of the faulting instruction, symbol (function name),  and the offset to where the fault occurred.

**page table entries** Page Global Directory, Page Upper Directory, Page Middle Directory. This relates to the multi-level page table structure used by the CPU's memory management unit and displays the state of the memory translation for the faulting address.

**oops code** Oops: 002

**number of occurrences** [#1]

**context** SMP

These examples show some of what you may encounter in the first few lines of a kernel oops, taken from the 2 sources listed in the reference section. In general, you'll first encounter a timestamp, description, and faulting address. From this line, you'll get more in-depth information regarding:

* Instruction Pointer (IP): this points to the CPU register that has the address of the next instruction it will execute.
* Faulting function and offset: this will give you the name of the function where the Oops was triggered as well as the offset. Offset between the beginning of the function to where the Oops was triggered.
* Module: if the Oops was triggered from a module, you will get a reference to it
* Page Directory Entry info
* CPU, PID, and Proc info: e.g. `CPU: 0 PID: 1234 Comm: insmod Tainted`, here we can see on which processor the Oops occurred, the process ID of the task where the Oops was triggered, and the command name of the task. We also get the taint state of the kernel
* Register info: this will be a dump of the registers at the time of the fault
* Call trace, Backtrace, stack trace: this, like in higher-level languages, will give us a view of the function calls leading up to the Oops

### Debugging approach in Sharma's post

Here Sharma starts by looking at the backtrace and pulling the faulting function and offset nearest to the Oops occurrence:

```bash
[   67.994624] Internal error: Oops: 5 [#1] PREEMPT SMP ARM
[   67.994926] CPU: 0    Not tainted  (3.8.13.23-XXXXXXXX #1)
[   67.994996] PC is at add_range+0x14/0x6c
```

Once the symbol is identified, in this case, add_range, a `System.map` look-up can be performed. So from the root of the kernel tree, running:

`grep add_range System.map` will yield the address which corresponds to add_range. This will vary from build to build. For example, in the post, it was `80049f28`, on my build `8115b060`.

With the faulting address in hand, arithmetic can be performed to find the location. Simply adding the offset will yield the location of where the fault was triggered. `80049f28 + 0x14 = 80049F3C`, which relates to the Program Counter register at a later point in the backtrace:

```bash
[   67.995117] pc : [<80049F3C>]
```

This confirms the faulting location, from here the kernel image can be disassembled to debug further.

```bash
objdump -D -S --show-raw-insn --prefix-addresses --line-numbers vmlinux > objdump
```

Breaking down this command:

-D -> Disassemble all sections of the object file (vmlinux)

-s -> Display full contents of sections (includes source code)

--show-raw-ins -> print disassembly instructions in hex as well

--prefix-addresses -> when disassembling, print complete addresses on each line

--line-numbers -> show source line numbers

vmlinux -> uncompressed linux kernel image

The oops indicates that the r0 register is pointing to an invalid address 02120bc0, leading to an invalid memory access when the instruction ldr r3, [r0, #4] is executed (ldr r3, [r0, #4] => r3 = *(r0 + 4) => r0 = 02120bc0 => 02120bc0 + 4 = 02120bc4). This causes the kernel to panic due to an inability to handle the paging request for the address 02120bc4

### Prabhakar's approach

Here we see the use of gdb (Sharma also covered this), specifically on a kernel module with an offending oops. Once the oops is captured:

the module is loaded into gdb

```bash
gdb oops.ko
```

the symbol file is passed along with the location in memory where the module is loaded

```bash
add-symbol-file oops.o 0xffffffffa03e1000
```

the function where the oops was triggered from is disassembled

```bash
disassemble my_oops_init
```

once the faulting function address is identified, we can add the offset to find the exact location of where the oops was triggered and get the symbol for it via

```bash
list *0x000000000000004a
```

which points to the intentional null derefrence

```c
*(int *)0 = 0;
```

### Conclusion

The general approach when it comes to debugging kernel oops is to understand how to interpret the backtrace to pinpoint the faulting function. Using tools like objdump and gdb prove to be very useful in disassembling kernel and debugging the kernel to find the culprit as well as allowing for a deeper understanding of how the machine is running the instructions.

## References

[Debugging Analysis of Kernel panics and Kernel Oopses using System Map, Sanjeev Sharma](https://sanjeev1sharma.wordpress.com/2014/06/23/debugging-analysis-of-kernel-panics-and-kernel-oopses-using-system-map/)

[Understanding a Kernel Oops!,  Surya Prabhakar](https://www.opensourceforu.com/2011/01/understanding-a-kernel-oops/)

[Bug hunting, Kernel Docs, Admin Guide](https://www.kernel.org/doc/html/latest/admin-guide/bug-hunting.html)

[Bug hunting in the Linux Kernel, Desmond Cheong](https://www.desmondcheong.com/blog/2021/05/31/bug-hunting-in-the-linux-kernel/)
