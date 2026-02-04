---
title: "Passing Multiple Root Filesystems to the Linux Kernel with root_pr"
date: 2026-02-05
categories: [Kernel]
tags: [VFS, rootfs]
---

## Introduction

During Embedded Linux development, the root filesystem is often unstable or under active development.
Mount failures may lead to kernel panic and make the system unbootable.
In other scenarios, multiple root filesystems may exist (eMMC, SD card, NAND, NFS), and the system should try them in priority order.

For example, NAND flash may wear out over time and fail to mount the root filesystem.
Without deploying a new firmware update, it may be desirable to automatically fall back to an SD card or a network-based root filesystem.

To address this problem, a new kernel command-line parameter called `root_pr` (short for root priorities) is introduced.

---

## The Idea

The `root_pr` parameter allows passing multiple root filesystem locations to the kernel in priority order:

```text
root_pr="/dev/nfs;/dev/mmcblk1p2;/dev/sda1"
```

At boot time, the kernel parses this list and attempts to mount each root filesystem in order.
If mounting fails, the kernel moves to the next entry.
Once a root filesystem is successfully mounted, the boot process continues.

---

## Supported Root Types

The `root_pr` parameter supports the following root filesystem sources:

- Block devices: `/dev/mmcblk1p2`, `/dev/sda1`
- PARTUUID: `PARTUUID=12345678-1234-1234-1234-123456789abc`
- MTD devices: `mtd0`, `mtd1`
- UBI volumes: `ubi0:rootfs`, `ubi0_0`
- Network filesystems: `/dev/nfs`, `/dev/cifs`

---

## Kernel Integration Overview

The feature is integrated into the Linux kernel root mounting flow.
A new early boot parameter is registered and processed before the standard `root=` handling.
If `root_pr` is present, the kernel tries each root filesystem entry in order.
If all entries fail, the kernel falls back to the default root mounting logic.


---

## Example Kernel Command Line

```text
linux /vmlinuz root_pr="/dev/nfs;/dev/mmcblk1p2" rootfstype=ext4 ro
```

---

## Example Boot Logs

```text
VFS: Trying root filesystems from priority list: /dev/nfs;/dev/mmcblk1p2
VFS: Attempting to mount root filesystem #1: /dev/nfs
VFS: Failed to mount root filesystem: /dev/nfs
VFS: Attempting to mount root filesystem #2: /dev/mmcblk1p2
VFS: Successfully mounted root filesystem #2: /dev/mmcblk1p2
```

---

### NFS Failure and Automatic Fallback

![NFS failure - fallback to next rootfs](/assets/img/root_pr-nfs-fail.png)

---

## Why Not initramfs or Bootloader?

This fallback mechanism can also be implemented in initramfs or the bootloader layer.
However, integrating the logic directly into the kernel root mounting stage provides:

- A uniform fallback behavior across all boot methods
- No dependency on userspace recovery scripts
- Reduced boot complexity
- A deterministic and transparent root filesystem selection mechanism

---

## Patch and Documentation

Enablement package (patch and documentation):

```text
https://mega.nz/file/BrNxDCCS#kU7jfym0ITwEYks-ISY3Ld8vClgJbUa7mva9Oe3-6cI
```

---


## Conclusion

The `root_pr` feature provides a robust way to improve boot reliability by allowing the kernel to try multiple root filesystem sources in priority order.
This reduces kernel panic incidents during development and enables resilient fallback strategies for production systems.
