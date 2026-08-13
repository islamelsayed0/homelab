# Home Lab: Architecture, Operations, and Lessons

A working home lab built as a study platform for Linux and cloud certification, a smart home control plane, and a place to practice the operational habits that production environments actually require: monitoring, alerting, backup discipline, and change control.

This document is the public edition. It covers architecture, design reasoning, and the mistakes worth writing down. Operational specifics such as addressing, device inventory, and physical security details are deliberately excluded. See [What this document does not cover](#what-this-document-does-not-cover).

---

## Contents

- [Architecture at a glance](#architecture-at-a-glance)
- [Layer 0: Physical and network](#layer-0-physical-and-network)
- [Layer 1: Compute and virtualization](#layer-1-compute-and-virtualization)
- [Layer 2: Observability and operations](#layer-2-observability-and-operations)
- [Layer 3: Knowledge and agent orchestration](#layer-3-knowledge-and-agent-orchestration)
- [Design principles](#design-principles)
- [Lessons learned](#lessons-learned)
- [Troubleshooting playbook](#troubleshooting-playbook)
- [Performance notes](#performance-notes)
- [Roadmap](#roadmap)
- [What this document does not cover](#what-this-document-does-not-cover)
- [License](#license)

---

## Architecture at a glance

The lab is designed as four layers rather than a pile of hardware. Each has a different failure mode, a different tolerance for experimentation, and a different upgrade trigger. Ordering them by blast radius is the single most useful framing I have applied to it, because it decides what may be changed on a weeknight and what may not.

![Home lab architecture in four layers](diagrams/architecture_layers.svg)

The rule that falls out of this: **anything the household depends on is production.** Internet access and home automation are not lab equipment. They get change windows, rollback plans, and weekend mornings rather than midnight experiments.

---

## Layer 0: Physical and network

A self built rack carrying the compute node, the control plane host, and the wired distribution layer. Internet is 1 Gbps symmetric fiber. Wireless is a Wi‑Fi 7 mesh running in access point mode behind the gateway, so routing, DHCP, and NAT stay in one place instead of being split across two devices.

![Network and service topology](diagrams/network_topology.svg)

Remote access is a Tailscale overlay and nothing else. There is no inbound port forwarding anywhere in the environment, and no service is published to the internet. This is a deliberate constraint rather than a limitation: it removes an entire category of exposure and it costs nothing in usability.

**The honest limitation at this layer:** the switching is unmanaged, so every wired segment shares one broadcast domain. Creating additional SSIDs on consumer mesh hardware gives you separate network names, not isolation, because there is no VLAN tagging to enforce the boundary. Recognising that distinction mattered more than any purchase, because it is the difference between believing you are segmented and actually being segmented. Managed switching is on the roadmap and is the real prerequisite for segmentation.

---

## Layer 1: Compute and virtualization

Proxmox VE on a small form factor node, running three guests with distinct jobs:

| Guest | Purpose | Notes |
|---|---|---|
| Rocky Linux | RHCSA study environment | Snapshotted at a clean baseline so destructive exercises revert in seconds |
| Home Assistant OS | Smart home control plane | Treated as a household service, not a lab toy |
| Docker host | Container workloads | The landing spot for DNS filtering, secrets management, and log collection |

Snapshot discipline is applied to all guests at known good states. The Rocky guest in particular is built to be broken: the entire point of an RHCSA environment is that you can destroy the bootloader, corrupt a filesystem, or lock yourself out of SELinux and roll back in under a minute.

**Structural honesty about this layer:** it is one node with one disk carrying three workloads. That is a normal and appropriate lab design, but it concentrates risk, and pretending otherwise would be the kind of self deception that makes labs useless as preparation. Multi node clustering is a deliberate future step, ahead of faster networking, because two modest nodes teach quorum, live migration, and failover while one fast node teaches nothing new.

---

## Layer 2: Observability and operations

This is the layer that turns a collection of services into something operable, and the part I would point to first as evidence of production thinking.

- **Metrics collection** for per second resolution on host and service health
- **Uptime and endpoint monitoring** for availability of internal services
- **A custom Python security event collector**, written rather than installed, so the collection logic is explainable rather than a black box
- **An external dead man switch**, hosted off site
- **Out of band alert delivery** to a phone, independent of the local network
- **A consolidated status dashboard** so health is one view rather than four browser tabs

### The design decision that matters most here

**Monitoring that runs on the thing it monitors cannot report its own death.** If the control plane host fails, every check on it fails silently, and silence is indistinguishable from health. The external dead man switch resolves that: the host checks in on a schedule from inside, and the absence of that check in triggers an alert from outside. It is the only monitor in the environment that survives the monitoring host failing.

The same reasoning applies to alert delivery. Notifications route out of band, so an alert about the network does not depend on the network behaving.

### Known gaps at this layer

Metrics answer *how much*. Logs answer *why*. Centralised log aggregation is the clearest remaining gap, alongside SMART attribute alerting on disks, since drives usually warn before they fail. Long term metric retention is also unconfigured, which means slow degradation over months is currently invisible.

---

## Layer 3: Knowledge and agent orchestration

The layer that makes this lab unusual. Most stop at services.

Multiple agent surfaces (Claude Code, Codex CLI, OpenClaw) coordinate through a SQLite backed room structure, with the control plane host acting as the coordinator and a central knowledge vault serving as shared memory and export target. Work performed by any agent lands in one durable, searchable place rather than scattering across tool specific histories.

Infrastructure access for agents is designed around three constraints, and they are not negotiable for convenience:

1. **Read only by default.** Query access is the normal mode.
2. **Reachable only over the private overlay.** Never exposed to the internet.
3. **Propose and approve on every write.** An agent proposes a change and a human approves it. Nothing that alters infrastructure state executes unattended.

The reasoning: an agent with write access to a hypervisor or a home automation controller is an agent that can take down the household network. The approval gate costs a few seconds and removes the entire class of outcome where an automated action does something irreversible at three in the morning.

---

## Design principles

These are the rules the lab is actually built on, stated plainly because they are more portable than any hardware list.

**Snapshots are not backups.** A snapshot protects against a bad change. It does not protect against a failed disk, a stolen machine, or ransomware, because it shares fate with the volume it lives on. Treating snapshots as backup coverage is the most common self inflicted data loss in home labs, and it is convincing precisely because rollback works so well for the problems it does solve.

**Three copies, two media types, one off site.** Nothing is considered protected until that is true. In practice the highest value data in a lab is rarely the virtual machines. It is the accumulated notes, configuration, and documentation, which is also the thing most likely to exist in exactly one place.

**An untested backup is a rumour.** Backup jobs that have never been restored from are a belief, not a capability. Restore testing is scheduled work.

**A silent backup is worse than no backup**, because it produces confidence without protection. Backup jobs are wired into the same alerting as everything else, so a job that quietly stopped running two months ago surfaces immediately rather than at the moment it is needed.

**Household services are production.** If people who did not build the lab depend on it, it gets change control.

**Separate the trusted path from the convenient one.** No inbound exposure, no port forwarding, one overlay for remote access.

**Name the weakness in writing.** Every environment has unfinished work. Documenting it is what makes it a roadmap instead of a surprise.

---

## Lessons learned

**A separate SSID is not a separate network.** Isolation is enforced at layer 2 by VLAN capable switching, not by naming a wireless network something different. Consumer mesh hardware will happily let you build the illusion.

**One flat network means one compromise radius.** A camera, an appliance, or an untrusted device on the same broadcast domain as a hypervisor management interface is a design decision, whether or not it was made deliberately.

**Fiber does not need a modem.** The optical terminal is already a separate device, which means the ISP supplied router is replaceable in a way that cable equipment usually is not. Knowing which handoff is in use decides whether a future upgrade is a purchase or a purchase plus a support call.

**Real world behaviour beats benchmark numbers.** Being able to game and download large titles simultaneously without visible lag is a stronger signal of network health than whether a speed test reports 550 or 800 Mbps. Speed tests vary by server and by load, and chasing the number is a distraction.

**The bottleneck is frequently not the thing being blamed.** During large game installs, throughput approached the gigabit ceiling while the system NVMe sat at 100 percent active time. The network was fine. Storage was the constraint. Measuring before diagnosing saved a pointless network investigation.

**Bus powered external drives fail at the cable, not the platter.** Micro USB connectors on portable drives are the most common cause of a backup job dying mid write. Suspect the cable before the disk.

**Consumer external drives use hardware encryption in the USB bridge.** If that bridge dies, moving the bare drive to a SATA port does not recover the data. This is a hard argument against ever letting one be the only copy of anything.

**Secrets belong in a vault, not a config file.** I learned this the expensive way on an earlier project, through a real credential exposure that required rotating everything and rewriting repository history. The lesson is now encoded in the environment rather than merely remembered.

**Sanitize before the first commit, not after.** Publishing an environment writeup and cleaning it in a later commit leaves the original permanently recoverable in git history. The redaction has to happen before the repository exists.

---

## Troubleshooting playbook

Condensed from actual incidents rather than written from theory.

<details>
<summary><b>Wired throughput drops on a single machine</b></summary>

1. Confirm the negotiated link speed at the operating system level before testing anything else. A port that negotiated at 100 Mbps looks exactly like a slow internet connection.
2. Pause background updaters and game launchers, then test three times against the same server. One test against one server is noise.
3. Move the machine to a different switch port. Port position should not matter, but a faulty port, cable, or duplex setting is common and cheap to rule out.
4. Test directly into the gateway. If direct is fast and switched is slow, the problem is the switch, its uplink, or the cable to it.
5. Remember that connected devices do not divide bandwidth equally. Only active traffic consumes it.
</details>

<details>
<summary><b>Wireless is inconsistent in part of the coverage area</b></summary>

1. Confirm which mesh node the client is actually associated with. Clients hold onto a distant node longer than expected.
2. Place satellites where they still hold a strong link back to the wired node, not at the far edge of the dead zone. A satellite with a weak backhaul extends the problem rather than solving it.
3. Avoid running two wireless systems in parallel. Once the mesh is verified, disable the gateway radios so they are not competing.
</details>

<details>
<summary><b>A machine feels sluggish during large downloads</b></summary>

1. Check disk active time before blaming the network or memory. Sustained 100 percent disk utilisation is the usual culprit during install and decompression.
2. Memory at 70 to 75 percent is not automatically a problem. Look for paging, not for a high percentage.
</details>

<details>
<summary><b>A virtual machine will not start</b></summary>

1. Check host storage capacity first. Accumulated snapshots silently consuming the volume is the most common cause in a single node environment.
2. Roll back to the last known good snapshot rather than debugging live, then investigate the failed state from a position of safety.
3. If the host filesystem is suspect after an unclean shutdown, stop writing to it and restore rather than repairing in place.
</details>

<details>
<summary><b>Alerts have gone quiet</b></summary>

1. Silence is ambiguous. It means either nothing is wrong, or the monitoring host is dead.
2. The external dead man switch is what disambiguates. If it has not fired, the host is alive and the silence is real.
3. Test the alert delivery path on a schedule. An alerting path that has never fired in anger is an assumption.
</details>

---

## Performance notes

| Observation | Finding |
|---|---|
| Baseline wired throughput | Roughly 850 to 900 Mbps on a clean test |
| Gateway latency | Sub millisecond, zero loss, indicating a healthy local path |
| Repeated speed tests | Varied widely by server and by concurrent load, from roughly 500 to 800 Mbps |
| Large game download | About 655 Mbps sustained to disk, with the interface later observed near 953 Mbps |
| Bottleneck during install | System NVMe at 100 percent active time while CPU sat near 18 percent |
| Resolution of an intermittent slowdown | Relocating the machine to a different switch port restored expected throughput |

The practical conclusion: the environment can saturate its uplink, and variance between individual tests is not evidence of a fault.

---

## Roadmap

Ordered by value delivered rather than by how interesting the project is. The unglamorous items are first on purpose.

| Priority | Item | Why it ranks here |
|---|---|---|
| 1 | Automated backup chain with an off site copy | Closes the largest structural risk. Everything else assumes the data survives. |
| 2 | Recovery runbook plus a tested restore | Converts backup from theory into verified capability |
| 3 | Power protection with graceful shutdown integration | Unclean shutdown is the most common cause of hypervisor filesystem damage |
| 4 | Managed switching and VLAN segmentation | The prerequisite for real isolation. The genuine reason to move off consumer gear. |
| 5 | DNS filtering, centralised logging, secrets vault | Fills the remaining service and operational gaps |
| 6 | Multi node cluster | Quorum, live migration, and failover are the next real learning step |
| 7 | Multigig internal networking | Only once local transfers actually saturate Gigabit |

Deliberately **not** on the roadmap: faster internet service. The internal network is the constraint, not the circuit, and buying bandwidth to solve a switching problem is how labs get expensive without getting better.

---

## What this document does not cover

Excluded on purpose: internal addressing and network layout, device models and serial numbers, physical security arrangements, specific configuration files, credentials of any kind, and location details.

This is not incidental. An environment writeup that includes a live inventory of what is deployed and what is currently unprotected is a useful document for exactly the wrong audience. The architecture and the reasoning are the parts worth publishing. The operational specifics stay private, and that boundary is itself part of the design.

---

## License

This work — the written documentation and the diagrams — is licensed under the [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/) (CC BY 4.0). The full legal text is in [LICENSE](LICENSE).

You are free to share and adapt the material for any purpose, including commercially, provided you give appropriate credit, link to the license, and indicate whether changes were made.

Suggested attribution:

> "Home Lab: Architecture, Operations, and Lessons" by [islamelsayed0](https://github.com/islamelsayed0/homelab), licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
