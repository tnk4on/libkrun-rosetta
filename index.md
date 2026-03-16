---
layout: default
title: "Rosetta 2 Support for libkrun: Technical Feasibility and Legal Analysis"
description: >-
  Technical investigation into adding Rosetta 2 (x86_64 binary translation)
  support to libkrun, a lightweight VMM library used by Podman on macOS.
---

# Rosetta 2 Support for libkrun: Technical Feasibility and Legal Analysis

## Abstract

This document presents my investigation into adding Rosetta 2 (x86_64 binary translation) support to [libkrun](https://github.com/containers/libkrun), a lightweight VMM library used by [Podman](https://github.com/containers/podman) on macOS. I implemented a working prototype, analyzed Apple's protection mechanisms, and assessed the legal implications of the approach. My conclusion is that **Rosetta 2 cannot be legally integrated into libkrun** due to fundamental architectural constraints and intellectual property concerns. I also survey alternative approaches and their trade-offs.

**Author**: Shion Tanaka ([@tnk4on](https://github.com/tnk4on))  
**AI-Assisted-By**: Anthropic Claude Opus 4.6 (via Antigravity coding agent)  
**Date**: March 2026  
**Status**: Research Report  
**License**: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)

---


## 1. Background

### 1.1 The Problem

On Apple Silicon Macs, Podman runs Linux containers inside a virtual machine (VM). Podman supports two VMM backends: `applehv` (using `vfkit` / `Virtualization.framework`) and `libkrun` (using `Hypervisor.framework`). The libkrun backend is primarily used for its **GPU acceleration** capability — it provides `virtio-gpu` with Venus support, enabling Vulkan-capable GPU passthrough to containers, which is essential for AI/ML workloads. This GPU support is not available through `Virtualization.framework`.

When users need to run x86_64 container images (e.g., for legacy applications), the VM must provide some form of x86_64 binary translation. Currently, libkrun-based VMs rely on **QEMU user-mode emulation**, which operates at roughly 20–40% of native ARM64 performance.

Apple's **Rosetta 2** offers significantly better x86_64 translation performance (70–80% of native), but is officially tied to the `Virtualization.framework`. libkrun uses the lower-level `Hypervisor.framework`, which does not officially support Rosetta. This creates a dilemma: users who choose the libkrun backend for GPU acceleration must accept slower x86_64 emulation.

### 1.2 Goal

I set out to investigate whether Rosetta 2 support could be added to libkrun/Podman, and whether such changes could be contributed upstream as open-source patches.

### 1.3 Environment

| Component | Version |
|-----------|---------|
| Host OS | macOS (Apple Silicon) |
| VMM | libkrun (Hypervisor.framework) |
| Guest OS | Fedora CoreOS 43 |
| Podman | v5.x / v6.x |

---

## 2. Architecture Comparison

Understanding the two macOS virtualization frameworks is essential to this investigation.

| Aspect | Hypervisor.framework | Virtualization.framework |
|--------|---------------------|--------------------------|
| **Introduced** | macOS 10.10 (Yosemite) | macOS 11 (Big Sur) |
| **Abstraction Level** | Low-level C API | High-level Swift/Obj-C API |
| **Use Cases** | Direct CPU virtualization | VM lifecycle management |
| **Rosetta Support** | ❌ None | ✅ macOS 13+ |
| **Users** | libkrun, QEMU, Xhyve | vfkit, Parallels, UTM |
| **Flexibility** | High (full control) | Limited (opinionated API) |

The key architectural difference: `Virtualization.framework` provides `VZLinuxRosettaDirectoryShare`, an official API that exposes Rosetta to Linux VMs. No equivalent exists for `Hypervisor.framework`, and Apple has shown no indication of providing one.

---

## 3. Implementation Attempt

I implemented a prototype across three repositories to understand the full technical picture.

### 3.1 libkrun Changes

libkrun already contained an existing `krun_set_rosetta()` function that shares the Rosetta directory (`/Library/Apple/usr/libexec/oah/RosettaLinux`) as a virtio-fs device with the tag `rosetta`.

### 3.2 krunkit Changes

Added a `--rosetta` CLI flag and the corresponding FFI call to `krun_set_rosetta()`. Switched to static linking to simplify deployment and code signing.

### 3.3 Podman Changes

- Wired `containers.conf`'s `Rosetta` setting through to krunkit's `--rosetta` flag
- Extended Ignition configuration to create the `/etc/containers/enable-rosetta` marker file
- Enabled `rosetta-activation.service` for VMs using the libkrun backend

### 3.4 Virtualization Identity Fix

The guest's `systemd-detect-virt` returned `none` instead of `apple`, preventing `rosetta-activation.service` from starting. I modified:
- **SMBIOS**: Set `sys_vendor` and `product_name` to Apple-compatible strings
- **FDT (Device Tree)**: Added `apple,vm-apple` to the root `compatible` property

After these changes, `systemd-detect-virt` correctly returned `apple`, and the activation service started successfully. The Rosetta virtio-fs mount and `binfmt_misc` registration completed without errors.

### 3.5 GPU + Rosetta Coexistence

Since libkrun is primarily chosen for its GPU acceleration, maintaining GPU functionality alongside Rosetta was a key requirement. This required solving a **guest physical memory layout conflict**.

**The Problem**: Rosetta requires a reserved region in the guest physical address space (approximately 8 GB – 16 GB) for its internal mappings. Meanwhile, libkrun's `virtio-gpu` implementation attempts to allocate the largest possible contiguous Shared Memory (SHM) region using DAX (Direct Access) for GPU BAR mapping. These two regions overlapped, causing failures when both features were enabled simultaneously.

**The Solution**: I implemented a "Rosetta hole avoidance" mechanism in the shared memory manager (`shm.rs`). When Rosetta is enabled, the GPU SHM allocation logic detects if the requested region would overlap with the 8 GB – 16 GB Rosetta range, and if so, skips past it by relocating the GPU BAR to above the 16 GB boundary. The `hv_vm_map` calls were also adjusted to handle non-contiguous memory regions.

This ensured that GPU acceleration (virtio-gpu with Venus) and Rosetta's virtio-fs mount could coexist within the same VM without memory mapping conflicts. I verified that both GPU workloads and Rosetta activation functioned correctly across various VM RAM configurations (4 GB, 7 GB, 10 GB, 15 GB).

### 3.6 Result

Despite the successful infrastructure setup, **actual x86_64 binary execution failed**:

```
rosetta error: Rosetta is only intended to run on Apple Silicon with a macOS host
using Virtualization.framework with Rosetta mode enabled
```

The Rosetta binary performs runtime verification that goes beyond simple environment checks.

---

## 4. Discovery: Rosetta's Protection Mechanisms

Through my investigation, I identified multiple layers of protection that Rosetta employs to verify it is running in an authorized environment.

### 4.1 Runtime Environment Verification

The Rosetta binary (`/Library/Apple/usr/libexec/oah/RosettaLinux`) validates:
1. The host is Apple Silicon
2. The host runs macOS
3. The VM uses **Virtualization.framework** (not just Hypervisor.framework)
4. Rosetta mode is properly enabled through the framework

### 4.2 Proprietary Verification Protocol

Rosetta communicates with the host through a series of `ioctl` calls on the virtio-fs device. These IOCTLs are not publicly documented by Apple:

| IOCTL | Direction | Description |
|-------|-----------|-------------|
| `0x80456122` | Read | Legacy verification |
| `0x80456125` | Read | Current verification (involves secret exchange) |
| `0x80806123` | Read | Returns 128 bytes |
| `0x00006124` | Void | Returns success |

### 4.3 Secret Exchange

The verification protocol involves a challenge-response mechanism where the Rosetta binary expects specific data to be returned by the host, confirming it is operating within Apple's `Virtualization.framework`. The details of this exchange are proprietary to Apple.

### 4.4 Implications

These protections are **intentional technical measures** by Apple to restrict Rosetta usage to `Virtualization.framework`. Circumventing them requires:
- Reverse engineering proprietary binaries
- Reimplementing undocumented protocols
- Spoofing hardware/software identity

Each of these actions carries significant legal risk, as analyzed in the next section.

---

## 5. Legal Analysis

> **Disclaimer**: This section provides a technical analysis of legal risks. It does not constitute legal advice. Consult qualified legal counsel for definitive guidance.

### 5.1 DMCA Considerations

The U.S. Digital Millennium Copyright Act (DMCA) contains provisions directly relevant to this work:

**§1201(a) — Anti-circumvention**: Prohibits circumventing technological protection measures (TPMs) that control access to copyrighted works. Rosetta's verification protocol constitutes a TPM.

**§1201(b) — Trafficking in circumvention tools**: Prohibits distributing tools or technologies primarily designed to circumvent TPMs. Publishing code that bypasses Rosetta's protections in an open-source repository constitutes distribution.

**§1201(f) — Interoperability exception**: Permits reverse engineering for achieving interoperability between independently created programs. However, this exception is narrowly scoped:
- It applies to the **act** of reverse engineering, not the **distribution** of circumvention tools
- It requires that the information obtained is used **solely** for interoperability purposes
- Its applicability to distributing circumvention code in OSS projects is legally uncertain

### 5.2 macOS Software License Agreement (EULA)

Apple's macOS EULA contains restrictions on:
- **Reverse engineering**: Prohibited except to the extent permitted by applicable law
- **Redistribution**: macOS components may not be redistributed

Embedding data extracted from proprietary Apple binaries into open-source code would likely violate these terms.

### 5.3 Trademark Concerns

Spoofing SMBIOS identifiers to claim the VM is manufactured by "Apple Inc." running "Apple Virtualization" raises trademark concerns under the Lanham Act, specifically:
- **False designation of origin** (15 U.S.C. §1125(a))
- **Likelihood of confusion** regarding the source of the virtualization platform

### 5.4 Trade Secret Risks

The undocumented IOCTL protocols and verification mechanisms may constitute Apple trade secrets. Publishing their details and reimplementations in open-source code could expose contributors to trade secret misappropriation claims.

### 5.5 Risk Summary

| Risk Category | Severity | Mitigation |
|--------------|----------|------------|
| DMCA §1201 circumvention | **Critical** | Cannot be mitigated while maintaining the feature |
| EULA violation | **High** | Cannot be mitigated without Apple's permission |
| Trademark misuse | **High** | Could be partially mitigated by using generic identifiers |
| Trade secret exposure | **Medium** | Cannot be mitigated once published |

---

## 6. Upstream Status

### 6.1 libkrun's Rosetta History

The upstream `containers/libkrun` repository previously contained Rosetta support using a similar approach (spoofing the virtio-fs verification via a secret file at `~/.krunvm-rosetta`). This support was **explicitly removed** in a commit titled ["**macos: drop Rosetta support**"](https://github.com/containers/libkrun/commit/0b6a735626ab5a38b3532cf01802ae0cf57aaf2b).

While the specific motivation for the removal was not publicly documented in detail, the timeline and nature of the change are consistent with the legal concerns outlined in this report.

### 6.2 Implications for Contributions

Given that the upstream maintainers have already **deliberately removed** equivalent functionality, a Pull Request reintroducing the same approach would almost certainly be rejected. Any contribution in this area would need to use a fundamentally different, legally sound approach.

---

## 7. How Other OSS Projects Handle Rosetta

### 7.1 Podman with AppleHV Backend (vfkit)

Podman's official macOS support uses [vfkit](https://github.com/crc-org/vfkit), which wraps `Virtualization.framework`. Rosetta support is enabled through Apple's public API (`VZLinuxRosettaDirectoryShare`), with **no reverse engineering required**.

- Rosetta is enabled by default on Apple Silicon
- The integration is fully sanctioned by Apple's framework
- No proprietary data is embedded in the source code

### 7.2 Docker Desktop

Docker Desktop uses `Virtualization.framework` for macOS virtualization, with Rosetta support enabled through official APIs. Since Docker Desktop 4.25, Rosetta integration is generally available (no longer experimental) and provides near-native x86_64 emulation performance on Apple Silicon. Docker also offers their own "Docker VMM" hypervisor (since 4.35), though it does not currently support Rosetta.

- Source: [Docker Desktop Release Notes — 4.25](https://docs.docker.com/desktop/release-notes/#4250)
- Source: [Docker Desktop Settings — General](https://docs.docker.com/desktop/settings-and-maintenance/settings/#general)

### 7.3 UTM

[UTM](https://mac.getutm.app/) is an open-source virtualization app (Apache 2.0 license) that uses both QEMU and `Virtualization.framework` as backends. When using the `Virtualization.framework` backend on macOS 13+, UTM provides an "Enable Rosetta" checkbox that leverages Rosetta for x86_64 binary translation in ARM Linux guests. When using the QEMU backend, UTM does not attempt to enable Rosetta.

- Source: [UTM Documentation — Rosetta](https://docs.getutm.app/guides/ubuntu/)
- Source: [UTM GitHub Repository](https://github.com/utmapp/UTM)

### 7.4 Asahi Linux

The [Asahi Linux](https://asahilinux.org/) project, which brings Linux to Apple Silicon, takes a particularly strict stance on intellectual property:

- **Explicitly forbids** using non-publicly-available copyrighted materials (leaked software, unreleased documentation, non-public betas)
- Uses only **clean-room reverse engineering** techniques
- Acknowledges that running Rosetta outside macOS is **legally impermissible** under Apple's licensing terms
- Integrates **FEX-Emu** in Fedora Asahi Remix for x86/x86_64 emulation as a legitimate alternative

- Source: [Asahi Linux Copyright Policy](https://asahilinux.org/copyright/)
- Source: [Fedora Asahi Remix — FEX-Emu Integration](https://fedoraproject.org/wiki/Changes/FEX)

### 7.5 Pattern

Every major OSS project that integrates Rosetta does so through `Virtualization.framework`'s public API. **No open-source project ships Rosetta support via Hypervisor.framework circumvention.**

---

## 8. Alternative Approaches

### 8.1 Status Quo: QEMU User-Mode Emulation

**Performance**: ~20–40% of native  
**Legal Risk**: None  
**Effort**: Zero (already included in Fedora CoreOS)

QEMU user-mode is already available in the guest OS. While slower than Rosetta, it provides broad x86_64 compatibility with no legal concerns.

### 8.2 FEX-Emu Integration

**Performance**: ~50–70% of native  
**Legal Risk**: None (MIT licensed, clean-room implementation)  
**Effort**: Medium

[FEX-Emu](https://github.com/FEX-Emu/FEX) is a JIT-based x86/x86_64 emulator designed for ARM64 Linux. It offers 2–3x better performance than QEMU user-mode, with no legal restrictions. On Apple Silicon, it can leverage the CPU's TSO (Total Store Ordering) mode for improved x86 memory model compatibility.

**Challenges**:
- Requires a 4K page-size kernel (Apple Silicon natively uses 16K)
- Not yet packaged in Fedora CoreOS by default
- May need a microVM approach for page-size compatibility

### 8.3 Box64 Integration

**Performance**: ~40–57% of native  
**Legal Risk**: None (MIT licensed, clean-room implementation)  
**Effort**: Medium

[Box64](https://github.com/ptitSeb/box64) is another open-source x86_64 emulator. It works with 16K page sizes (unlike FEX-Emu), making it potentially easier to deploy on Apple Silicon. It has been successfully used on Asahi Linux.

### 8.4 Virtualization.framework Backend for libkrun

**Performance**: ~70–80% of native (with Rosetta)  
**Legal Risk**: None  
**Effort**: Not feasible in practice

In theory, adding a `Virtualization.framework` backend to libkrun would enable official Rosetta support. However, based on my research, this approach is **not technically viable**. libkrun's developers have explicitly chosen `Hypervisor.framework` because `Virtualization.framework` is closed-source and does not allow implementing custom virtual devices or altering functionality — capabilities essential to libkrun's features such as GPU passthrough via `virtio-gpu` with Venus. In fact, `libkrun-efi` was specifically created as an **open-source alternative to `Virtualization.framework`**, not as a wrapper around it. Switching to `Virtualization.framework` would fundamentally contradict libkrun's architectural goals and eliminate its key differentiators over `vfkit`.

### 8.5 Performance Comparison

| Approach | Relative Performance | Legal Risk | Implementation Effort |
|----------|---------------------|------------|----------------------|
| Native ARM64 | 100% | — | — |
| Rosetta 2 (via Vz.framework + vfkit) | 70–80% | None | N/A (use vfkit instead) |
| FEX-Emu | 50–70% | None | Medium |
| Box64 | 40–57% | None | Medium |
| QEMU user-mode (current) | 20–40% | None | None |
| Rosetta circumvention | 70–80% | **Critical** | Done (but unpublishable) |

---

## 9. Conclusions

1. **Rosetta 2 is architecturally bound to Virtualization.framework.** Apple has implemented multiple layers of protection to enforce this restriction.

2. **Circumventing these protections is technically possible** but carries unacceptable legal risks for an open-source project, particularly under the DMCA.

3. **The upstream libkrun project already removed equivalent functionality**, signaling that the maintainers share these concerns.

4. **Every legitimate OSS integration of Rosetta uses Apple's official API** (`VZLinuxRosettaDirectoryShare`). There is no precedent for an accepted circumvention approach in open-source projects.

5. **Viable alternatives exist.** FEX-Emu and Box64 can significantly improve x86_64 emulation performance (2–3x over QEMU) without any legal risk.

I conducted this investigation to better understand the technical and legal landscape, and I am sharing it in case it is useful to others exploring similar questions. While the direct circumvention approach turned out to be unsuitable for open-source distribution, I hope the findings here — particularly around the architectural constraints and available alternatives — will be a useful reference.

---

## 10. References

### Apple Documentation
- [Virtualization Framework](https://developer.apple.com/documentation/virtualization)
- [Hypervisor Framework](https://developer.apple.com/documentation/hypervisor)
- [Running Intel Binaries in Linux VMs with Rosetta](https://developer.apple.com/documentation/virtualization/running_intel_binaries_in_linux_vms_with_rosetta)
- [VZLinuxRosettaDirectoryShare](https://developer.apple.com/documentation/virtualization/vzlinuxrosettadirectoryshare)

### Legal References
- [DMCA §1201 — Circumvention of Copyright Protection Systems](https://www.law.cornell.edu/uscode/text/17/1201)
- [Lanham Act §43(a) — False Designations of Origin](https://www.law.cornell.edu/uscode/text/15/1125)

### Related Projects
- [libkrun](https://github.com/containers/libkrun) — Lightweight VMM library
- [Podman](https://github.com/containers/podman) — Container management
- [vfkit](https://github.com/crc-org/vfkit) — Virtualization.framework CLI tool
- [FEX-Emu](https://github.com/FEX-Emu/FEX) — x86/x86_64 emulator for ARM64
- [Box64](https://github.com/ptitSeb/box64) — x86_64 emulator
- [Asahi Linux Copyright Policy](https://asahilinux.org/copyright/)

### Community
- [Podman Discussions](https://github.com/containers/podman/discussions)
- [libkrun Issues](https://github.com/containers/libkrun/issues)

---

*This document is published under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). The analysis represents the author's technical assessment and does not constitute legal advice.*
