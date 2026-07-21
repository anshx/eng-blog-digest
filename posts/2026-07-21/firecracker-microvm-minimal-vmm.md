---
title: "Firecracker: Lightweight Virtualization for Serverless Applications"
source: https://www.usenix.org/system/files/nsdi20-paper-agache.pdf
author: Alexandru Agache, Marc Brooker, Alexandra Iordache, Anthony Liguori, Rolf Neugebauer, Phil Piwonka, Diana-Maria Popa
company: AWS
date_posted: 2020-02-25
date_digested: 2026-07-21
---

# Firecracker: Lightweight Virtualization for Serverless Applications

## What's new to learn

- **The isolation hierarchy as a design space**: Language sandboxes, containers, microVMs, VMs, and physical hosts form a spectrum trading isolation strength for startup latency and per-unit overhead — most engineers conflate two or three of these levels, but treating it as a continuous spectrum reveals where the sweet spot actually lies.

- **The minimal device model principle**: A VMM that implements fewer virtual devices has a smaller Trusted Computing Base (TCB), fewer vulnerability classes, and faster boot time; Firecracker cuts QEMU's ~1.5M-line C device zoo down to three virtio devices in ~50k lines of Rust, and each absent device is a removed attack vector, not a missing feature.

- **Defense-in-depth via the Jailer**: Wrapping the VMM process itself in a second sandbox (new Linux namespaces + seccomp-bpf syscall filter) means an attacker who escapes the hardware VM boundary still faces a tightly restricted OS process, turning what would be a full host compromise into a contained foothold.

## Prerequisites

- **KVM (Kernel-based Virtual Machine)**: The Linux kernel module that lets a process use x86 hardware virtualization extensions (Intel VT-x / AMD-V). KVM exposes a `/dev/kvm` fd; a VMM opens it and makes ioctls to create and run virtual CPUs. Understanding that KVM handles the "trap privileged instructions and route them to the VMM" mechanism is essential.
- **Virtio**: A paravirtualization standard for I/O. Instead of emulating a real SATA controller or RTL8139 NIC, virtio exposes a simple shared-ring interface the guest driver writes descriptors to and the host processes. Guest VMs must have virtio drivers (modern Linux kernels ship them).
- **seccomp-bpf**: A Linux syscall filter that applies a BPF program to every syscall before it executes; the program returns ALLOW, KILL, or TRAP. Whitelisting syscalls reduces the kernel attack surface from ~350 calls to whichever dozen the process actually needs.
- **Namespaces**: Linux's mechanism for isolating kernel resources (network stacks, PID trees, filesystem views, user IDs) so that a process inside a namespace sees a fresh, private copy of that resource.

## The core idea

Lambda and Fargate need to run untrusted customer code at massive scale — thousands of functions per host — with the isolation guarantees of a VM but the startup latency and memory overhead of a container. Neither extreme is right:

- **Containers alone** (namespaces + cgroups) boot in milliseconds and use near-zero extra memory, but share the host kernel. A kernel exploit from inside the container breaks out completely. For multi-tenant workloads running arbitrary customer code, this is not an acceptable risk surface.
- **Full VMs** (QEMU + KVM) provide strong hardware isolation, but QEMU emulates dozens of legacy devices and carries ~1.5M lines of C code. It boots in seconds, uses tens of MB per instance, and has a long history of CVEs traceable to device emulation code (e.g., VENOM, CVE-2015-3456, was a floppy controller escape).

Firecracker's answer: build a purpose-built VMM that uses real hardware virtualization (KVM) — so the isolation boundary is enforced in silicon — but replaces QEMU's device zoo with the bare minimum required to boot a production Linux workload. The result boots in under 125 ms and uses ~5 MB overhead per microVM.

The headline insight: **removing a device from the VMM is not a missing feature, it is a removed attack vector**. Every line of device-emulation code that doesn't exist cannot be exploited.

## Mechanics

### The isolation stack

Firecracker adds two isolation layers, applied in order:

```
Customer code
  └─ Guest Linux kernel    ← isolation layer 1: KVM hardware virtualization
       └─ Firecracker VMM  ← isolation layer 2: Jailer (namespaces + seccomp-bpf)
            └─ Host Linux
```

