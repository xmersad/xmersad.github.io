---
title: "Resetting Login Password by Changing Kernel Command Line"
date: 2025-10-27
categories: [Kernel Command Line]
tags: [init]
---


## Summary
One easy way to reset a local login password is to set a shell (e.g. Bash) as the system **init** via the kernel command line. After boot, the kernel will start that shell instead of the normal init system, letting you run programs such as `passwd` to change user passwords.

## What happens
- The kernel launches your chosen shell as PID 1, giving you an interactive prompt before normal services start.  
- You can run utilities (for example `passwd`) to reset credentials from that environment.

## Important caveats
- If you **exit** or close that shell, the real init is gone and the kernel will usually panic.  
- The normal boot flow is bypassed: system services will not start, and mounts or setups normally done by init (including some `tmpfs` mounts under `/proc` or `/sys`) may be missing.  
- This is a recovery technique — it alters the system runtime and is **not** a replacement for normal boot procedures.

## Where to change the kernel command line
- On x86-64 desktops/servers you typically edit the boot entry in **GRUB** at boot time (or update its config).  
- On embedded systems, modify the bootloader configuration or the file where the kernel command line is defined for that platform.

## Safety & authorization
Only use this method on machines you own or are explicitly authorized to administer. For lost-password recovery on production systems, prefer safer procedures such as booting a trusted rescue image, using vendor-provided recovery tools, or following your organization's approved recovery workflow.

