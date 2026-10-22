# Compute Plug IDE: A Hardware-Bound Remote Backend for Air-Gapped Environments

**White paper and architectural reference design**
**Version:** 1.1.0 (revision of 1.0.0) | **Date:** October 2026
**Author:** Ashwin H (GitHub: ashwin9390)
**Status:** Architectural specification. No prototype exists yet. Every
hardware and performance figure is a target to validate, not a measurement.
**Reference model:** JetBrains Remote Development (Gateway)

---

## 1. Executive Summary

Client-server IDEs, such as JetBrains Remote Development launched through
Gateway, let a thin client drive a backend that does the heavy work. AI
coding assistants have become a standard part of that workflow.

Regulated and air-gapped environments (defense contractors, financial
institutions, intelligence labs, healthcare providers, secure R&D
facilities) often cannot adopt either. Typical constraints:

1. No external connectivity, so no cloud APIs and no cloud AI assistants.
2. Source code and build artifacts must not land on unmanaged host disks.
3. A remote-backend model fails when no approved network route to the
   backend exists.

**Proposal.** Keep the client-server model but change the transport.
Instead of reaching a server across a network, package the backend (build
tools, language analysis, and a small local coding model) inside a
bus-powered USB-C appliance. The host sees a virtual Ethernet adapter and
talks to the backend over a private point-to-point link.

This is a new physical deployment target for an existing architecture, not
a new IDE architecture.

---

## 2. System Architecture

```
+---------------------------------------------------------------------+
|                    COMPUTE PLUG (USB-C DEVICE)                      |
|                                                                     |
|  +----------------+   +-------------------+   +------------------+  |
|  | USB gadget     |   | ARM SoC           |   | Encrypted NVMe / |  |
|  | CDC-NCM / ECM  |<->| Embedded Linux    |<->| UFS storage      |  |
|  | + USB-PD sink  |   | IDE backend + LSPs|   |                  |  |
|  +----------------+   +---------+---------+   +------------------+  |
|         |                       |                                   |
+---------|-----------------------|-----------------------------------+
          |                       |
 (virtual Ethernet,        (NPU or CPU inference)
  private link address)            |
          v                        v
 +----------------+      +-----------------------------------------+
 | HOST MACHINE   |      | Small local code model                  |
 | thin client    |      | (completion, lint help, single-file     |
 | (see section 3)|      |  rewrites; NOT multi-file agentic work) |
 +----------------+      +-----------------------------------------+
```

### 2.1 Hardware interface layer

- **Link protocol:** USB gadget mode presenting a virtual Ethernet
  adapter. CDC-NCM and CDC-ECM are the preferred classes. CDC-EEM has
  patchier host-OS support and should only be listed after testing.
  Per-OS driverless behavior (notably older Windows releases, which have
  historically needed RNDIS or an installed driver) must be verified on
  real hosts before any "works everywhere" claim.
- **Addressing:** a private link address (for example `192.168.7.1`). This
  is not loopback.
- **Power:** the device draws from the host port via USB Power Delivery or
  default USB power. Many laptop ports supply well below 15 W, and
  locked-down machines may restrict it further. The design must degrade
  gracefully (reduced clocks, LSP-only mode) and the real power curve must
  be measured under indexing load.
- **Thermal:** a passive metal enclosure is a reasonable starting point.
  Anything more elaborate should wait for prototype thermal data.

### 2.2 Backend and storage

- **Storage:** NVMe or UFS rather than SD/eMMC, because indexing and builds
  generate heavy small random I/O and write wear. Choose parts with
  adequate endurance ratings and keep transient indexes on RAM-backed
  storage where memory allows. NVMe power draw must be counted in the
  device's power budget.
- **JVM languages (Java, Kotlin, Scala):** a headless JetBrains IDE
  backend as used by Remote Development.
- **Other languages (Rust, C/C++, Python, and others):** standard Language
  Server Protocol servers running on the device.
- **Toolchains:** JDKs, SDKs, and compilers are baked into the image so
  the environment is identical wherever the device is plugged in.
- **Open question:** Gateway documents Linux servers as supported
  backends. Whether current IDE backends run acceptably on the aarch64
  Linux this hardware would use, within its memory budget, is unverified.

### 2.3 On-device AI assistant

- **Hardware:** an SoC with an NPU (RK3588-class is one example), or
  CPU-only inference on simpler boards. NPU choice should follow
  measured model throughput, not marketing figures.
- **Preferred model candidate:** Mellum-all, JetBrains' open-source
  code-completion model, run through a local runtime. Alternatives to
  benchmark: small code-tuned models such as Qwen-Coder.
- **Scope:** inline completion, lint explanations, single-file rewrites.
  Small on-device models are not a substitute for cloud agentic coding on
  multi-file tasks, and this paper does not claim they are.