**Layer 1 — microVM boundary**: The guest vCPU runs in VMX non-root mode (Intel) or SVM guest mode (AMD). Any attempt to execute a privileged instruction or access I/O traps into the VMM via KVM's file descriptor interface. The hardware enforces this separation at the CPU level; no kernel code path in the guest can escape it without a hypervisor vulnerability.

**Layer 2 — Jailer process sandbox**: Before exec()-ing the Firecracker binary, a small "jailer" binary applies:
- New `netns`, `pidns`, `mntns`, `userns` — each Firecracker process has its own private namespace
- A chroot into a minimally-populated directory (a few device nodes)
- A seccomp-bpf filter that whitelists ~20 syscalls (the ones Firecracker actually uses)
- cgroup limits for CPU and memory

If someone exploits a Firecracker vulnerability and escapes the KVM hardware boundary, they land in this jailer-sandboxed process and are confined to the whitelisted syscalls and private namespace. The host's network stack, PID namespace, and filesystem are not visible.

### The device model

Firecracker implements exactly the devices needed to run a modern Linux workload:

| Device | Purpose |
|---|---|
| `virtio-net` | Network interface (up to 3 per microVM) |
| `virtio-block` | Block storage |
| `virtio-vsock` | Host-guest communication channel |
| 8250/16550 UART | Serial console for debugging |
| i8042 keyboard controller | Only enough to generate the `reboot` signal |

Deliberately absent: PCI bus, USB host controller, IDE/SATA controller, floppy, VGA/GPU, BIOS/UEFI firmware, ACPI tables (minimal subset only), ISA bus, PS/2 mouse, CMOS/RTC.

**No firmware**: Firecracker loads the Linux kernel binary directly using the x86 multiboot / ARM64 boot protocol, bypassing the BIOS/UEFI firmware execution stage entirely. Firmware has its own history of CVEs and adds 100+ ms to boot time. Removing it takes both away.

**No PCI bus**: PCI enumeration is a significant source of boot delay and a historically vulnerable subsystem. Virtio devices are exposed via MMIO (memory-mapped I/O) instead, which requires no bus enumeration.

### Boot sequence

```
[0 ms]   Firecracker process starts (already Jailer-wrapped)
[5 ms]   REST API ready; caller configures vCPUs, RAM, block/net devices
[10 ms]  Caller sends StartMicroVM; Firecracker maps kernel+initrd into guest memory
[15 ms]  vCPU thread starts in long mode; Linux kernel begins decompression
[125 ms] Kernel has printed first userspace init log line
```

The REST API over a Unix socket is intentional: no TCP parsing in the VMM, no HTTP library with a large attack surface. Configuration arrives before the VM starts and is sealed at boot.

### Rate limiting and resource control

Firecracker implements a token-bucket rate limiter per virtio-net and virtio-block device (configurable via the REST API). This prevents one microVM from monopolizing I/O bandwidth on a shared host — the same noisy-neighbor problem that causes tail latency spikes in multi-tenant databases. The token bucket refills at a configured rate; bursts consume extra tokens up to a cap.

### Memory balloning

A `virtio-balloon` device (optional) lets the host reclaim memory from a guest that isn't using all of its configured RAM. The guest balloon driver "inflates" — allocates pages and notifies the host — and the host returns them to the allocator. At Lambda scale, functions often allocate less than their configured maximum; ballooning recovers that slack without requiring complex memory overcommit bookkeeping.

### The production Lambda model

Each Lambda "worker" host runs dozens to thousands of concurrent microVMs. AWS groups microVMs into "cells" — collections of microVMs managed together as a unit. When a Lambda function is invoked:

1. If a pre-warmed slot exists (a paused microVM with the runtime already initialized), it resumes in milliseconds.
2. If not, Firecracker boots a new microVM (~125 ms VMM boot, plus kernel boot, plus runtime init = total cold start 200–400 ms).

Snapshots (introduced after the original paper) allow Firecracker to serialize the entire microVM memory state to disk and restore it later, enabling "clone on demand" patterns where thousands of identical runtime environments can be forked from one pre-warmed snapshot.

## Where it breaks

