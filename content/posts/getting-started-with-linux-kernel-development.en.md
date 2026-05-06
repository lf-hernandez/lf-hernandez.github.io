---
title: "Getting Started With Linux Kernel Development"
date: 2024-05-09T09:05:57-04:00
summary: "In this post, I cover my experience diving into writing a Loadable Kernel Module (LKM) for the Linux kernel. From setting up the development environment to compiling and testing the module."
tags: ["linux", "kernel", "c"]
categories: ["linux kernel"]
---

Kernel development. Probably not the most exciting topic that comes to mind, but it's been a fascination of mine for some time. A lot of work goes into maintaining this project, and I owe a lot to it.

The goal of this post is to document the initial process of getting into the development side of the Linux kernel. They say you need to crawl before you walk so I will first focus on getting up and running with building and loading the kernel from source, followed by writing and loading a module into the kernel.

## Environment

I'll be working with Ubuntu 23.10 running on a VPS. It's a repurposed box I had initially set up for pentesting labs.

## 1. Setup

It's probably a good idea to give the kernel trees a dedicated home in the filesystem:

```bash
sudo mkdir /kernel_trees
sudo chown felipe:felipe /kernel_trees
cd /kernel_trees
```

I'll need to pull in a few dependencies in order to successfully build the kernel:

```bash
sudo apt update
sudo apt install build-essential libncurses-dev libssl-dev libelf-dev bison flex -y
```

Since I'm working and building from source there's no need to pull the distribution's kernel header files since they're all included in the upstream git repo.

## 2. Pulling the kernel

Linus's tree or Mainline contains the latest changes and bug fixes maintained by Linus Torvalds:

```bash
git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git mainline
```

Living life on the edge is fun but I think a good first place to start for my purposes is the latest stable tree:

```bash
git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git stable
```

## 3. Housekeeping

Cool, so I have the tools I need to build the kernel from source and I have the tree I want to work with. Before I jump into kicking off compilation I want to see what my current system is running:

```bash
$ uname -r
6.5.0-28-generic
```

What about what other versions I have installed from Ubuntu's repo:

```bash
$ apt list --installed | grep linux-image
linux-image-6.5.0-14-generic/mantic-updates,mantic-security,now 6.5.0-14.14 amd64 [installed,automatic]
linux-image-6.5.0-28-generic/mantic-updates,mantic-security,now 6.5.0-28.29 amd64 [installed,automatic]
linux-image-generic/mantic-updates,mantic-security,now 6.5.0.28.28 amd64 [installed,automatic]
```

Ok, so it looks like I'm running 6.5. This is an Ubuntu kernel flavor, maintained and published by Canonical.

What's the latest upstream stable kernel version:

```bash
$ head Makefile -n 6
# SPDX-License-Identifier: GPL-2.0
VERSION = 6
PATCHLEVEL = 8
SUBLEVEL = 9
EXTRAVERSION =
NAME = Hurr durr I'ma ninja sloth
```

Nice, looks like it's 3 minor versions ahead.

## 4. Configuring the kernel

From my understanding, you can go a few routes when it comes to configuring the kernel (e.g. menuconfig, localmodconfig). This is all new to me and over time I will dive deeper into this specific step, but for now, I want to focus on building the kernel with as little deviation from my distribution's flavor. So I'll simply copy the old config into the stable tree:

```bash
cp /boot/config-6.5.0-28-generic .config
```

Not quite done yet though, I still need to generate the actual configuration file for the kernel build process. This is where `make oldconfig` comes into play:

```bash
make oldconfig
```

`make oldconfig` reads the copied configuration file (.config) and presents a series of prompts, asking for input on any new configuration options introduced since the last kernel version. Essentially updating the configuration file to match the options available in the new kernel source tree while retaining the settings from the old configuration file.

I could also leverage `lsmod` and the `localmodconfig` make target to tweak the config to use the modules currently loaded and disable any module options not needed for the loaded modules.

This step is important as it dictates how to build the kernel. Without it, the compilation will fail and you'd see something like this:

```bash
make[3]: *** No rule to make target '.config', needed by 'kernel/config_data'.  Stop.
```

Now, with the config in place, I can kick off the compilation:

```bash
make all
```

But pausing for a second, this is a large codebase so it would make sense to optimize the compilation process by specifying the number of workers we want to throw at it. First, let's see what the server is working with:

```bash
$ lscpu
felipe@localhost:/linux_work/linux_stable$ lscpu
Architecture:             x86_64
  CPU op-mode(s):         32-bit, 64-bit
  Address sizes:          48 bits physical, 48 bits virtual
  Byte Order:             Little Endian
CPU(s):                   2
  On-line CPU(s) list:    0,1
Vendor ID:                AuthenticAMD
  Model name:             AMD EPYC 7713 64-Core Processor
    CPU family:           25
    Model:                1
    Thread(s) per core:   1
    Core(s) per socket:   2
    Socket(s):            1
    Stepping:             1
    ...
```

So the server is sporting a AMD EPYC 7713 64-Core Processor with 64-cores/128-threads but I'm on a modest VPS plan so I only have 2 cores and 2 threads to work with. I could use a value higher than 2, and I'm sure there's a sweet spot where it won't consume too many system resources, but for the sake of simplicity I'll stick to 2. Alternatively, you can run `nproc` to determine the number of available processors.

So readjusting with this understanding:

```bash
$ make -j2 all 
...
No rule to make target 'debian/canonical-certs.pem', needed by 'certs/x509_certificate_list'.
```

During the compilation, I encountered an error related to missing keys. Since I'm not building a canonical/ubuntu kernel tree, I applied a workaround:

