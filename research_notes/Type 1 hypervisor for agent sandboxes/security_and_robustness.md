# Security and Robustness Engineering for a Lightweight Type-1 Hypervisor (Agent Sandboxes)

Notes compiled 2026-11-26. Items marked **[dated]** are older than about 3 years and may not reflect the current state. Items marked **[secondary]** come from aggregators or press, not the primary advisory, and should be re-verified before being quoted in a final report.

## 1. Vulnerability history: which bug classes cause VM escapes or host compromise, and where?

### Takeaway
Across every major hypervisor, escapes cluster in (a) device emulation and paravirtual I/O backends, (b) complex per-VM memory-management logic (shadow paging, PV page tables, grant tables, nested virtualization), and (c) instruction emulation. Bugs in the bare VT-x/SVM entry/exit path are rare by comparison. The 2026 record shows that Rust VMMs remove most memory-corruption bugs but still have logic bugs, `unsafe` bugs, and bugs in the host kernel underneath them.

### Cited Findings

**Quantitative breakdowns (Xen, 2017 — [dated])**
- Of 191 XSAs studied, 144 (75.39%) were in the core hypervisor; the other 24.61% were mostly QEMU and toolstack flaws residing in Dom0. — [Shi et al., "Deconstructing Xen", NDSS 2017](https://dev.ndss-symposium.org/wp-content/uploads/2017/09/ndss2017_02A-4_Shi_paper.pdf)
- Core-hypervisor vulnerabilities by module: memory management 25.69%, CPU virtualization 21.53%, code (instruction) emulation 13.19%, I/O 9.03%, exception handling 5.56%, grant table 4.86%, global 4.17%, domain control 4.17%, domain building 3.47%, event channel 2.08%, XSM 1.39%, scheduler 0.69%, others 3.47%. — [Shi et al., NDSS 2017](https://dev.ndss-symposium.org/wp-content/uploads/2017/09/ndss2017_02A-4_Shi_paper.pdf)
- Core-hypervisor vulnerabilities by outcome: host DoS 61.81%, privilege escalation 15.28%, information leak 13.89%. By cause: memory corruption 45.14%, misuse of hardware feature 22.22%, live lock 8.33%, infinite loop 6.25%, false `BUG_ON` 6.25%, general fault 4.86%, resource exhaustion 4.17%, deadlock 3.47%. — [Shi et al., NDSS 2017](https://dev.ndss-symposium.org/wp-content/uploads/2017/09/ndss2017_02A-4_Shi_paper.pdf)
- More than half of the core vulnerabilities were in per-VM logic (guest memory management, CPU virtualization, instruction emulation); isolating that logic into per-VM "slices" would have contained 107 of 144 (74%) at 1.2% average overhead. — [Shi et al., NDSS 2017](https://dev.ndss-symposium.org/wp-content/uploads/2017/09/ndss2017_02A-4_Shi_paper.pdf)
- Xen grew to about 270K LoC by v4.0. — [Shi et al., NDSS 2017](https://dev.ndss-symposium.org/wp-content/uploads/2017/09/ndss2017_02A-4_Shi_paper.pdf)
- An earlier Princeton study of Xen and KVM found more than one third of attacks were due to I/O device emulation, and availability breaches were the most common outcome. **[dated, 2013; figure taken from a search summary of the report]** — [Perez-Botero et al., Princeton TR-960](https://www.cs.princeton.edu/techreports/2013/960.pdf)
- NIST classified Xen and KVM vulnerabilities by hypervisor functionality, attack type and attack source to guide forensic analysis. **[dated, 2019]** — [NISTIR 8221](https://csrc.nist.gov/pubs/ir/8221/final)

**QEMU / device emulation**
- VENOM (CVE-2015-3456, May 2015) was a buffer overflow in QEMU's virtual floppy disk controller, reachable from the guest even when no floppy was configured, affecting Xen, KVM and VirtualBox deployments that used QEMU. **[dated]** — [CrowdStrike VENOM page](https://venom.crowdstrike.com/)
- HyperPill found 26 new bugs in recent versions of QEMU, Hyper-V and the macOS Virtualization Framework, across four device categories. — [Bulekov et al., USENIX Security 2024](https://www.usenix.org/conference/usenixsecurity24/presentation/bulekov)

**KVM (core memory virtualization and nested virtualization)**
- CVE-2021-29657 was a KVM-only (no userspace VMM bug needed) guest-to-host escape in AMD nested SVM code, a TOCTOU on the guest-controlled VMCB. **[dated]** — [Project Zero, "An EPYC escape"](https://googleprojectzero.blogspot.com/2021/06/an-epyc-escape-case-study-of-kvm.html)
- "Januscape" (CVE-2026-53359, public 6 July 2026) is a use-after-free in KVM's x86 shadow MMU, introduced by a commit merged in August 2010, giving a guest-to-host path on Intel and AMD. Reports say exploitation needs root in the guest and nested virtualization exposed by the host; the public PoC panics the host. **[secondary]** — [The Hacker News](https://thehackernews.com/2026/07/16-year-old-linux-kvm-flaw-lets-guest.html); [SecurityWeek](https://www.securityweek.com/linux-kernel-vulnerability-allows-vm-escape-on-intel-and-amd-systems/); [Ubuntu](https://ubuntu.com/blog/januscape-linux-vulnerability-mitigations-available)
- A closely related bug in the same shadow-paging path, CVE-2026-46113, had been fixed upstream in May 2026. **[secondary]** — [TuxCare](https://tuxcare.com/blog/januscape-exposes-the-kvm-shadow-paging-bug-that-kept-coming-back/)

**Host kernel beneath a minimal VMM**
- "PwnEscape" (reported as CVE-2026-99999; October 2026) is a Linux host IPv4 networking bug, not a KVM or Firecracker bug: guest-supplied virtio-net checksum-offload metadata could alter an IPv4 header after host validation. It was found by Pwn.ai during a $1M challenge against Vercel's Firecracker-based sandbox; researchers showed host-memory leakage against production sandboxes and root code execution on their own systems. The fix (by M. Tsirkin) adds validation at five checksum-completion points. **[secondary; the CVE number looks unusual and should be verified]** — [CSO Online](https://www.csoonline.com/article/4237999/kvm-escape-exposes-broader-attack-surface-in-linux-networking.html)

**Xen**
- XSA-148 (CVE-2015-7835) was a PV page-table validation flaw allowing a PV guest to gain full control of the host. **[dated]** — [XSA-148](https://xenbits.xen.org/xsa/advisory-148.html)
- XSA-212 (CVE-2017-7228) was an insufficient check in the `memory_exchange` hypercall allowing PV guests to write hypervisor memory. **[dated]** — [XSA-212](https://xenbits.xen.org/xsa/advisory-212.html)
- XSA-472 (9 Sept 2025; CVE-2025-27466, CVE-2025-58142, CVE-2025-58143) covered multiple flaws in Xen's Viridian (Hyper-V enlightenment) interface, including NULL dereferences in reference-TSC and SIM-page handling. — [XSA-472](https://xenbits.xen.org/xsa/advisory-472.html)
- Recent Xen advisories include XSA-494 (mismatched mapcache metadata, CVE-2026-42488), XSA-490 (CPU opcode cache corruption, CVE-2025-54518) and XSA-508 (pygrub supported only in deprivileged mode). — [Xen XSA index](https://xenbits.xen.org/xsa/)

**VMware ESXi**
- CVE-2025-22224 (CVSS 9.3, March 2025) is a TOCTOU leading to out-of-bounds write in the VMX process; Broadcom stated it was exploited in the wild, alongside CVE-2025-22225 and CVE-2025-22226. **[secondary]** — [Kaspersky summary](https://www.kaspersky.com/blog/vmware-critical-vulnerabilities-cve-2025-22224/53167/)
- CVE-2025-41236 (VMSA-2025-0013, July 2025) is an integer overflow in the VMXNET3 virtual NIC allowing a guest administrator to execute code on the host; it was reported through Pwn2Own. — [NVD](https://nvd.nist.gov/vuln/detail/CVE-2025-41236); [ZDI-26-189](https://zerodayinitiative.com/advisories/ZDI-26-189)

**Hyper-V**
- Microsoft describes the Hyper-V guest-facing surface as hypercalls, intercepts/instruction emulation, VMBus channels and virtual devices, with the per-VM worker process (vmwp.exe) in user mode and VSPs in the root kernel. **[dated, 2018]** — [MSRC, "First steps in Hyper-V research"](https://msrc.microsoft.com/blog/2018/12/first-steps-in-hyper-v-research/)
- 2026 Hyper-V advisories include guest-to-host RCE and EoP entries described as out-of-bounds read, type confusion, heap overflow reachable via hypercall, use-after-free and TOCTOU into the VM worker process (e.g. CVE-2026-45607, MSRC CSAF dated 9 June 2026). **[secondary; third-party trackers conflict on scores and CWE labels, and several CVE numbers look unusual]** — [MSRC CSAF CVE-2026-45607](https://msrc.microsoft.com/csaf/advisories/2026/msrc_cve-2026-45607.json); [SentinelOne](https://www.sentinelone.com/vulnerability-database/cve-2026-45607/)

**Rust VMMs (Firecracker, Cloud Hypervisor, Kata)**
- CVE-2026-5747: out-of-bounds write in Firecracker's virtio PCI transport, versions 1.13.0–1.14.3 and 1.15.0, fixed in 1.14.4 / 1.15.1. Triggered by guest root changing queue configuration registers after activation. Only reachable with `--enable-pci`; the default MMIO transport is unaffected. Reported by Anthropic. Sources disagree on severity (8.7 vs 7.5) and on impact (VMM crash vs host code execution). — [AWS bulletin 2026-015](https://aws.amazon.com/security/security-bulletins/2026-015-aws); [NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-5747); [Red Hat](https://access.redhat.com/security/cve/CVE-2026-5747)
- CVE-2026-1386: symlink-following in the Firecracker jailer allowed a local host user to overwrite arbitrary host files when the jailer ran as root; fixed in 1.13.2 / 1.14.1. — [AWS bulletin 2026-003](https://aws.amazon.com/security/security-bulletins/2026-003-AWS)
- CVE-2019-18960: buffer overflow in Firecracker's vsock implementation (v0.18.0, v0.19.0). **[dated]** — [NVD](https://nvd.nist.gov/vuln/detail/CVE-2019-18960)
- CVE-2026-27211: Cloud Hypervisor 34.0–50.0 allowed arbitrary host file exfiltration (within process privileges): a guest using a raw virtio-block image could overwrite its disk header with a crafted QCOW2 structure (image-format autodetection). Fixed in 50.1. A logic bug, not memory corruption. **[secondary]** — [INCIBE](https://www.incibe.es/en/incibe-cert/early-warning/vulnerabilities/cve-2026-27211)
- CVE-2026-47243: Kata Containers runtime-rs virtio-fs path allowed guest-root to host-root escape when virtiofsd ran as root unsandboxed (crafted `FUSE_SYMLINK`). Fixed in 3.31.0. **[secondary]** — [OSV](https://osv.dev/vulnerability/CVE-2026-47243)
- CVE-2023-41051: unsoundness in the rust-vmm `vm-memory` crate (0.1.0 to before 0.12.2). — [Wiz database entry](https://www.wiz.io/vulnerability-database/cve/cve-2023-41051)

### Inferences
- The Xen breakdown argues that the biggest wins are structural: no shadow paging, no PV MMU, no in-hypervisor instruction emulator, no grant-table-like shared-memory machinery in the privileged core, and no compatibility interfaces (Viridian). Together those categories (memory management + code emulation + grant table) were roughly 44% of core XSAs in the 2017 data.
- Nested virtualization recurs as an escape source (CVE-2021-29657, Januscape). For agent sandboxes it should be off by default, and if offered, treated as a separate higher-risk tier.
- Rust VMM CVEs in 2026 are dominated by (i) `unsafe`/index arithmetic in virtio transport state machines, (ii) filesystem and path logic (jailer symlink, virtio-fs, image-format probing). Design implications: forbid image-format autodetection, forbid device reconfiguration after activation, keep file-sharing devices out of the default profile.
- PwnEscape shows that for a KVM-style design the host network stack is in the TCB of every guest. A type-1 design that puts the network backend in a deprivileged per-VM or per-tenant component turns that class into a contained compromise.
- Roughly 60% of Xen core bugs were host DoS; robustness (preemption, bounded loops, no `BUG_ON` on guest-reachable paths) is a first-class security requirement in a multi-tenant sandbox host.

### Gaps
- No recent (post-2019) quantitative breakdown by component for XSAs, KVM CVEs or Hyper-V was found; the 2017 NDSS figures predate PVH-by-default and transient-execution XSAs.
- No authoritative list of crosvm guest-to-host CVEs was found.
- The exact trigger and nested-only scope of CVE-2026-53359 were not verified against the kernel commit.
- Whether CVE-2026-5747 is exploitable beyond DoS is unresolved between sources.

## 2. TCB minimization

### Takeaway
The proven pattern is a small privileged core that only multiplexes CPU, memory and interrupts, with one deprivileged VMM per VM and a minimal virtio-only device model. AWS's 2026 Nitro Isolation Engine goes a step further by deprivileging the hypervisor's own policy code beneath a verified isolation kernel.

### Cited Findings
- Firecracker is about 50k lines of Rust versus more than 1.4M lines for QEMU, boots guests in under 125 ms with under 5 MiB of per-VM overhead, and deliberately offers only a handful of devices (virtio-net, virtio-block, serial, minimal keyboard controller) with no BIOS, PCI (at the time) or instruction emulation. **[dated, 2020]** — [Agache et al., NSDI 2020](https://www.usenix.org/conference/nsdi20/presentation/agache)
- Firecracker's PCI transport, added later as an opt-in, is where its 2026 out-of-bounds write occurred; the legacy MMIO default was unaffected. — [AWS bulletin 2026-015](https://aws.amazon.com/security/security-bulletins/2026-015-aws)
- seL4's originally verified kernel was about 8,500 lines of C. — [Heiser, 2026 article](https://www.trustworthy.systems/publications/papers/Heiser_26.pdf)
- NOVA is a capability-based microhypervisor with user-level per-VM VMMs; BlueRock (formerly Bedrock Systems) has published a formal specification of it. — [FOSDEM 2025 paper](https://fosdem.org/2025/events/attachments/fosdem-2025-5083-a-formal-specification-of-the-nova-microhypervisor/slides/23385/paper_WDpfx6o.pdf)
- In AWS's Graviton5 design the Nitro Hypervisor still handles policy but is deprivileged; it must ask the Nitro Isolation Engine (Rust) to perform any operation touching guest state. — [AWS Compute Blog](https://aws.amazon.com/blogs/compute/aws-nitro-isolation-engine-formally-verifying-the-hypervisor-in-the-aws-nitro-system/)
- SeKVM retrofits KVM into a small verified core (KCore) plus untrusted services. — [Li et al., IEEE S&P 2021](https://www.cs.columbia.edu/~nieh/pubs/ieeesp2021_kvm.pdf)
- Xen's pygrub is now supported only in deprivileged mode (XSA-508), an example of toolstack deprivileging forced by advisories. — [Xen XSA index](https://xenbits.xen.org/xsa/)

### Inferences
- Target architecture for a new hypervisor: ring -1 / EL2 core with no device models, no instruction decoder (require EPT/NPT, unrestricted guest, and virtio-MMIO or a paravirtual transport so MMIO exits carry decoded information or can be bounced to the per-VM VMM), and no legacy platform devices (no PIC/PIT/RTC/floppy/VGA; use x2APIC, TSC-deadline timer, PVH-style direct boot).
- Hypercall surface should be small, versioned, capability-addressed (no global IDs or ambient authority), fixed-size arguments, no hypercall that takes guest pointers to complex structures, and every hypercall bounded or preemptible.
- Decoding of the faulting instruction, if ever needed, belongs in the deprivileged per-VM VMM, where a bug compromises only that VM.

### Gaps
- No current, like-for-like LoC table (Xen, KVM+QEMU, Hyper-V, Firecracker, Cloud Hypervisor, crosvm, NOVA, seL4, pKVM) was found from a single source; the NSDI 2020 figures are dated.
- Size of the Nitro Isolation Engine code base was not found.

## 3. Transient-execution and hardware side channels

### Takeaway
A new hypervisor must assume a steady stream of new cross-domain speculation bugs (at least one guest-to-host-relevant disclosure per year from 2018 to 2026). The durable defenses are structural: keep no other tenant's secrets mapped while a guest runs, flush predictors and buffers at trust-boundary crossings, and never co-schedule mutually distrusting vCPUs on SMT siblings.

### Cited Findings

**Attack record**
- L1TF (2018): a guest can read L1D contents of any physical address; mitigations are conditional or unconditional L1D flush on VM entry and SMT control. Linux documents that with SMT enabled a sibling guest can still attack. — [Linux kernel L1TF docs](https://docs.kernel.org/admin-guide/hw-vuln/l1tf.html)
- MDS/TAA: mitigated by `VERW` buffer clearing on transitions; full mitigation also requires disabling SMT on affected parts. — [Linux kernel MDS docs](https://docs.kernel.org/admin-guide/hw-vuln/mds.html)
- Core scheduling lets only mutually trusting tasks share a core, as an alternative to disabling SMT. — [Linux kernel core-scheduling docs](https://docs.kernel.org/admin-guide/hw-vuln/core-scheduling.html)
- Retbleed (CVE-2022-29900 / CVE-2022-29901): return instructions exploitable on AMD Zen 1/2 and Intel Skylake-era parts; the authors measured 14%–39% mitigation overhead. — [ETH COMSEC Retbleed](https://comsec.ethz.ch/research/microarch/retbleed/)
- Inception / SRSO (CVE-2023-20569) on AMD Zen. — [ETH COMSEC Inception](https://comsec.ethz.ch/research/microarch/inception/)
- Downfall / GDS (CVE-2022-40982): gather-instruction data sampling on Intel 6th–11th gen; microcode mitigation with up to 50% overhead on gather-heavy workloads. — [downfall.page](https://downfall.page/)
- Zenbleed (CVE-2023-20593): AMD Zen 2 vector-register leak across any boundary including VMs. — [Ormandy, Zenbleed](https://lock.cmpxchg8b.com/zenbleed.html)
- Reptar (CVE-2023-23583): Intel redundant-prefix microcode bug; a guest could crash the host (machine check) and possibly worse. — [Ormandy, Reptar](https://lock.cmpxchg8b.com/reptar.html)
- BHI (2022) and Native BHI (CVE-2024-2201) defeat eIBRS-only isolation via branch history. — [VUSec BHI](https://www.vusec.net/projects/bhi-spectre-bhb/); [VUSec Native BHI](https://www.vusec.net/projects/native-bhi/)
- GhostRace (CVE-2024-2193): speculative race conditions make synchronization primitives bypassable in speculation. — [VUSec GhostRace](https://www.vusec.net/projects/ghostrace/)
- Training Solo (May 2025): self-training Spectre-v2 attacks work even with domain isolation fully enabled. Two hardware issues, Indirect Target Selection (CVE-2024-28956) and a Lion Cove issue (CVE-2025-24495), break user and guest isolation; ITS also breaks hypervisor isolation. Mitigations: microcode for the IBPB bypass, relocating indirect branches out of the lower half of cache lines, BHB-clearing sequence on older CPUs. — [VUSec Training Solo](https://vusec.net/projects/training-solo); [Linux kernel ITS docs](https://docs.kernel.org/admin-guide/hw-vuln/indirect-target-selection.html)
- Xen's ITS response is XSA-469, with patches for 4.18–4.20. — [oss-security](https://openwall.com/lists/oss-security/2025/05/12/5)
- XSA-471 (8 July 2025, updated Nov 2025): AMD Transient Scheduler Attacks. — [XSA-471](https://xenbits.xen.org/xsa/advisory-471.html)
- VMScape (CVE-2025-40300, Sept 2025): a guest can influence branch prediction in the host's *userspace* VMM (notably QEMU), on AMD Zen (families 0x17, 0x19, 0x1a), Hygon and Intel parts. Linux's default mitigation issues an IBPB before the first return to userspace after a VM exit (`vmscape=ibpb`); overhead is largest for workloads that exit to userspace often. — [Linux kernel VMSCAPE docs](https://docs.kernel.org/admin-guide/hw-vuln/vmscape.html); [Intel](https://www.intel.com/content/www/us/en/developer/articles/news/more-information-vmscape.html); [NVD](https://nvd.nist.gov/vuln/detail/CVE-2025-40300)
- 2026 disclosures: XSA-488 (April 2026), transient sampling of floating-point divider state (low severity) — [Xen XSA index](https://xenbits.xen.org/xsa/); "Skyfall" (CVE-2026-40671, AMD-SB-3061), a transient attack on Zen 3/4 letting a malicious *host* read SEV-SNP guest data, decrypted microcode and SMM memory **[secondary]** — [SecurityOnline](https://securityonline.info/skyfall-attack-amd-sev-snp-cve-2026-40671/); CVE-2025-27572 (published Feb 2026), transient information exposure in Intel TDX — [NVD](https://nvd.nist.gov/vuln/detail/cve-2025-27572).

**Structural defenses**
- A "secret-free" hypervisor design implemented in Xen (Microsoft Research, 2022) removes other domains' secrets from the address space active while a guest runs; the authors report performance comparable to baseline and up to 37% improvement on some hypervisor paths versus Xen's default mitigations. **[dated; authors' own figures]** — [MSR paper](https://www.microsoft.com/en-us/research/uploads/prod/2022/07/sf-hypervisor.pdf)
- Linux: a `KVM_CREATE_GUEST_MEMFD` flag to remove guest memory from the host direct map (Roy v4, Feb 2025; Kalyazin v12, April 2026) was still under review in April 2026, with open questions about how KVM itself accesses unmapped guest memory. — [v4 cover letter](https://lkml.iu.edu/hypermail/linux/kernel/2502.2/09680.html); [v12 thread](https://lkml.iu.edu/2604.2/07630.html); LWN coverage: [1](https://lwn.net/Articles/1064090/), [2](https://lwn.net/Articles/1049461/)
- Developers on the list distinguish ASI (prevents only *speculative* access through the direct map) from direct-map removal (prevents architectural access too). — [lore thread](https://lore.kernel.org/all/CA+i-1C3DtXzzkatepVvn-E45Gyxb3YmYd-irxfjDL5bL5MhWVA@mail.gmail.com/T/)

**Rowhammer and dedup channels**
- Flip Feng Shui combined memory deduplication with Rowhammer to flip bits in a co-hosted victim VM's memory (e.g. corrupting RSA keys). **[dated, 2016]** — [Razavi et al., USENIX Security 2016](https://www.usenix.org/conference/usenixsecurity16/technical-sessions/presentation/razavi)

### Inferences
- Minimum mitigation set for a new x86 hypervisor: eIBRS/AutoIBRS plus BHI controls; IBPB on vCPU switch between security domains and before running the deprivileged VMM after a guest exit (VMScape); RSB/RAS stuffing on exit; `VERW` and (on L1TF parts) L1D flush on entry; STIBP or, better, core scheduling so SMT siblings only ever run vCPUs of one tenant (or SMT off); ITS-safe thunk alignment; microcode loading at boot with a minimum-revision check.
- A new hypervisor can be secret-free by construction far more cheaply than Linux can be retrofitted: no direct map of all RAM, per-vCPU hypervisor stacks and state mapped only while that vCPU runs, guest memory never mapped in the core. This is the single most valuable design decision against future speculation bugs, because it removes the secret instead of the gadget.
- VMScape is directly relevant to a per-VM deprivileged VMM design: the VMM process is a speculation target for its own guest. If each VMM holds only its own guest's secrets (no host credentials, no other tenants' data), the leak is self-contained and the IBPB can be skipped or made cheap.
- Disable page deduplication/KSM across tenants and avoid cross-tenant shared page cache for writable or secret-dependent content; shared read-only base images are a (weaker) timing channel only.
- Rowhammer: prefer server platforms with ECC and TRR/RFM, and avoid placing page tables or hypervisor metadata adjacent to guest-controlled memory (guard rows) where topology is known.
- CPU-bug classes like Zenbleed and Reptar need no gadget in the hypervisor; the only defense is prompt microcode, plus the ability to mask CPU features (e.g. hide AVX gather) per VM.

### Gaps
- No published performance numbers were found for ITS or VMScape mitigations; researchers call VMScape overhead negligible, the kernel commit warns of cost on exit-heavy workloads.
- Status of secret-free / ASI work in Xen 4.20–4.21 was not found.
- Whether KVM ASI or guest_memfd direct-map removal has merged as of November 2026 was not confirmed.
- No 2026 guest-to-host (as opposed to host-to-confidential-guest) transient-execution disclosure beyond XSA-488 was found; absence of results is not proof none exists.

## 4. Memory-safe implementation (Rust)

### Takeaway
Rust removes most spatial and temporal memory bugs from device models and control logic, but a bare-metal hypervisor's `unsafe` core (page tables, context switch, guest-memory access, MMIO) is irreducible, and guest memory is concurrently mutable shared memory that Rust's type system does not model. The 2023–2026 CVE record shows residual bugs in exactly those places plus logic bugs.

### Cited Findings
- An independent audit of rust-vmm components (X41 D-Sec via OSTIF, 2023) was broadly positive but stressed that the hypervisor beneath the crates is a separate concern needing its own audit. — [X41 rust-vmm audit](https://ostif.org/wp-content/uploads/2023/11/X41-rustvmm-final_release.pdf)
- `vm-memory` (guest-memory abstraction used by Firecracker, Cloud Hypervisor and others) had an unsoundness CVE (CVE-2023-41051). — [Wiz database entry](https://www.wiz.io/vulnerability-database/cve/cve-2023-41051)
- `kvm-ioctls` had an advisory (GHSA-3qx8-rv27-j6gp) for undefined behaviour in `VmFd::create_device` from violating Rust pointer rules. — [Wiz database entry](https://www.wiz.io/vulnerability-database/cve/ghsa-3qx8-rv27-j6gp)
- Firecracker's 2026 OOB write was in virtio-PCI queue reconfiguration after activation. — [AWS bulletin 2026-015](https://aws.amazon.com/security/security-bulletins/2026-015-aws)
- NCC Group's "Rustproofing Linux" series shows Rust does not prevent double-fetch/TOCTOU on memory shared with an untrusted party. — [NCC Group, part 4: shared memory](https://www.nccgroup.com/research/rustproofing-linux-part-44-shared-memory/)
- Microsoft's FORGE lab (9 Nov 2026) used agentic tools to find memory-safety bugs in Rust's own `std`/`core`. Root-cause patterns: panic safety (invariant broken, then a call that can panic), `unsafe` code trusting user-supplied safe trait implementations, and `unsafe` code trusting process-wide state. Examples include `BTreeMap::insert` unsoundness when a custom allocator unwinds and a new heap-overflow path in `join`. — [Microsoft Security Blog](https://microsoft.com/en-us/security/blog/2026/11/09/forge-lab-uses-agentic-tools-to-identify-memory-safety-issues-in-rust)
- An empirical study of Rust memory-safety bugs found all but one (a compiler bug) required `unsafe` code. **[dated, 2020]** — [Xu et al., arXiv 2003.03296](https://arxiv.org/pdf/2003.03296)
- AWS wrote the Nitro Isolation Engine in a restricted subset ("μRust") to make it tractable for proof. — [types-announce post](https://lists.seas.upenn.edu/pipermail/types-announce/2025/012222.html)
- Atmosphere is a full microkernel in Rust verified with Verus (SOSP 2025). — [Atmosphere paper](https://mars-research.github.io/doc/2025-sosp-atmo.pdf)
- X41 found security vulnerabilities and missing mitigations in the Rust unikernel RustyHermit, a reminder that Rust kernels still need classic hardening. — [X41 advisory](https://www.x41-dsec.de/news/rustyhermit-security-vulnerabilities-and-missing-mitigations/)

### Inferences
- Patterns for containing `unsafe`: confine it to a few small crates (arch, paging, guest-memory accessor) with `#![forbid(unsafe_code)]` everywhere else; treat guest memory as volatile bytes only (copy-in once, validate the copy, never hold references into guest RAM); make page-table and frame ownership typed (linear frame tokens); document each `unsafe` block's safety contract and check it with Miri/Kani where possible.
- Panic policy matters in a hypervisor: `panic=abort` in the core removes the unwind-safety bug class highlighted by FORGE, but then every guest-reachable panic is a host DoS. Guest-reachable paths need to be panic-free by construction (no indexing, no `unwrap`, checked arithmetic), enforced by lints and fuzzing. In per-VM VMMs a panic can simply kill that VM.
- Avoid custom allocators that can unwind, minimize generic `unsafe` code that calls back into trait implementations, and pin the toolchain so `std` soundness fixes are tracked deliberately.

### Gaps
- No published `unsafe` line counts or percentages were found for Hyperlight, OpenVMM/OpenHCL, Edera, crosvm, Cloud Hypervisor, pKVM Rust work, Android pVM firmware, Asterinas or Theseus in this pass. (Asterinas's "framekernel" claim of a small `unsafe` TCB is well known but no figure was verified here.)
- No security advisories specific to Hyperlight, OpenVMM or Edera were found; that may reflect search coverage, not absence.

## 5. Formal verification

### Takeaway
Full functional-correctness proofs of hypervisors exist but x86 virtualization remains the least-covered area. In 2026 the practical options for a small team are Verus-style proofs over a Rust core (about 1.5 person-years for a microkernel) and executable-specification runtime checking as pioneered for pKVM; AWS has shown production-scale proof of a Rust isolation kernel in Isabelle/HOL.

### Cited Findings
- seL4 verified configurations: 32-bit Arm (EL1) has functional correctness, integrity/availability, confidentiality and binary correctness; AArch64 with the kernel in EL2 (hypervisor mode) has functional correctness, integrity/availability and confidentiality; 32-bit Arm hyp has functional correctness only; RISC-V 64 has functional correctness, integrity and confidentiality; x86-64 has functional correctness only and **no VT-x coverage**. No verified configuration covers IOMMU/SMMU, boot, or MCS (in progress). — [seL4 verified configurations](https://docs.sel4.systems/projects/sel4/verified-configurations.html)
- seL4's proof base has grown past a million lines; the original proof was about 200,000 lines for about 8,500 lines of C. — [Heiser, 2026](https://www.trustworthy.systems/publications/papers/Heiser_26.pdf)
- SeKVM: machine-checked (Coq) proofs of a KVM retrofitted into a small trusted core, including multiprocessor support. **[dated, 2021]** — [Li et al., IEEE S&P 2021](https://www.cs.columbia.edu/~nieh/pubs/ieeesp2021_kvm.pdf); [Li et al., USENIX Security 2021](https://www.cs.columbia.edu/~rgu/publications/usenix-security21-li.pdf)
- pKVM (Cambridge/Google, SOSP 2025): an executable "test-oracle" specification checked at runtime instead of a full proof; it found a number of critical bugs. The authors note the hypervisor spec is entangled with the hardware architecture spec. — [Memarian et al., exec-spec paper](https://www.cl.cam.ac.uk/~pes20/Stuff/exec-spec-paper.pdf)
- Follow-on Cambridge work (PLDI 2026) adds separation-logic specifications incrementally to existing code and reuses them for runtime checking, property-based testing and proof. — [PLDI 2026 supplementary](https://www.cl.cam.ac.uk/~pes20/pldi2026-paper155-supplementary.pdf)
- Atmosphere (SOSP 2025): Rust microkernel verified with Verus; under 2.5 person-years total, about 1.5 on verification; proof-to-code ratio 3.32:1; verifies in under 20 seconds on a laptop. — [Atmosphere paper](https://mars-research.github.io/doc/2025-sosp-atmo.pdf)
- AWS Nitro Isolation Engine: Rust, verified in Isabelle/HOL; functional correctness of operations such as VM creation and memory mapping; concurrency primitives additionally checked with Iris and Verus; proofs assume the Rust compiler and hardware are correct. Proof size is reported as about 260,000 lines (Dec 2025 announcement) and 330,000 lines (later coverage). Generally available on Graviton5 instances. — [types-announce](https://lists.seas.upenn.edu/pipermail/types-announce/2025/012222.html); [AWS Compute Blog](https://aws.amazon.com/blogs/compute/aws-nitro-isolation-engine-formally-verifying-the-hypervisor-in-the-aws-nitro-system/); [The New Stack](https://thenewstack.io/graviton5-nitro-isolation-engine/); [ITP 2026 invited paper](https://drops.dagstuhl.de/entities/document/10.4230/LIPIcs.ITP.2026.1)
- NOVA has a published formal specification from BlueRock. — [FOSDEM 2025](https://fosdem.org/2025/events/attachments/fosdem-2025-5083-a-formal-specification-of-the-nova-microhypervisor/slides/23385/paper_WDpfx6o.pdf)

### Inferences
- A realistic 2026 plan for a small team: (1) design the core to be verifiable (small, sequential-per-CPU or big-lock, no dynamic allocation after boot in the core, Rust subset); (2) verify the highest-value invariants first with Verus — page-table/EPT code ("no frame is mapped into two tenants; hypervisor memory never guest-mapped"), frame allocator ownership, capability table; (3) use Kani bounded model checking on `unsafe` blocks and parsers (virtqueue descriptor walking); (4) adopt the pKVM executable-spec approach as a runtime oracle in debug/fuzz builds.
- Atmosphere's 3.3:1 ratio suggests a 5–10k-line core costs on the order of 15–35k lines of proof, feasible for 2–3 engineers over a year; the Nitro figure (hundreds of thousands of proof lines) is the cost of the interactive-prover route.
- Proofs do not cover speculation, hardware bugs, or (in seL4's case) IOMMU and boot, so verification complements rather than replaces sections 3 and 10.
- If the product needs verified isolation on day one, Arm (seL4 AArch64 EL2, Nitro on Graviton) is far ahead of x86.

### Gaps
- No sources were retrieved for CertiKOS, Hyperkernel, VeriSMo, Arm CCA RMM verification, Prusti or Asterinas OSTD verification in this pass; they are known prior art but uncited here.
- No public statement of the Nitro Isolation Engine's code size or the proof's coverage of interrupt/vCPU paths was found.
- No evidence was found of a verified x86 VT-x hypervisor in production.

## 6. Testing and fuzzing practice

### Takeaway
State of the art is snapshot-based fuzzing at the hardware virtualization interface (VM-exit level), which works on any hypervisor, plus conformance suites and executable-spec oracles. A new hypervisor should be built so its exit handlers and device models can be driven from a harness without real hardware.

### Cited Findings
- HyperPill snapshots a hypervisor while it handles a VM exit, moves it into an emulator, enumerates its input spaces from the hardware state, and fuzzes; it needs no hypervisor-specific hooks and works on closed-source targets. 26 new bugs in QEMU, Hyper-V and macOS Virtualization Framework. — [USENIX Security 2024](https://www.usenix.org/conference/usenixsecurity24/presentation/bulekov)
- Nyx: coverage-guided hypervisor fuzzing using fast snapshots and nested virtualization. — [Schumilo et al., USENIX Security 2021](https://www.usenix.org/conference/usenixsecurity21/presentation/schumilo)
- HyperCube: a custom minimal guest OS for high-throughput black-box hypervisor fuzzing. **[dated]** — [NDSS 2020](https://www.ndss-symposium.org/ndss-paper/hyper-cube-high-dimensional-hypervisor-fuzzing/)
- Morphuzz: generic virtual-device fuzzing for QEMU and others. — [Bulekov et al., USENIX Security 2022](https://www.usenix.org/conference/usenixsecurity22/presentation/bulekov)
- kvm-unit-tests: tiny guest kernels exercising CPU/virtualization features. — [kvm-unit-tests](https://gitlab.com/kvm-unit-tests/kvm-unit-tests)
- Xen Test Framework (XTF): microkernel-style tests, including per-XSA regression tests. — [XTF docs](https://xenbits.xen.org/docs/xtf/)
- syzkaller fuzzes KVM through its ioctl interface. — [syzkaller](https://github.com/google/syzkaller)
- pKVM's executable specification acts as a runtime test oracle and found critical bugs. — [Memarian et al., SOSP 2025](https://www.cl.cam.ac.uk/~pes20/Stuff/exec-spec-paper.pdf)
- A survey list of hypervisor-testing papers is maintained. — [Hypervisor Testing Survey](https://marklee131.github.io/Hypervisor-Testing-Survey/)
- Januscape survived 16 years in heavily fuzzed code; PwnEscape and the Firecracker PCI bug were found by AI-assisted research in 2026. **[secondary]** — [The Hacker News](https://thehackernews.com/2026/07/16-year-old-linux-kvm-flaw-lets-guest.html); [CSO Online](https://www.csoonline.com/article/4237999/kvm-escape-exposes-broader-attack-surface-in-linux-networking.html); [AWS bulletin 2026-015](https://aws.amazon.com/security/security-bulletins/2026-015-aws)

### Inferences
- Recommended stack: (1) unit and property tests with the core compiled as a host library ("hypervisor as a library" with a mocked CPU); (2) `cargo-fuzz`/libFuzzer harnesses on each exit handler, hypercall and virtqueue parser, with the executable spec as oracle; (3) run kvm-unit-tests-style and XTF-style guest tests as conformance; (4) HyperPill/Nyx-style full-system fuzzing under nested virtualization or an emulator; (5) differential testing of any CPU-state handling against Bochs/QEMU-TCG; (6) deterministic simulation of scheduling and interrupt interleavings for concurrency bugs; (7) a regression test per security bug.
- State-machine bugs (reconfigure-after-activate, as in CVE-2026-5747) need stateful, sequence-aware fuzzing, not single-message fuzzing.
- Budget for continuous AI-assisted auditing: 2026 disclosures indicate attackers and researchers both now use it.

### Gaps
- No sources were retrieved for IRIS, TDXfuzz, Truman (NDSS 2025) or deterministic-simulation-testing practice in hypervisors.
- No public data on CI practice for hypervisors on nested virtualization (flakiness, coverage) was found.

## 7. Robustness: fault containment, live update, DoS defenses

### Takeaway
Most historical hypervisor bugs are availability bugs, so bounded execution and per-tenant resource accounting are core security features. Linux now has mainline building blocks for kexec-based host live update (KHO in 6.16, LUO in 6.19), but preserving guest memory across it was still in RFC during 2026.

### Cited Findings
- 61.81% of Xen core vulnerabilities led to host DoS; causes included live lock (8.33%), infinite loop (6.25%), false `BUG_ON` (6.25%), resource exhaustion (4.17%) and deadlock (3.47%). The paper notes long non-preemptible operations under locks as a pattern, and that fault handlers often had to kill the whole host. **[dated]** — [Shi et al., NDSS 2017](https://dev.ndss-symposium.org/wp-content/uploads/2017/09/ndss2017_02A-4_Shi_paper.pdf)
- Xen supports live patching of the running hypervisor. — [Xen livepatch docs](https://xenbits.xen.org/docs/unstable/misc/livepatch.html)
- Kexec Handover (KHO) was added in Linux 6.16 and the Live Update Orchestrator (LUO) in 6.19. — [LPC 2025 LUO slides](https://lpc.events/event/19/contributions/2052/attachments/1729/4180/LPC'25%20Live%20Update%20Orchestrator.pdf)
- Linux 6.19 documents memfd preservation across kexec via LUO, with state handed over through KHO. — [kernel docs](https://docs.kernel.org/6.19/mm/memfd_preservation.html)
- guest_memfd preservation for live update was at RFC stage in May–July 2026, limited to fully shared, pre-faulted guest_memfd; private guest_memfd was future work. — [RFC v1](https://lkml.indiana.edu/2605.2/02485.html); [v4 docs patch](https://lkml.iu.edu/2607.3/07903.html)
- One operator reported patching all Proxmox clusters for Januscape without downtime, illustrating operational demand for live migration or live patching during hypervisor CVEs. **[secondary]** — [Claranet](https://www.claranet.com/ch-en/blog/januscape-cve-2026-53359-how-we-patched-all-our-proxmox-clusters-without-any-downtime-workshop/)
- Firecracker runs each VMM under a jailer (chroot, namespaces, cgroups, seccomp) so a crashed or compromised VMM affects one microVM; rate limiters exist on its network and block devices. **[dated, 2020]** — [Agache et al., NSDI 2020](https://www.usenix.org/conference/nsdi20/presentation/agache)

### Inferences
- Design rules: every hypercall and exit handler has a static bound or explicit preemption/continuation; no guest-reachable assertion crashes the host (kill the VM instead); all memory consumed on a VM's behalf (page tables, EPT, vCPU state, virtqueue shadow state) is charged to a fixed per-VM quota allocated at creation; interrupt and exit rates are throttled per VM; the scheduler enforces hard CPU caps.
- Per-VM deprivileged VMMs give crash containment for free: restart the VMM, or kill the sandbox. For short-lived agent sandboxes, "kill and recreate from snapshot" is an acceptable recovery policy, which removes the need for VMM state recovery.
- For agent sandboxes with lifetimes of minutes to hours, drain-and-reboot is a viable patching strategy and far simpler than live update; hypervisor live patching or kexec-style live update becomes necessary only for long-lived sandboxes. Fast boot plus snapshot/restore of guests to another host is the cheapest route.
- A hardware watchdog (and NMI watchdog for stuck cores) driven from the core, plus host-level fencing, is needed since a hung type-1 hypervisor has no OS underneath to recover it.

### Gaps
- No published data on hypervisor crash rates or on VMM fast-restart techniques in production was found.
- Current Xen live-update (as opposed to live-patch) status was not checked.
- No source was retrieved on interrupt-storm defenses specifically.

## 8. Agent-specific containment controls

### Takeaway
Documented 2025–2026 agent "sandbox escapes" mostly did not break the isolation mechanism; they abused what crosses the boundary (workspace files acted on by trusted host tools, credentials, network egress). VM isolation is necessary but the controls around the VM decide the outcome.

### Cited Findings
- Pillar Security (2026) crossed the security boundary of Cursor, OpenAI Codex, Gemini CLI and Antigravity without breaking the sandbox: a prompt-injected agent writes a file inside the workspace (e.g. project configuration) that a trusted tool outside the sandbox (Git integration, IDE extension, local daemon) later reads and acts on. Three vendors patched; Google downgraded its findings. — [BleepingComputer](https://www.bleepingcomputer.com/news/security/cursor-codex-gemini-cli-antigravity-hit-by-sandbox-escapes/); [The Next Web](https://thenextweb.com/news/ai-coding-agents-sandbox-escapes-pillar)
- Cymulate documented the same pattern ("Configuration-Based Sandbox Escape") across Claude Code, Gemini CLI and Codex CLI in April 2026. **[secondary, via TNW]** — [The Next Web](https://thenextweb.com/news/ai-coding-agents-sandbox-escapes-pillar)
- Docker describes a case where an approved, genuinely harmless command yielded arbitrary code execution because its meaning had been changed moments earlier by something the user never saw. — [Docker blog](https://www.docker.com/blog/coding-agent-horror-stories-the-command-you-already-approved/)
- A $1M challenge against Vercel's Firecracker-based sandbox produced a host-kernel networking exploit path through virtio-net; Vercel said no report showed access to customer files, secrets or application data. **[secondary]** — [CSO Online](https://www.csoonline.com/article/4237999/kvm-escape-exposes-broader-attack-surface-in-linux-networking.html)
- Vercel (Feb 2026): a secret-injection proxy sits outside the sandbox boundary and adds credentials to outbound requests; but if agent and generated code share one sandbox, generated code can still *use* credentials through the proxy, so they recommend separate security contexts for the agent harness and generated code. — [Vercel blog](https://vercel.com/blog/security-boundaries-in-agentic-architectures)
- Docker Sandboxes (microVMs): a host-side proxy overwrites auth headers; the sandbox sees only a sentinel value and the real credential never enters the VM. — [Docker docs](https://docs.docker.com/ai/sandboxes/security/credentials/)
- Cloudflare Sandboxes (April 2026): outbound handlers run outside the sandbox and provide credential injection, TLS interception and egress allow/deny lists. — [Cloudflare changelog](https://developers.cloudflare.com/changelog/post/2026-04-13-sandbox-outbound-workers-tls-auth/index.md)
- LangSmith (May 2026): an auth proxy injects headers at the network layer and can call back for credentials per destination host. — [LangChain blog](https://langchain.com/blog/how-auth-proxy-secures-network-access-for-langsmith-agent-sandboxes)

### Inferences
- The hypervisor/platform should provide these as first-class primitives, since every vendor has converged on them: default-deny egress with per-sandbox allowlists enforced outside the guest (ideally in a deprivileged per-tenant network component); an L7 egress proxy that injects credentials scoped per destination and per sandbox; no long-lived secrets in guest memory, images or snapshots; metadata-service access blocked.
- Guest I/O surface for agent sandboxes can be very small: virtio-block (read-only shared base image plus per-sandbox copy-on-write overlay), a vsock-like control channel, and either no NIC or a NIC whose only route is the proxy. Avoid virtio-fs/9p host file sharing (see CVE-2026-47243) — export workspace changes as diffs or tarballs that are treated as untrusted data.
- The Pillar/Cymulate pattern implies an "outbound artifact" rule: anything leaving the sandbox (files, git metadata, config, hooks) must never be executed or interpreted by trusted tooling without sanitization; the platform should offer a quarantined export path.
- Snapshot-based rollback, immutable base images, per-sandbox audit logs of all network flows and control-channel calls (recorded outside the guest), hard CPU/memory/disk/network rate limits, and short maximum lifetimes are cheap to provide at the hypervisor layer and limit both exfiltration and resource abuse (e.g. cryptomining, outbound attacks).
- Because a credential proxy can still be misused by code in the sandbox, policy should bind credentials to narrow scopes (method, path, repository) and rate-limit them; egress allowlists alone do not stop exfiltration to allowed multi-tenant services (e.g. a permitted code host).

### Gaps
- No confirmed real-world incident of data exfiltration from a hypervisor-isolated agent sandbox via a VM escape was found; reported cases are research demonstrations and disclosed vulnerabilities.
- Claims that Anthropic and OpenAI documented failed sandbox-escape attempts in internal red-teaming were seen only in a secondary arXiv review and could not be verified against primary sources.
- No primary source was retrieved for Anthropic, E2B, Modal or Fly credential-handling designs.

## 9. Confidential computing (SEV-SNP, TDX, Arm CCA)

### Takeaway
Confidential computing protects a guest from the host; the agent-sandbox threat is the reverse (protect host and other tenants from the guest), so it is largely orthogonal and can be deferred. Supporting it adds substantial complexity and its own steady flow of vulnerabilities.

### Cited Findings
- Skyfall (CVE-2026-40671, AMD-SB-3061; ETH Zurich and Google) lets a malicious host read SEV-SNP guest data on Zen 3/4 via transient execution; requires a privileged attacker; AMD shipped microcode. A second CVE (CVE-2026-18448) covers SMM memory leakage. **[secondary]** — [SecurityOnline](https://securityonline.info/skyfall-attack-amd-sev-snp-cve-2026-40671/)
- CVE-2025-27572 (published 10 Feb 2026): transient-execution information exposure affecting Intel TDX, requiring a privileged adversary. — [NVD](https://nvd.nist.gov/vuln/detail/cve-2025-27572)
- Linux guest_memfd, originally built for confidential guests, is the vehicle for removing guest memory from the host direct map for non-confidential guests too. — [v4 cover letter](https://lkml.iu.edu/hypermail/linux/kernel/2502.2/09680.html)
- AWS's approach to "operator cannot access guest" on Graviton5 is a verified isolation kernel beneath a deprivileged hypervisor, not CPU-vendor memory encryption alone. — [AWS Compute Blog](https://aws.amazon.com/blogs/compute/aws-nitro-isolation-engine-formally-verifying-the-hypervisor-in-the-aws-nitro-system/)

### Inferences
- For multi-tenant development sandboxes the provider is trusted by tenants, so SEV-SNP/TDX are not required for the core goal. They matter only if customers demand that the operator cannot read their source code or model traffic, or for compliance marketing.
- Cost of support for a new hypervisor: private/shared memory conversion, RMP or TDX-module integration, attestation plumbing, paravisor or `#VC`/`#VE` handling, bounce buffers for I/O, loss of memory overcommit/dedup/cheap snapshots, slower boot and restore — in tension with fast, snapshot-heavy sandbox lifecycles.
- The useful idea to borrow without the hardware is the design discipline: the hypervisor core never maps or reads guest memory, which also serves the secret-free goal of section 3. Keep the memory-management interfaces "confidential-ready" (explicit shared regions for virtqueues) so support can be added later.

### Gaps
- No quantitative data on engineering effort or runtime overhead of adding SEV-SNP/TDX/CCA to a small hypervisor was retrieved.
- Arm CCA RMM status and verification were not researched in this pass.

## 10. Measured/secure boot, attestation, and IOMMU/DMA protection

### Takeaway
Little primary-source material was gathered here; the main verified point is that even the best-verified kernels leave boot and IOMMU outside their proofs, so these need conventional engineering and testing.

### Cited Findings
- No seL4 verified configuration covers kernel startup at boot or device address translation (IOMMU / System MMU). — [seL4 verified configurations](https://docs.sel4.systems/projects/sel4/verified-configurations.html)
- Skyfall also exposed decrypted microcode updates and SMM memory, showing firmware-level secrets are in scope for transient attacks. **[secondary]** — [SecurityOnline](https://securityonline.info/skyfall-attack-amd-sev-snp-cve-2026-40671/)
- Hosts must be patched for host-side bugs regardless of guest state (PwnEscape advisory guidance), which presumes a way to prove which hypervisor build a host runs. **[secondary]** — [CSO Online](https://www.csoonline.com/article/4237999/kvm-escape-exposes-broader-attack-surface-in-linux-networking.html)

### Inferences
- Baseline for a new type-1 hypervisor: UEFI Secure Boot of a signed hypervisor image; TPM 2.0 measured boot (hypervisor, config, microcode revision, deprivileged service images extended into PCRs); remote attestation gating a host's admission to the fleet and its access to tenant secrets (sealed keys). DRTM (Intel TXT / AMD SKINIT) shrinks the trusted boot chain but adds platform-specific complexity; reasonable as a later phase.
- IOMMU (VT-d / AMD-Vi / SMMU) must be enabled before any device is initialized, with default-deny DMA, interrupt remapping on, and every passthrough or driver-domain device confined to its own domain. For agent sandboxes, avoid device passthrough to untrusted guests entirely; use the IOMMU to confine the hypervisor's own deprivileged driver components.
- Microcode loading and minimum-revision enforcement belong in early boot and in attestation policy, since many section-3 mitigations are microcode-delivered.

### Gaps
- No primary sources were retrieved on UEFI Secure Boot bypasses, TPM measured-boot practice for hypervisors, TrenchBoot/DRTM status in 2026, or IOMMU attack research (e.g. DMA attacks, interrupt-remapping bypasses). This section is mostly inference and needs dedicated research if it is to carry weight in the report.