**Linux-only guests**: Firecracker has no BIOS or UEFI, no PCI bus, and no legacy device support. Windows, FreeBSD, or any OS that requires these cannot run inside a Firecracker microVM without significant porting work.

**No live migration**: Traditional VM migration (vMotion, QEMU migration) requires a complete device emulation stack that can serialize its state and reconstruct it on another host. Firecracker's minimal device model makes snapshot/restore easy but cross-host migration (which requires serializing in-flight I/O) is not supported.

**No hardware passthrough (SR-IOV, VFIO)**: GPU workloads cannot access hardware directly. This makes Firecracker unsuitable for ML training or inference workloads that need low-overhead GPU access. AWS uses different isolation mechanisms (Nitro Cards + hardware security) for GPU instances.

**Cold-start floor**: The 125 ms boot floor is for the microVM kernel. Applications add more. Snapshot restore is faster, but requires the snapshot to already exist. For truly latency-sensitive paths, even container restart (~10 ms for a process fork) is faster.

**seccomp-bpf filter maintenance**: The Jailer's syscall whitelist must be kept in sync with any Firecracker feature addition. Adding a new capability to Firecracker may require updating the seccomp filter — failing to do so silently breaks the new feature; incorrectly adding a syscall quietly expands the attack surface.

## Why it works

The paper's insight is a precise application of the **principle of least privilege** to the hypervisor layer, simultaneously attacking three independent factors that contribute to vulnerability probability:

**Attack surface = TCB size × interface width × code vulnerability density**

- *TCB size*: Firecracker is ~50k lines of Rust vs. QEMU's ~1.5M lines of C. Fewer lines = fewer places bugs can hide.
- *Interface width*: 3 active device types vs. QEMU's 200+. Each QEMU device is a separate parseable input stream that can contain malformed data. Firecracker's virtio interfaces have a well-specified, minimal format.
- *Code vulnerability density*: Rust's ownership/borrow system eliminates use-after-free, buffer overflow, and data race bugs — the three vulnerability classes that historically dominate VMM CVE lists.

The deepest connection: **Firecracker is a microkernel applied to the hypervisor layer**. In microkernel design, the principle is to put only the minimum in ring 0 (address-space management, IPC, interrupt dispatch) and move everything else — device drivers, filesystem servers, network stacks — into user-space processes. Firecracker applies the same logic one level down: put only the minimum in the KVM trap handler (the VMM), and exclude everything else. The jailer then extends this by putting the VMM itself inside a further restricted user-space process.

This is the same pattern already seen in the archive:
- **eBPF verifier** (2026-07-08): restrict the kernel interface to a provably-safe subset. Firecracker restricts the VMM interface to a provably-minimal subset.
- **seccomp-bpf** (in the eBPF post): whitelist syscalls = "Firecracker's jailer" applied to any process.
- **io_uring** (2026-06-01): replace many per-operation syscalls with a shared-memory interface — fewer syscall crosses = smaller interface.

The principle generalizes: wherever you draw a trust boundary, the attack surface on that boundary equals the number of things that can cross it times the validation cost per crossing. Every device removed from a VMM, every syscall removed from a whitelist, every opcode removed from a bytecode interpreter is not a missing feature — it is a constraint that makes the boundary cheaper to validate and harder to subvert.

## Going deeper

1. **The NSDI 2020 paper itself** — the primary source, with production performance numbers and the full architecture description: https://www.usenix.org/system/files/nsdi20-paper-agache.pdf

2. **Marc Brooker's "Seven Years of Firecracker" retrospective (2025)** — covers how Firecracker evolved after the paper: snapshot/restore, ARM64 support, AWS Bedrock using microVMs for agent isolation, and lessons learned at Lambda scale: https://brooker.co.za/blog/2025/09/18/firecracker.html

3. **gVisor design doc (Google, 2019)** — the complementary design point: instead of a minimal VMM, run a user-space Go reimplementation of the Linux kernel that intercepts syscalls via ptrace or KVM. Comparing gVisor's "reimplement the kernel safely" approach with Firecracker's "run the real kernel but behind hardware isolation" approach teaches the full space of hypervisor/sandbox design tradeoffs: https://gvisor.dev/docs/architecture_guide/
