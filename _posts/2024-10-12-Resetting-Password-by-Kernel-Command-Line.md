---
layout: post
title: "Resetting Password by Changing the Kernel Command Line"
date: 2025-10-09
categories: [recovery, kernel, security]
tags: [password-reset, kernel-cmdline, grub, embedded, init]
excerpt: "A focused, practical note on temporarily booting a shell as PID 1 via the kernel command line to reset a local account password, with essential caveats."
---

# Resetting Password by Changing the Kernel Command Line

**Short summary**  
One quick recovery method is to tell the kernel to run a shell as `init` (for example `init=/bin/bash`). After boot you get a root shell early in the process and can run utilities such as `passwd` to reset local account passwords. The method is temporary and comes with important side effects.

---

## Problem statement
You need to reset a local login password but cannot log in. Normal recovery options (rescue image, existing single-user mode) are not available or accessible. You want a minimal, local technique to obtain a root shell and change the password.

---

## Solution (high-level, focused)
1. Edit the kernel command line at boot to append a shell as `init` (for example `init=/bin/bash` or `init=/bin/sh`).  
   - On x86-64 systems with GRUB: interrupt GRUB, press `e` to edit the entry, append the argument to the kernel `linux` line, and boot the edited entry.  
   - On embedded systems: modify the bootloader environment or the file that supplies `bootargs` (for example U-Boot `bootargs`) to include `init=/bin/sh` for a one-shot boot or change the file used by your flash/boot flow.  
2. Boot. The kernel will start the specified shell as PID 1 and drop you into that shell with root privileges.  
3. Prepare the environment:
   - Remount root read-write if needed (`mount -o remount,rw /`).  
   - Mount any separate filesystems required for `passwd` or its libraries (for example `/usr`, `/var`).  
4. Reset the password with available tools (`passwd username`) or, if `passwd` is unavailable, update `/etc/shadow` carefully (prepare a hash elsewhere and paste it).  
5. Sync (`sync`) and reboot or attempt to exec normal init. Be aware `exec /sbin/init` may fail if services and mounts are missing; otherwise use the normal reboot method available on the system.

---

## Key caveats (must-read)
- **Kernel panic if the shell exits:** Because the shell is PID 1, closing or killing it usually causes kernel panic. Do not exit the shell unintentionally. Use `exec /sbin/init` or a controlled reboot when ready.  
- **Normal init and services do not run:** You bypass the distribution init; services, daemons, and automated mounts defined by init will not start. Expect limited system behavior.  
- **Missing mounts (`/proc`, `/sys`, `/run`, etc.):** Many mounts are normally created by init. You may need to mount `/proc` and `/sys` manually if required by tools.  
- **Encrypted or LVM roots:** If the root or `/usr` is encrypted or on LVM, you must unlock and mount underlying volumes first — this trick does not bypass disk encryption.  
- **Use rescue modes when possible:** Prefer distro-provided rescue/single-user modes (`rescue`, `emergency`, `rd.break`, `systemd.unit=rescue.target`) or booting a live/rescue image, which provide a safer recovery environment.

---

## Minimal checklist before attempting
- Have direct console or serial access.  
- Know partition layout and whether `/usr` or other components are on separate devices.  
- Be ready to remount and mount needed filesystems.  
- Prepare a recovery plan if the system does not return to normal (rescue media, physical access).

---

<!-- Image placeholder: replace the path below with your image path if needed. This image must be the last item in the post. -->
![recovery diagram](/path/to/your/image.png)
---
layout: post
title: "Resetting Password by Changing the Kernel Command Line"
date: 2025-10-09
categories: [recovery, kernel, security]
tags: [password-reset, kernel-cmdline, grub, embedded, init]
excerpt: "A focused, practical note on temporarily booting a shell as PID 1 via the kernel command line to reset a local account password, with essential caveats."
---

# Resetting Password by Changing the Kernel Command Line

**Short summary**  
One quick recovery method is to tell the kernel to run a shell as `init` (for example `init=/bin/bash`). After boot you get a root shell early in the process and can run utilities such as `passwd` to reset local account passwords. The method is temporary and comes with important side effects.

---

## Problem statement
You need to reset a local login password but cannot log in. Normal recovery options (rescue image, existing single-user mode) are not available or accessible. You want a minimal, local technique to obtain a root shell and change the password.

---

## Solution (high-level, focused)
1. Edit the kernel command line at boot to append a shell as `init` (for example `init=/bin/bash` or `init=/bin/sh`).  
   - On x86-64 systems with GRUB: interrupt GRUB, press `e` to edit the entry, append the argument to the kernel `linux` line, and boot the edited entry.  
   - On embedded systems: modify the bootloader environment or the file that supplies `bootargs` (for example U-Boot `bootargs`) to include `init=/bin/sh` for a one-shot boot or change the file used by your flash/boot flow.  
2. Boot. The kernel will start the specified shell as PID 1 and drop you into that shell with root privileges.  
3. Prepare the environment:
   - Remount root read-write if needed (`mount -o remount,rw /`).  
   - Mount any separate filesystems required for `passwd` or its libraries (for example `/usr`, `/var`).  
4. Reset the password with available tools (`passwd username`) or, if `passwd` is unavailable, update `/etc/shadow` carefully (prepare a hash elsewhere and paste it).  
5. Sync (`sync`) and reboot or attempt to exec normal init. Be aware `exec /sbin/init` may fail if services and mounts are missing; otherwise use the normal reboot method available on the system.

---

## Key caveats (must-read)
- **Kernel panic if the shell exits:** Because the shell is PID 1, closing or killing it usually causes kernel panic. Do not exit the shell unintentionally. Use `exec /sbin/init` or a controlled reboot when ready.  
- **Normal init and services do not run:** You bypass the distribution init; services, daemons, and automated mounts defined by init will not start. Expect limited system behavior.  
- **Missing mounts (`/proc`, `/sys`, `/run`, etc.):** Many mounts are normally created by init. You may need to mount `/proc` and `/sys` manually if required by tools.  
- **Encrypted or LVM roots:** If the root or `/usr` is encrypted or on LVM, you must unlock and mount underlying volumes first — this trick does not bypass disk encryption.  
- **Use rescue modes when possible:** Prefer distro-provided rescue/single-user modes (`rescue`, `emergency`, `rd.break`, `systemd.unit=rescue.target`) or booting a live/rescue image, which provide a safer recovery environment.

---

## Minimal checklist before attempting
- Have direct console or serial access.  
- Know partition layout and whether `/usr` or other components are on separate devices.  
- Be ready to remount and mount needed filesystems.  
- Prepare a recovery plan if the system does not return to normal (rescue media, physical access).

---

<!-- Image placeholder: replace the path below with your image path if needed. This image must be the last item in the post. -->
![recovery diagram](/path/to/your/image.png)