---

## 3. Client Modes: What "Zero Install" Really Means

| Mode | Host requirement | Notes |
|---|---|---|
| **A. JetBrains Client** | JetBrains Client installed on the host, pre-approved by IT | Closest to the existing remote-development experience, with full IDE features. Requires a client install, so it is not zero-install. |
| **B. Browser frontend** | A browser only | Genuinely zero-install, but not an existing JetBrains product. Needs a separate web IDE layer with weaker feature parity. |

Claiming both full IDE features and zero host installation at once would
be inaccurate. Mode A is the credible path for JetBrains-quality features
in managed environments, where IT can pre-approve a client. Mode B is a
lighter fallback for locked-down hosts.

---

## 4. Data Flow

1. **Connect.** The user plugs the device into a host USB-C port. The OS
   enumerates a virtual Ethernet adapter and the host receives a private
   link address.
2. **Launch.** The user opens the thin client (Mode A or B) and connects
   to the device's address.
3. **Edit, build, test.** Source changes are saved to the device's
   encrypted storage. Builds, tests, and static analysis run in the
   device's embedded Linux. No host disk access is required.
4. **Local assistance.** Completion and lint help come from the on-device
   model, with no network traffic.
5. **Optional cloud bridge (off by default).** When network policy
   permits and the user explicitly enables it, a failing file and its
   stack trace can be escalated to a cloud model (for example JetBrains AI
   Assistant) via bounded tool calls. The patch is applied locally and the
   test is re-run, up to a small retry limit. Only the minimum necessary
   context leaves the device, and the toggle is visible and auditable. The
   air-gapped case never depends on this path.

---

## 5. Enterprise Enrollment and Security Posture

The device is meant to work with IT policy, not around it:

- Stable device identity and signed firmware, so IT can allow-list it.
- Standard, documented USB device classes.
- Encrypted storage, with key handling suited to the target environment.
- A documented enrollment workflow for managed USB-control tooling.

The project does not aim to evade endpoint detection or USB controls. In
the regulated environments this targets, a device that fights the security
policy will not be adopted.

---

## 6. Risks and Open Questions

| Risk or open question | Mitigation direction |
|---|---|
| Can a full IDE backend run inside the power, memory, and thermal budget? | Measure first; fall back to LSP-only mode for lighter languages |
| Port power varies widely across hosts | Degraded operating modes; measure real draw |
| Flash wear from indexing and builds | NVMe/UFS with adequate endurance; RAM-backed transient indexes |
| Thermal throttling in a small enclosure | Prototype thermal testing before enclosure design |
| Per-OS driverless USB behavior | Test on real Windows, macOS, Linux, and iPadOS hosts |
| USB policy blocks in managed fleets | Standard classes, signed firmware, IT enrollment workflow |
| Local model quality ceiling | Tiered design: local for fast tasks, optional cloud for deep ones |
| IDE plugin support in a headless backend | Scope to a known-working subset and document it |
| Overlap with existing remote-dev tooling | Position as a deployment target for it, not a replacement |

---

## 7. Prototype Plan

1. **Weeks 1-4:** USB gadget mode on a small ARM board (Raspberry Pi Zero
   2 W or Radxa Zero) serving a minimal web page to Windows, macOS, and
   Linux hosts. Then attempt a headless backend and a Mode A connection.
2. **Weeks 5-8:** Measure memory, indexing time, link latency, and power
   draw. Publish the numbers, including unflattering ones.
3. **Weeks 9-12:** Add local model completion, then the optional cloud
   bridge. Record a demo that works with the network unplugged.

---

## 8. Related Work

- **JetBrains Remote Development / Gateway:** the architecture this
  project extends to a USB-attached backend.
- **Mellum:** JetBrains' completion model, the preferred local model
  candidate.
- **Fleet:** discontinued by JetBrains in December 2025, so this design
  does not depend on it.
- **Cloud and hybrid AI IDEs (Cursor, Replit, Project IDX, Trae):** adjacent
  products that assume connectivity. The gap this project targets is the
  disconnected, hardware-isolated case.

---

## 9. Summary

Compute Plug IDE does not require a new IDE architecture. It proposes a
new transport and hardware boundary for the client-server model that
already exists, aimed at environments where a reachable backend is not an
option. The central unproven question is whether a useful backend fits the
power, memory, and thermal limits of a bus-powered device. The prototype
plan exists to answer that question with data before any further claims
are made.

---

*Copyright 2026 Ashwin H, licensed under Apache License 2.0 (see `LICENSE`).
JetBrains, IntelliJ, Gateway, and Mellum are trademarks of JetBrains s.r.o.
This is an independent project and is not affiliated with or endorsed by
JetBrains.*
