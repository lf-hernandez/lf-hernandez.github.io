---
title: "Linux Kernel Development: Exploring USB device drivers"
date: 2024-05-20T21:40:26-04:00
summary: "In this follow-up post, I cover my experience experimenting with USB device drivers."
tags: ["linux", "kernel", "c"]
categories: ["linux kernel"]
---
## Introduction

In this post, I explore some of the basics of USB device drivers by iterating over last post's LKM. There's quite a bit of low-level detail that goes into this, so I would recommend heading over to a resource like the Linux Device Drivers (LDD) book as I am still in the very early stages of kernel development. Although dated, it goes into much more detail than what I will cover here.

## The Linux Device Model (LDM)

We can define the Linux Device Model as a unified abstraction that represents hardware devices in the kernel and allows for efficient device interaction and management. Going off of some historical context provided in LDD3:

> One of the stated goals for the 2.5 development cycle was the creation of a unified
device model for the kernel. Previous kernels had no single data structure to which
they could turn to obtain information about how the system is put together. Despite
this lack of information, things worked well for some time. The demands of newer
systems, with their more complicated topologies and need to support features such
as power management, made it clear, however, that a general abstraction describing
the structure of the system was needed.

The key takeaway, and I'm sure my understanding of this will evolve as I continue to dig deeper into the kernel, is that the device model provides a consistent interface for device drivers and user space applications. While abstractions simplify the complexities of managing implementation details, it does not mean that the device model itself is simple. Luckily, driver authors do not need to directly deal with the low-level details in order to write device drivers.

## Writing a custom USB device driver for my Lexar USB flash drive

This exercise aims to be my first run with device drivers. A large portion of kernel development is dedicated to this and it seemed like a reasonable next step to research after writing my first module.

### Algorithm

1. Define a table of devices supported by the driver. In my case, just one, providing the manufacturer and the device model IDs (device descriptors), and initialize the structure via the USB_DEVICE macro (L983 linux/usb.h). Notice the terminating entry `{}`, this is used by USB core to determine the end of the table.
2. Export the device table via the MODULE_DEVICE_TABLE macro (L243 linux/module.h), this essentially informs the kernel which module to load when a matching device is connected. This is also exposed in user space.
3. Define the probe function. This is triggered when a matching device from the table is connected. In our case, we simply log vendor and product IDs.
4. Define the disconnect function. This is triggered, you guessed it, when the matching device from the table is disconnected. We simply log a message acknowledging this.
5. Define the USB driver structure (L1212 linux/usb.h) and designated initializers for the relevant members. This essentially links our device table with the corresponding probe and disconnect functions as well as assigning a name to it.
6. Define module initialization and clean-up functions. These are called in their respective lifecycle phase.
   1. Attempt to register the USB device driver on load
   2. Deregister it on exit.

Let's go over the implementation:

#### Device table

```c
static struct usb_device_id usb_table[] = {
 { USB_DEVICE(0x21c4, 0x0809) },
 {}
};
MODULE_DEVICE_TABLE(usb, usb_table);
```

#### Device driver

```c
static int usb_probe(struct usb_interface *interface, const struct usb_device_id *id)
{
 pr_alert("FELIPE MODULE: USB flash drive plugged in (VID: %04X, PID: %04X)\n", id->idVendor, id->idProduct);
 return 0;
}

static void usb_disconnect(struct usb_interface *interface)
{
 pr_alert("FELIPE MODULE: USB flash drive removed\n");
}

static struct usb_driver custom_lexar_driver = {
 .name = "felipe_usb_flash_driver",
 .id_table = usb_table,
 .probe = usb_probe,
 .disconnect = usb_disconnect,
};
```

#### Module initialization and clean up

```c
static int __init mod_init(void)
{
 int result;
 result = usb_register(&custom_lexar_driver);
 if (result) {
  pr_alert("FELIPE MODULE: usb_register failed. Error: %d\n", result);
 } else {
  pr_alert("FELIPE MODULE: USB driver registered succesfully\n");
 }
 return result;
}

static void __exit mod_exit(void)
{
 usb_deregister(&custom_lexar_driver);
 pr_info("FELIPE MODULE: USB driver unregistered\n");
}

module_init(init_mod);
module_exit(cleanup_mod);
```

Compile and load the module:

```bash
make
sudo insmod felipe.ko
```

Test by plugging and unplugging the flash drive and check the kernel logs:

```bash
$ sudo dmesg
[   49.758016] felipe: loading out-of-tree module taints kernel.
[   49.758026] felipe: module verification failed: signature and/or required key missing - tainting kernel
[   49.758665] usbcore: registered new interface driver felipe_usb_flash_driver
[   49.758668] FELIPE MODULE: USB driver registered successfully
[   59.585539] usb 8-1: new SuperSpeed USB device number 2 using xhci_hcd
[   59.600320] usb 8-1: Product: USB Flash Drive
[   59.600322] usb 8-1: Manufacturer: Lexar
[   59.600324] usb 8-1: SerialNumber: ***
[   59.601392] FELIPE MODULE: USB flash drive plugged in (VID: 21C4, PID: 0809)
[   59.618910] usbcore: registered new interface driver usb-storage
[   59.620740] usbcore: registered new interface driver uas
[   70.217931] usb 8-1: USB disconnect, device number 2
[   70.218079] FELIPE MODULE: USB flash drive removed
[   94.651087] usbcore: deregistering interface driver felipe_usb_flash_driver
[   94.651138] FELIPE MODULE: USB driver unregistered
```

A few things to note:

In order to grab the device IDs, I simply ran `lsusb` with the flash drive plugged in, which yielded:

```bash
$ lsusb
...
                   # vendor:product
Bus 008 Device 007: ID 21c4:0809 Lexar USB Flash Drive
```

Assuming we wanted to automatically load our module anytime the device was plugged in and not just when we manually load our module, we'd have to add a custom udev rule and reload udev.

```bash
# e.g /etc/udev/rules.d/99-usb-flash.rules
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="21c4", ATTR{idProduct}=="0809", RUN+="/sbin/modprobe felipe_usb_flash_driver"
---
$ udevadm control --reload-rules
$ udevadm trigger
```

udev (`userspace /dev`) is a dynamic device manager. It takes care of managing device-related events in user space such as when hotplug devices are added to or removed from the system. In our case, we're choosing which kernel module to load when this specific USB device is detected.

## Conclusion

This was a simple yet challenging exercise for me. My main hurdles were wrapping my head around the device model and how all the pieces fit together (e.g. the usb host controller and usb core hand-off, kobjects, sysfs, and much, much, more). The kernel provides driver and module APIs that are intuitive enough to get started and allow for quick experimentation. A big help in understanding this was being able to dig into the headers as well as read through various device driver sources. Which is great because it provides a perfect the feedback loop for a programmer to understand how something works. This was a necessary first step for me to have a better working understanding and dive back into learning from a better point of reference.
