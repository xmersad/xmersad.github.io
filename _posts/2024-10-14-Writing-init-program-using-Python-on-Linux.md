---
title: "Writing init program using Python on Linux"
date: 2024-10-14
categories: [Embedded-Linux]
tags: [init]
---



As you may know, the first process in Linux is the **init** (PID 1) — it runs while the system is up. Init is normally a compiled program (statically or dynamically linked). Can we write init in **Python**? **Yes.** Two simple ways, with focus on cross-compiling Python and writing the init script.

**1) PyInstaller (quick prototype)**  
Bundle the Python script into a single executable with PyInstaller and place it as `/sbin/init`. Fast to try, but the result is large and opaque — not recommended for production.

**2) Cross-compile / install Python on the target (main focus)**  
1. Prepare a cross-toolchain and sysroot for the target architecture (e.g. `arm64`).  
2. Configure the Python build for the target (`./configure` with proper `--host`/`--build`/`--prefix`). Decide static vs dynamic:
   - **Static**: bigger but fewer runtime deps on target.  
   - **Dynamic**: smaller, but you must ship matching shared libraries (`libc`, `libssl`, `libpython`, ...).  
3. `make` and `make install` into a staged rootfs so `/usr/bin/python3` and stdlib are present.  
4. Copy required shared `.so` files into the rootfs if Python is dynamic and fix `ld.so` paths as needed.  
5. Write your init script with a shebang (e.g. `#!/usr/bin/python3`), put it at `/sbin/init` inside the rootfs and `chmod +x`.  
6. Test the image under QEMU and on hardware; capture early logs on serial/tty.

**Tip:** Ensure `/dev` exists — the kernel usually mounts `devtmpfs` on `/dev`. If `devtmpfs` is disabled or you use an initramfs, create `/dev/urandom` (device node) in the rootfs before starting Python; otherwise Python may block or fail. Also keep a small fallback init (e.g., BusyBox) for recovery if Python cannot start.


