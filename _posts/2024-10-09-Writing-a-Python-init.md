---
layout: post
title: "Writing a Python init (PID 1)"
date: 2025-10-09
categories: [embedded, system, init]
tags: [init, python, pid1, rootfs, busybox]
excerpt: "A practical, structured guide to implementing PID 1 (init) in Python: trade-offs, deployment steps for embedded/rootfs targets, and reliability considerations."
---

# Writing a Python `init` (PID 1)

**Short summary**  
This post explains whether and how you can implement the system `init` (PID 1) in Python, practical approaches, deployment steps for a target `rootfs`, and key caveats for robustness and reliability. The content is organized for use as a Chirpy/Jekyll post.

---

## 1. Purpose and constraints (concise)
1. `init` runs as PID 1 and has special responsibilities: adopt orphaned processes, reap children, handle signals, coordinate shutdown/reboot, and ensure essential mounts (`/proc`, `/sys`, `/dev`).  
2. PID 1 behavior differs from ordinary processes: incorrect behavior or exiting unexpectedly can destabilize the system.  
3. Typical `init` implementations are in C for minimal size and deterministic startup. Writing `init` in Python is possible but introduces trade-offs (image size, startup time, dependency surface, and early-boot blocking issues).

---

## 2. Viable approaches (high-level)
1. **Use a native Python interpreter as `/sbin/init`** — place a Python binary and the script in the rootfs and point kernel `init=` to it.  
   - Pros: rapid development, rich libraries, easy iteration.  
   - Cons: larger rootfs, more dependencies, possible early-boot blocking.
2. **Bundle Python into your distribution/build (e.g., include Python in rootfs or integrate with BusyBox builds)**  
   - Pros: more controlled build and packaging.  
   - Cons: more complex build system and maintenance.
3. **Produce a single executable (PyInstaller/Cython)**  
   - Pros: single-file distribution.  
   - Cons: often large and brittle cross-target; not generally recommended for small embedded targets.

---

## 3. Key prerequisites before deploying Python PID 1
1. Provide a working Python interpreter in the target rootfs (statically built or dynamically linked with correct shared libraries).  
2. Ensure the rootfs contains directories the kernel or init will mount: `/dev`, `/proc`, `/sys`, `/run` (create them in the rootfs tree before packing).  
3. Avoid early calls that depend on kernel entropy if the kernel RNG is not yet initialised (see “entropy” note).  
4. Make your init script executable and place it as `/sbin/init` (or pass `init=/path/to/init` on the kernel command line).

---

## 4. Filesystems, `/dev`, and entropy (practical notes)
- **Create `/dev` in the rootfs**: the kernel will mount `devtmpfs` on `/dev` if the directory exists; without it, device nodes will be missing and many userland components (including Python) may fail.  
- Also create `/proc` and `/sys` so they can be mounted by the kernel or by init. Typical rootfs setup command (performed on the build host) is: create directories `dev`, `proc`, `sys`, `run` with correct ownerships.  
- **Entropy / `crng` issue**: some Python operations (or libraries) may call system randomness and block until the kernel's cryptographic RNG is initialised (`crng init done`). To avoid blocking early in boot:
  - defer cryptographic or randomness-heavy operations until after boot;
  - use `/dev/urandom` for non-cryptographic needs;
  - ensure your init avoids calls that rely on blocking `getrandom()`.

---

## 5. Design principles for a Python-based init (what an implementer must plan)
1. **Reap children reliably**: PID 1 must handle `SIGCHLD` and call `waitpid()` to prevent zombies.  
2. **Forward signals**: receive termination signals and forward them to supervised child process groups.  
3. **Minimal early-boot surface**: perform the smallest, most deterministic steps in the earliest phase (mounts, basic device availability) and delay complex operations. Consider a tiny C wrapper for the first actions if you need extreme minimalism.  
4. **Supervision strategy**: decide whether Python acts only as a launcher that `exec`s a main process, a simple supervisor that restarts services, or a full init system. Each choice changes complexity and reliability requirements.  
5. **Logging & observability**: ensure kernel console/serial output or persistent logging is available to debug boot-time failures.

---

## 6. Recommended deployment flow (step-by-step)
1. **Build/obtain a Python interpreter for the target**  
   - cross-compile or build on the target; prefer dynamic linking appropriate to your libc (musl vs glibc).  
2. **Prepare the rootfs tree**  
   - create required directories: `/bin`, `/sbin`, `/usr/bin`, and at minimum `dev`, `proc`, `sys`, `run`, `etc`.  
   - copy Python interpreter and required shared libraries into the rootfs; use `ldd`/`scanelf` on the target or toolchain helpers to collect dependencies.  
3. **Place the init script**  
   - add your Python init script to `rootfs/sbin/init` and ensure it is executable. Optionally add a small shell/C wrapper as `/sbin/init` that performs critical early mounts then `exec`s Python.  
4. **Ensure `/dev` exists in the rootfs**  
   - create `/dev` so the kernel can mount `devtmpfs` and populate device nodes during boot.  
5. **Pack rootfs**  
   - create an initramfs (cpio) or filesystem image from the rootfs. Ensure ownerships and permissions are preserved.  
6. **Boot & test in an emulator**  
   - test first in QEMU or a VM; monitor console output and iterate. Only test on hardware when you have a recovery path.

---

## 7. Reliability, size, and alternatives
- **Size**: Python + standard libs inflate image size; for tight devices consider `MicroPython` or a tiny C init plus helper scripts.  
- **Robustness**: PID 1 must be bulletproof for reaping and shutdown handling. Consider a small C shim for the minimal, critical tasks (mounting, early dev management), then hand control to Python.  
- **Security surface**: embedding a full Python interpreter increases attack surface—minimize included third-party modules, and audit what is shipped.

---

## 8. Best practices checklist (quick)
1. Create `rootfs/{dev,proc,sys,run}` before packing.  
2. Provide a working interpreter and collect shared-library dependencies.  
3. Make `rootfs/sbin/init` executable and confirm `init=` if needed.  
4. Avoid blocking entropy calls during early boot.  
5. Implement `SIGCHLD` handling and child reaping.  
6. Forward termination signals to child process groups.  
7. Test in QEMU/VM with serial console logging.

---

## 9. Recommended minimal file layout example (build host)
- rootfs/
  - bin/
  - sbin/
    - init    ← your Python script (executable)
  - usr/bin/   ← python interpreter and tools
  - lib/       ← shared libraries
  - dev/
  - proc/
  - sys/
  - run/
  - etc/

---

## 10. References & further reading
- Linux process model and PID 1 responsibilities (kernel documentation and classic init/systemd discussions).  
- Python cross-compilation and embedding guides (official Python build docs).  
- BusyBox and minimal init strategies for embedded systems.

---

<!-- Image placeholder: replace the path below with your actual image path. This image must be the last item in the post. -->
<!--![init diagram](/path/to/your/image.png)
-->