```bash
scripts/config --disable SYSTEM_TRUSTED_KEYS && scripts/config --disable SYSTEM_REVOCATION_KEYS

make -j2 all
```

This takes a while so I'll catch up on some reading in the meantime, 65m23.927s later...

And it's done compiling! Now I can go ahead and install it:

```bash
su -c "make modules_install install"
```

Followed by a reboot and confirmation that I'm running on the latest stable version:

```bash
$ uname -r 
6.8.9
```

## 5. Writing a Loadable Kernel Module (LKM)

At this point, I've found hello world exercises less and less useful when picking up a new language (e.g. I got a feel for Go by building a simple web API server because it provided me with exposure to the most relevant language features for my purposes...this is something simply printing hello world to stdout cannot provide), but I do think it's useful in this case. The kernel is a large project and it's the sort of thing you want to spend some time reading all the changes that goes into a specific area of interest and get a feel for how work is carried out before jumping into making contributions (unless you have a specific missing feature you need, in which case you run with that). So before I modify existing features in the kernel, inserting a hello world module that logs on insertion and removal will allow me to write some kernel space code and have it interact with the kernel.

```c
#include <linux/init.h>
#include <linux/module.h>
MODULE_LICENSE("GPL");

static int init_mod(void) 
{
        pr_alert("Hello\nModule loaded into kernel\n");
        return 0;
}

static void cleanup_mod(void) 
{
        pr_alert("Module removed from kernel\nGoodbye\n");
}

module_init(init_mod);
module_exit(cleanup_mod);
```

Pretty straightforward:

1. The `init.h` header is imported, which will allow us to call `module_init`    and `module_exit` macros which are module routines that run for LKMs on module insertion via `insmod` and removal `rmmod`

2. The `module.h` header is imported, which is required for all LKMs and in this case has the header `printk.h` and `kern_levels.h` containing useful macros like `pr_alert` which is a convenience function for `printk(KERN_ALERT "")`. See

   ```c
   #define pr_alert(fmt, ...) \
    printk(KERN_ALERT pr_fmt(fmt), ##__VA_ARGS__)
   ```

3. Next, the initialization routine is defined. Here I'm simply invoking the `printk` utility function with a KERN_ALERT log level priority implicitly to log a message. Since we're running in kernel space and not user space functions like printf are not available.

   ​ A note on this, because coming from higher-level languages I was a bit thrown off by the lack of a comma separating what appears to be 2 arguments. So KERN_ALERT is defined as follows:

   ```c
   #define KERN_ALERT KERN_SOH "1" 
   ```

   ​ Internally the preprocessor concats the `KERN_LEVEL` with the string, resulting in something like: `printk("<1>Hello\n...");`

4. Any non-zero return status means something went wrong when trying to load the module. So sunny case scenario, `0` is returned.

5. The cleanup method is self-explanatory, where defining a routine to be run at the removal of the module

6. And lastly we pass in our user-defined module initialization and cleanup functions to the macros

Now I just to define a Makefile for building the module, cleaning build artifacts, and testing:

```makefile
obj-m += felipe_mod.o

all:
        make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
        make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
test:
        @sudo dmesg -C
        @sudo insmod felipe_mod.ko
        @sudo rmmod felipe_mod.ko
        @dmesg
```

Breaking down the first target which should help understand the second as well:

1. `make` is executed within the kernel build environment (/lib/modules/6.8.0-31-generic/build), which provides the necessary tools and scripts for compiling kernel modules.
2. The `M` variable is used to let the kernel build system know the location (/linux_trees/felipe_mod/) of my module(s)
3. By specifying the `modules` target (found in /lib/modules/<kernel-version>/build/Makefile), `make` compiles the kernel module(s) defined in `M`

Giving it a go:

```bash
$ make
make -C /lib/modules/6.8.9/build M=/linux_work/felipe_module modules
make[1]: Entering directory '/linux_trees/stable'
  CC [M]  /linux_trees/felipe_module/felipe_mod.o
  MODPOST /linux_trees/felipe_module/Module.symvers
  CC [M]  /linux_trees/felipe_module/felipe_mod.mod.o
  LD [M]  /linux_trees/felipe_module/felipe_mod.ko
make[1]: Leaving directory '/linux_trees/stable'
$ ls
felipe_mod.c   felipe_mod.mod    felipe_mod.mod.o  Makefile       Module.symvers
felipe_mod.ko  felipe_mod.mod.c  felipe_mod.o      modules.order
```

Cool, it compiled without a hiccup!

To confirm the module is working:

```bash
$ sudo insmod ./felipe_mod.ko
$ sudo dmesg
[ 2916.371071] Hello
               Module loaded into kernel
$ sudo rmmod felipe_mod
$ sudo dmesg
[ 2916.371071] Hello
               Module loaded into kernel
[ 3103.226512] Module removed from kernel
               Goodbye
```

Via our test target:

```bash
$ make test
[ 4066.316722] Hello
               Module loaded into kernel
[ 4066.332877] Module removed from kernel
               Goodbye
```

Let's break down the test make target:

1. First I clear the kernel ring buffer - a region of memory within the kernel that serves as temporary storage for log messages and debugging info generated by the kernel - to allow me to see output uncluttered by any other logs.
2. Then I insert and remove the module.
3. Lastly call dmesg to verify the init and cleanup routines ran successfully and logged their corresponding message.

## What's next

So I was able to do a test run of familiarizing myself with the environment, tooling, and basics of LKMs. There's a lot more ground to cover, but knowing I can pull, build, and install upstream kernels and modify existing modules is a good start. The next thing I plan to tackle is interacting with hardware interfaces, specifically USB (mouse or keyboard), and dig into + modify how the kernel does this.
