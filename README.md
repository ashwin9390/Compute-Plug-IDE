# Compute Plug IDE

> **A USB-attached remote development backend for air-gapped environments.**

**Status: architectural specification. No prototype code yet.** This repo
documents a design and the open questions around it. Treat every hardware
and performance figure below as a target to validate, not a measured result.

## 1. The Problem

Client-server IDEs (JetBrains Gateway / Remote Development, for example)
let a thin client drive a backend that does the heavy lifting. They assume
the client can reach that backend over a network.

Regulated and air-gapped environments (defense, finance, healthcare,
secure R&D labs) often cannot grant that:

1. No external connectivity, so no cloud APIs and no cloud AI assistants.
2. Source and build artifacts must not land on unmanaged host filesystems.
3. Developers move between locked-down machines with inconsistent toolchains.

## 2. The Idea

Keep the client-server model, but change the transport. Instead of a
remote server reached over a network, the backend lives inside a
bus-powered USB-C appliance. The host sees a virtual network adapter and
talks to the backend over a private point-to-point link. Source code,
toolchains, indexes, and a small local AI model stay on the device.

This is not a new IDE architecture. It is a new physical deployment target
for an existing one.

## 3. Client Modes (and what "zero install" really means)

| Mode | Host requirement | Status |
|---|---|---|
| **A. JetBrains Client** (Gateway / Toolbox launch) | JetBrains Client installed on the host, pre-approved by IT | Closest to existing JetBrains remote development; the device would act as the Linux [...] |
| **B. Browser frontend** | A browser only | Experimental. Not an existing JetBrains product; would need a separate web IDE layer and has weaker feature parity |

Mode A is the credible path for JetBrains-style IDE features, but it means
a client install on the host. Mode B delivers true zero-install but is a
different, lighter experience. The project should be explicit about this
tradeoff rather than claim both at once.

## 4. Architecture

```
+-----------------------------------------------------------------+
|                  COMPUTE PLUG (USB-C DEVICE)                    |
|                                                                 |
|  +---------------+   +----------------------+   +------------+  |
|  | USB gadget    |   | ARM SoC, embedded    |   | Encrypted  |  |
|  | (CDC-NCM /    |<->| Linux                |<->| NVMe / UFS |  |
|  |  ECM)         |   | Headless IDE backend |   | storage    |  |
|  +---------------+   | + LSP servers        |   +------------+  |
|         |            +----------+-----------+                   |
+---------|-----------------------|-------------------------------+
          |                       |
 (virtual Ethernet,        (NPU or CPU inference)
  private link address)            |
          v                        v
   +-------------+        +----------------------------+
   | HOST        |        | Small local code model     |
   | thin client |        | (completion, lint help,    |
   +-------------+        |  small rewrites only)      |
                          +----------------------------+
```

### A. USB interface
- Linux USB gadget mode (`libcomposite`) presenting a virtual Ethernet
  adapter. CDC-NCM and ECM are preferred over legacy RNDIS for driverless
  operation on current Windows, macOS, Linux, and iPadOS.
- The link uses a private address (for example `192.168.7.1`), not
  loopback.

### B. Enterprise enrollment
- The device should be easy for IT to **enroll and allow-list** (stable
  device identity, signed firmware, documented USB class behavior).
- It is not designed to evade endpoint security or USB controls. In a
  regulated environment, working with the policy is the only viable path.

### C. Backend
- JVM languages: a headless JetBrains IDE backend, as used by Remote
  Development.
- Other languages: standard Language Server Protocol servers.
- Toolchains (JDKs, SDKs, compilers) are baked into the image so the
  environment is identical wherever the device is plugged in.
- Open question: Gateway documents Linux servers as supported backends;
  whether the current IDE backends run well on the aarch64 Linux this
  hardware would use must be verified.

### D. Local AI assistance
- First candidate: **Mellum-all**, JetBrains' open-source code-completion
  model, run through a local runtime. Alternatives to benchmark: small
  code-tuned models such as Qwen-Coder.
- Scope: inline completion, lint explanations, single-file rewrites.
  **Not** multi-file agentic workflows; small on-device models cannot
  match cloud agents there.
- **Optional cloud bridge** (off by default, explicit and auditable
  toggle): when network policy allows, escalate a failing file plus
  stack trace to a cloud model through bounded tool calls. The air-gapped
  case never depends on it.

## 5. Open Questions (unproven)

1. Can a full IDE backend (indexing, code analysis, builds) run acceptably
   inside the power, memory, and thermal budget of a bus-powered device?
   This is the first thing to measure.
2. What sustained power can a typical locked-down laptop port actually
   deliver, and how does throttling behave under indexing load?
3. Which local model gives usable completion latency on the chosen
   NPU/CPU, and at what quality?
4. How much IDE plugin functionality survives in a headless backend?

## 6. Risks

| Risk | Mitigation direction |
|---|---|
| Backend memory footprint on small hardware | Measure early; fall back to LSP-only mode for lighter languages |
| Flash wear from indexing and builds | Require NVMe/UFS with wear-leveling; avoid SD/eMMC |
| Thermal throttling in a small enclosure | Prototype thermal testing before any enclosure design |
| USB policy blocks in managed fleets | Standard device classes, signed firmware, IT enrollment workflow |
| Local model quality ceiling | Tiered design: local for fast/simple, optional cloud for deep tasks |
| Overlap with existing JetBrains remote-dev tooling | Position as a deployment target for it, not a competitor |

## 7. Prototype Plan

1. **Weeks 1-4:** USB gadget mode on a Raspberry Pi Zero 2 W or Radxa
   Zero serving a minimal web page to Windows, macOS, and Linux hosts
   with no drivers. Then attempt a headless backend and Mode A client
   connection.
2. **Weeks 5-8:** Measure memory, indexing time, link latency, power draw.
   Publish the numbers, including the unflattering ones.
3. **Weeks 9-12:** Add local model completion, then the optional cloud
   bridge. Record a demo that works with the network unplugged.

## 8. Related Products

- **JetBrains Gateway / Remote Development:** the model this project
  extends to a USB-attached backend.
- **Mellum:** JetBrains' completion model, used here as the preferred
  local model candidate.
- **Fleet:** discontinued by JetBrains in December 2025, so this project
  does not depend on it. Components and ideas from it live on in other
  JetBrains tooling, including the newer agentic tool Air (in preview).

## 9. License

Copyright 2026 Ashwin H. Code is licensed under the [Apache License 2.0](LICENSE). Hardware designs, when added: CERN-OHL-W v2.
Model weights keep their own upstream licenses.

## 10. Contributing

Early stage. The most useful contributions right now are measurements:
backend footprint on small ARM boards, USB gadget behavior on locked-down
hosts, and local model latency numbers.

---

*JetBrains, IntelliJ, Gateway, and Mellum are trademarks of JetBrains s.r.o.
This is an independent project and is not affiliated with or endorsed by
JetBrains.*
