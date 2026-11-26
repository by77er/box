# Implementation path and tooling for a new lightweight type-1 hypervisor (state: Nov 2026)

Reader notes
- Research date: 2026-11-26. Items marked **[dated]** are older than about two years and may have changed.
- "Cited Findings" are things I read in a source during this session. "Inferences" are my engineering judgement or background knowledge that I did **not** verify against a source in this session; treat them as planning hypotheses.
- Several sources are search-result summaries of the primary page rather than a full read of it. Those are marked **(snippet)**. Numbers from them should be re-checked before they go into a decision document.

---

## 1. Language and toolchain (Rust no_std vs C, C++, Zig, Ada/SPARK)

### Takeaway
Rust `no_std` is the default choice of nearly every new hypervisor started since about 2018, and the bare-metal targets are stable enough to build on, but `build-std` is still not stabilized in 2026, so a custom-target or custom-`core` build still means a pinned nightly. The existing small type-1 hypervisors that are actually deployed (NOVA/Hedron in C++, ACRN/Bao/Xen in C, Muen in SPARK) are not Rust; the Rust ones are mostly research, embedded/ARM, or user-space VMMs on top of KVM.

### Cited Findings
- `build-std` is an accepted 2026 Rust project goal with implementation under way but **not stabilized**; the goal page names Cargo-team review bandwidth as the main risk to delivery. Design is split over RFCs 3873, 3874, 3875 with further RFCs not yet opened. No stabilization date found. — [Rust project goals 2026: build-std](https://goals.rust-lang.org/2026/build-std.html); [RFC 3873](https://rust-lang.github.io/rfcs/3873-build-std-context.html) (snippet)
- JSON custom target specs were gated in Rust 1.95 so that they stay unstable even if build-std is stabilized. Practical consequence: if you need a custom target JSON (e.g. to change code model, disable red zone, or soft-float differently from the built-in target), you stay on nightly. — [Rust 1.95 release notes mirror](https://overctrl-web.vercel.app/r/rust-lang/rust/releases/1.95.0) (snippet, secondary mirror)
- `x86_64-unknown-none` tier: the platform-support table and the compiler target spec list it as Tier 2 (no host tools), promoted in 2022; the target's own doc page still says "Tier: 3", which looks stale. Sources conflict; Tier 2 is the likelier truth. — [compiler-team issue 499](https://github.com/rust-lang/compiler-team/issues/499); [target page](https://doc.rust-lang.org/rustc/platform-support/x86_64-unknown-none.html) (snippet)
- The `limine` Rust crate (0.6.3, 2026) implements the Limine boot protocol base revision 6, including a multiprocessor (MP) request with per-arch code for x86_64, aarch64, riscv64, loongarch64. The API changed significantly between 0.5.x and 0.6.x. The Limine bootloader itself supports x86-64, aarch64, riscv64, loongarch64 (UEFI). — [limine crate 0.6.3](https://docs.rs/crate/limine/0.6.3); [Limine repo](https://github.com/Limine-Bootloader/Limine) (snippet)
- `uefi-rs` documents support for i686, x86_64 and aarch64 UEFI applications. — [uefi-rs](https://github.com/ben-krieger/uefi-rs) (snippet, via a fork's README)
- A solo-developer bare-metal ARM64 hypervisor (Hafnium SPMC replacement) reports 30K lines of `no_std` Rust, a 230 KB release binary, one named external dependency (`fdt`), `alloc` with a bump allocator, 34 test suites running in about 5 s under QEMU, and 35/35 end-to-end tests against real TF-A + pKVM. Reported pain: debug-mode codegen turned `read_volatile` into NEON instructions, which EL3 firmware trapped, giving a silent hang that took hours in GDB. Lessons: start with `opt-level = 1`, model lifecycle as enums, use enum dispatch instead of trait objects for MMIO routing, no `unwrap()`. Duration and unsafe ratio are not stated. — [users.rust-lang.org post](https://users.rust-lang.org/t/30k-lines-of-no-std-rust-a-bare-metal-arm64-hypervisor-that-replaces-googles-hafnium-spmc/139497)
- Hedron (Cyberus' NOVA derivative) builds with cmake ≥ 3.13 and gcc ≥ 10 or clang ≥ 12, recommends Nix, uses Catch2 unit tests, clang-format/clang-tidy and GitLab CI. The language is not named on the page; the toolchain implies C++. — [Hedron README](https://github.com/cyberus-technology/hedron)
- AxVisor (ArceOS-based) is a Rust type-1 hypervisor with one code base for x86_64, aarch64 and RISC-V, Apache-2.0. Guests: ArceOS, Starry-OS, NimbOS, Linux. Hardware listed: QEMU (ARM64, x86_64) and four ARM64 boards (Orange Pi 5 Plus, Phytium Pi, ROC-RK3568-PC, EVM3588). No real x86 hardware is listed. 118 commits, 47 stars, no releases at time of reading. x86 boot supports multiboot and UEFI paths. — [AxVisor repo](https://github.com/arceos-hypervisor/axvisor); [docs.rs axvisor](https://docs.rs/axvisor) (snippet)
- Rust-Shyper (Beihang) is an AArch64 embedded type-1 hypervisor in Rust with VM migration and hypervisor live update; primary reference is Mo et al., Journal of Systems Architecture 2023. No LOC or unsafe ratio retrieved. — [openEuler write-up](https://www.openeuler.org/en/blog/20230428-rust/20230428-rust) (snippet) **[dated]**
- Oxide rejected VirtualBox partly on language grounds: its "C++ codebase makes it unpalatable to work with", and they chose to write their VMM userspace (Propolis) in Rust. — [Oxide RFD 26](https://rfd.shared.oxide.computer/rfd/26)

### Inferences
- Practical 2026 Rust setup: stable toolchain with the prebuilt `x86_64-unknown-none` and `aarch64-unknown-none` (or `-softfloat`) targets if the built-in target settings are acceptable; pinned nightly plus `-Zbuild-std=core,alloc` only if you need a custom target or want `core` rebuilt with different flags. Plan for the nightly pin either way for things like naked functions on older compilers, custom test frameworks, and sanitizers.
- Likely crate set (none verified for 2026 versions in this session): `x86_64` and/or `x86` (control registers, paging structures, MSRs), `raw-cpuid`, `acpi` (table parsing only; no AML needed if you avoid devices that need `_PRT`/`_OSC` evaluation), `uefi` or `limine`, `bitflags`, `spin`, `linked_list_allocator` or `buddy_system_allocator`/`talc`, `fdt` for ARM64, `aarch64-cpu`, `arm-gic-driver` or equivalent, `log`, `heapless`, `zerocopy`/`bytemuck`. Expect to write VMX/SVM structures yourself: the public crates cover field encodings but not a production VMCS/VMCB layer.
- Zig has no deployed hypervisor that I know of and no relevant ecosystem (virtio, ACPI) to reuse; C++ is only attractive if you fork NOVA/Hedron; Ada/SPARK (Muen) fits static separation kernels, not a dynamic VM-churn workload. I found no 2026 source that changes this picture.
- The Rust advantage for this project is less about the ring-0 core (which is dominated by `unsafe` hardware manipulation) and more about the device-model/VMM layer, where the reusable code (rust-vmm, Cloud Hypervisor, OpenVMM) already is Rust.

### Gaps
- No verified 2026 version/maintenance status for `x86_64`, `x86`, `raw-cpuid`, `acpi`, `bootloader` crates.
- No primary source retrieved for Muen (SPARK) or Zig experience reports.
- No measured unsafe-code ratio for any Rust hypervisor.
- No source confirming whether Asterinas OSTD or Theseus are usable as a base for VMX root-mode code; not researched.

---

## 2. Reusable building blocks

### Takeaway
There is no drop-in `no_std` hypervisor-side device stack: rust-vmm is built around hosted Linux/Unix (mmap, ioctls, eventfd), so in ring 0 it is reference material, but it becomes directly reusable if device models run as ordinary Rust `std` programs in a service VM or in a hypervisor user mode with a small runtime. The cleanest reuse boundaries on offer are (a) a microhypervisor with a user-level VMM (NOVA/Hedron, seL4+libvmm) and (b) an existing VMM's hypervisor-abstraction layer (Cloud Hypervisor's KVM/MSHV split, OpenVMM's modular design).

### Cited Findings
- `vm-memory` README lists platform support as x86_64/ARM64/RISCV64 on Linux/Unix/Windows, with the default backend being "standard Unix memory mapping" and an alternative `xen` feature (Unix only; the crate is built for either mmap or Xen, not both). The README says nothing about `no_std`. — [rust-vmm/vm-memory](https://github.com/rust-vmm/vm-memory)
- `virtio-queue` (vm-virtio workspace) implements virtio 1.0+ queues and descriptor chains for the **device** side; legacy virtio is not supported. — [vm-virtio](https://github.com/lucido-simon/vm-virtio) (snippet, fork README)
- `virtio-drivers` (rcore-os, used in Android pKVM/Trusty) is `no_std` but is the **guest/driver** side (block, net, GPU, input, console): useful if the hypervisor itself runs nested on virtio during development, not for emulating devices to guests. — [virtio-drivers on docs.rs](https://docs.rs/crate/virtio-drivers/0.7.5) (snippet)
- rust-vmm's hypervisor wrappers are `kvm-ioctls`/`kvm-bindings` and `mshv-ioctls`/`mshv-bindings`; a FOSDEM 2026 talk covered ecosystem evolution and a move toward a monorepo. — [FOSDEM 2026 rust-vmm slides](https://fosdem.org/2026/events/attachments/WEHLEY-rust-vmm_evolution_on_ecosystem_and_monorepo/slides/266719/rust-vmm_q4zaofh.pdf) (snippet)
- Cloud Hypervisor runs on both KVM and Microsoft Hypervisor (MSHV) from a single binary (since v26), i.e. it already has a working hypervisor-abstraction seam; v52 (May 2026) added SEV-SNP guests on KVM. A third-party page estimates about 106K lines of Rust for Cloud Hypervisor vs about 83K for Firecracker (March 2026, method unverified). — [Phoronix on CH 26](https://www.phoronix.com/news/Cloud-Hypervisor-26.0); [Phoronix on CH 52](https://phoronix.com/news/cloud-hypervisor-52); [rywalker.com](https://rywalker.com/research/cloud-hypervisor) (snippets; LOC figure is low-confidence)
- NOVA: microhypervisor about 9 KLOC, thin user-level environment about 7 KLOC, user-level VMM about 20 KLOC, about 36 KLOC total; nearly all virtualization functionality lives in a deprivileged per-VM VMM so that a VMM bug compromises only that VM. — [NOVA FOSDEM 2013 slides](https://hypervisor.org/nova-fosdem-2013.pdf); [NOVA EuroSys 2010 paper](https://HyperVisor.org/eurosys2010.pdf); [survey arXiv 2112.06875](https://arxiv.org/pdf/2112.06875) (snippets) **[dated: 2010–2013 figures]**
- Hedron: GPLv2; derived from NOVA (last common commit 2015); capability-based; runs on x86 machines with ACPI, XSAVE, FSGSBASE; "AMD systems are currently not supported"; core of Cyberus' Secure Virtual Platform; GitHub is a mirror of an internal repo and "We are not actively monitoring PRs and issues on Github"; 1,637 commits. — [Hedron README](https://github.com/cyberus-technology/hedron)
- seL4 route: libvmm 0.2.0 was released 2026-08-19 and is stated to run on AArch64 and x86-64, but the libvmm README says x86 virtualization and other needed patches are not yet in mainline Microkit and points at a development SDK. Microkit 2.3.0 notes mention x86_64 IOMMU support, x86_64 VM improvements and calls to turn x86 vCPUs on/off. A 2025 list thread said x86 libvmm work was on hold over prerequisites and a multi-IOAPIC kernel issue. — [Trustworthy Systems virtualisation page](https://www.trustworthy.systems/projects/virtualisation/); [au-ts/libvmm](https://github.com/au-ts/libvmm); [Microkit 2.3.0 notes](https://docs.sel4.systems/releases/microkit/2.3.0); [seL4 devel list](https://lists.sel4.systems/hyperkitty/list/devel@sel4.systems/thread/FHQYU2A3RYSZKCJAR5EXEDLSAEJDF2VO/) (snippets)
- ACRN: hypervisor kept to about 40K LOC by design; most of the device model lives in a privileged Linux Service VM; a kernel module (HSM, upstream in Linux) exposes hypervisor services to the userspace device model. Early (2018) target was 25K LOC. — [ACRN introduction](https://projectacrn.github.io/3.3/_sources/introduction/index.rst.txt); [kernel.org ACRN docs](https://docs.kernel.org/virt/acrn/introduction.html); [The Register 2018](https://www.theregister.co.uk/2018/03/19/acrn_hypervizor_prject/) (snippets)
- Bao: clean-slate static partitioning hypervisor, cited as under 20K SLOC (second-hand figure); self-sufficient at boot, whereas Jailhouse needs a Linux root cell to boot and launch cells. — [GearV paper arXiv 2106.04514](https://arxiv.org/pdf/2106.04514.pdf); [Bao discussion 85](https://www.github.com/bao-project/bao-hypervisor/discussions/85) (snippets)
- OpenVMM: modular cross-platform Rust VMM from Microsoft; same repo hosts OpenHCL (paravisor, Linux kernel + user-mode VMM and drivers); development is "currently focused on its role in the OpenHCL paravisor"; repo layout `openvmm/`, `openhcl/`, `vm/` (devices, chipset), `support/`, `vmm_tests/` (petri framework). — [microsoft/openvmm](https://github.com/microsoft/OpenVMM); [OpenHCL announcement](https://techcommunity.microsoft.com/blog/windowsosplatform/openhcl-the-new-open-source-paravisor/4273172) (snippets)
- Hyperlight: Rust library for micro-VMs on KVM or Hyper-V with **no guest kernel or OS**; 1–2 ms cold spawn claimed (vendor), CNCF sandbox. A `hyperlight-unikraft` variant runs Linux applications on a Unikraft guest. Not a fit for full Linux dev sandboxes without that layer. — [Microsoft Open Source blog](https://opensource.microsoft.com/blog/2024/11/07/introducing-hyperlight-virtual-machine-based-security-for-functions-at-scale/); [hyperlight README](https://cdn.jsdelivr.net/gh/hyperlight-dev/hyperlight@main/README.md) (snippets)
- Edera/krata: uses "the core of the Xen hypervisor" with a memory-safe Rust control plane replacing the Xen toolstack; `krata-daemon` about 8K SLoC and `krata-guest` about 3K SLoC per lib.rs; x86_64 marked complete, aarch64 10–30% depending on listing. — [edera-dev/krata](https://www.github.com/edera-dev/krata); [lib.rs krata-daemon](https://lib.rs/crates/krata-daemon); [Edera paper arXiv 2501.04580](https://arxiv.org/html/2501.04580v2) (snippets; vendor-authored)

### Inferences
- Where rust-vmm is usable depends on where device models run, not on the crates themselves:
  - In ring 0 / `no_std`: not directly. Expect to re-implement the guest-memory abstraction (trivial when the hypervisor has a direct map) and port or rewrite the virtqueue code. `virtio-bindings`, `linux-loader` (boot-params/bzImage/ELF/PVH logic) and `acpi_tables` are mostly pure data manipulation and are the likeliest to port with modest patches, but I did not verify their `no_std` status.
  - In a Linux service VM: everything works unchanged, including `vhost`, `vhost-user-backend` and the `vhost-device` daemons.
  - In a hypervisor user-mode VMM process: usable only if you supply an mmap-like and eventfd-like shim, which is a non-trivial runtime of its own.
- The most leveraged reuse for a from-scratch core is to copy the *architecture* of NOVA/ACRN (tiny ring-0 core, per-VM deprivileged VMM) and take *code* from Cloud Hypervisor/Firecracker for the device model by adding a third backend next to KVM and MSHV. Cloud Hypervisor's existing two-backend split is the evidence that this seam is real.
- Hedron is a weaker base than its heritage suggests for commodity servers: Intel-only, GPLv2, and effectively a code drop without a public community.
- seL4 on x86 servers is the least mature option: x86 VM support still needs a development SDK, and seL4's formal verification does not cover the x86 VT-x configuration (background knowledge, not verified this session).

### Gaps
- `no_std` status of `virtio-queue`, `linux-loader`, `vm-superio`, `vm-allocator`, `acpi_tables`, `virtio-bindings`: not confirmed either way; check each `Cargo.toml`.
- OpenVMM crate-level reusability, LOC, and whether its virtio/VMBus/NVMe user-mode drivers build outside Linux/Windows: not found.
- No information gathered on crosvm, Hafnium, Bareflank/MicroV, Barbervisor, Hypatia, Theseus, uhyve, Syzygy, Bedrock Systems, memN0ps projects. Bareflank/MicroV is, to my knowledge, no longer actively developed, but I did not verify this.
- NOVA upstream (Udo Steinberg / BedRock) current feature state and licence terms not checked.

---

## 3. Boot and platform bring-up: ordered milestones and pitfalls

### Takeaway
The sequence is well understood and the first Linux guest boot is a matter of months for experienced people; the open-ended work starts after that (interrupt virtualization correctness, SMP guests, IOMMU, real-hardware quirks). Develop the VMX path under Bochs and nested KVM first, because both tell you *why* a VM entry failed, which hardware does not.

### Cited Findings
- VM-entry failure on bad guest state is reported as a VM exit with basic reason 33 (0x21), not as VMfailValid, and no guest state is saved, so the VMCS guest area does not reflect the failing state. Satoshi Tanda publishes a checker that simulates the SDM guest-state checks to find the offending field. — [Tanda gist (fork)](https://gist.github.com/4144/a849ed44a08bc511a59ffa661d70cee3); [Intel SDM Vol. 3C](https://cdrdv2-public.intel.com/671506/326019-sdm-vol-3c.pdf) (snippets)
- Common VM-instruction error numbers: 7 = invalid control fields, 8 = invalid host-state fields, 4/5 = VMLAUNCH on non-clear VMCS / VMRESUME on non-launched VMCS; VMfailInvalid = bad VMCS pointer or VMX not enabled. — [VMCS by practice (2026 blog)](https://youssix.github.io/2026/05/20/vmcs-by-practice/) (snippet)
- Bochs prints the reason for a failed VMX instruction (example log line: "VMFAIL: VMCS host state invalid CR0 0x00000000") and has a built-in debugger; Tanda's "Hypervisor 101 in Rust" course uses it for exactly this. — [Hypervisor-101-in-Rust: Testing with Bochs](https://tandasat.github.io/Hypervisor-101-in-Rust/hypervisor-setup-and-operation-cycle/testing-with-bochs.html) (snippet)
- AxVisor's x86 path supports both multiboot and UEFI boot protocols, which shows both are workable entry points for a Rust hypervisor. — [docs.rs axvisor](https://docs.rs/axvisor) (snippet)
- The Limine protocol hands the kernel an MP (SMP) response on x86_64 and aarch64, i.e. AP startup can be delegated to the bootloader early on. — [limine crate 0.6.3 MP module](https://docs.rs/crate/limine/0.6.3/source/src/mp/mod.rs) (snippet)
- Hedron's hardware floor (ACPI, XSAVE, FSGSBASE, Intel only) is an example of how deployed microhypervisors cut scope by requiring modern CPUs. bhyve similarly "was designed to require hardware virtualisation features (VT-x, EPT)" and so avoids shadow paging and other legacy paths, which Oxide cites as why it is "much smaller and more manageable than KVM". — [Hedron README](https://github.com/cyberus-technology/hedron); [Oxide RFD 26](https://rfd.shared.oxide.computer/rfd/26)

### Inferences
Suggested milestone order (background engineering knowledge; not verified against sources this session). Rough effort is for 2–3 experienced engineers and is my estimate only.

1. **M0 – Boots and prints (1–2 weeks).** Limine or a `uefi-rs` stub loader; 16550 serial; panic handler; GDT/IDT/TSS with IST stacks for NMI, #DF, #MC; early page tables with a physical direct map; frame allocator from the UEFI/Limine memory map.
   - Pitfalls: exiting boot services with a stale memory-map key; forgetting to keep ACPI reclaim and runtime regions mapped; red zone and SSE in interrupt paths (use the `-none` target defaults); serial port not at 0x3F8 on servers (parse SPCR, or use the BMC's serial-over-LAN).
2. **M1 – Platform discovery (2–4 weeks).** ACPI: RSDP → XSDT → MADT (LAPIC/x2APIC/IOAPIC/overrides/NMI), MCFG (ECAM), DMAR or IVRS, HPET, SRAT/SLIT. x2APIC mode; calibrate TSC (CPUID 0x15/0x16, fall back to HPET/PIT); TSC-deadline timer. Avoid an AML interpreter by not needing `_PRT` (use MSI/MSI-X only) and not doing power management beyond `hlt`/`mwait`.
   - Pitfalls: firmware hands over in x2APIC mode with more than 255 CPUs and you cannot go back to xAPIC; IOAPIC redirection for legacy IRQs if you ever need the UART interrupt; invariant TSC assumptions; TSC_ADJUST across sockets.
3. **M2 – SMP (2–3 weeks).** AP startup (via Limine MP, or INIT-SIPI-SIPI with a real-mode trampoline below 1 MiB), per-CPU data via GS base, IPIs, TLB shootdown, per-CPU idle.
   - Pitfalls: trampoline page tables must be reachable from 32-bit code; races on shared boot stack; microcode not loaded on APs; NMI during `swapgs`-style windows if you ever have a user mode.
4. **M3 – First guest instruction (2–4 weeks).** CPUID/`IA32_FEATURE_CONTROL` checks; CR4.VMXE; VMXON region with revision ID; VMCS allocation; controls derived from the `IA32_VMX_*` capability MSRs (allowed-0/allowed-1, use the TRUE variants); host state; minimal guest state; VMLAUNCH; exit stub that saves GPRs. On AMD: EFER.SVME, VMCB, host save area, VMRUN, plus `VMSAVE`/`VMLOAD` for hidden state.
   - Pitfalls: this is where most first attempts stall. Guest-state check failures (segment access-rights "unusable" bit, CR0/CR4 fixed bits, RFLAGS bit 1, activity state, `IA32_EFER` vs entry controls, PDPTEs for PAE guests). Host RSP/RIP not rewritten per CPU; host FS/GS/TR base wrong on the CPU that did not create the VMCS; forgetting VMCLEAR before migrating a VMCS between CPUs; NMIs arriving in root mode with a guest GS base.
5. **M4 – Memory virtualization (2–3 weeks).** EPT/NPT with 4K/2M/1G mappings, EPT-violation handler, VPID/ASID, INVEPT/INVVPID, unrestricted guest. MTRR/PAT handling for memory types.
   - Pitfalls: EPT misconfiguration from reserved bits or write-only entries; memory-type bits in leaf entries; stale TLB without VPID invalidation; splitting large pages under a running vCPU.
6. **M5 – Linux boots to an initramfs shell, 1 vCPU (4–8 weeks).** Direct kernel boot (bzImage 64-bit protocol or PVH entry) with `boot_params`, e820, command line, initrd; CPUID filtering (hide VMX, PMU, features you do not context-switch; set the hypervisor bit and a leaf range at 0x4000_0000); MSR bitmaps and emulation of the dozen or so MSRs Linux insists on; XSETBV and XSAVE area sizing per guest XCR0; CR0/CR4 guest/host masks and read shadows; HLT exiting; I/O bitmap with an emulated 16550; a clock source (pass TSC through with offsetting, expose kvmclock or Hyper-V reference TSC page so Linux does not need PIT/HPET calibration).
   - Pitfalls: Linux falling back to PIT calibration and hanging; missing `IA32_APIC_BASE`/x2APIC MSR range handling; guest enabling features the CPUID promised but the host has not enabled in XCR0; AVX-512/AMX state size; `MSR_IA32_SPEC_CTRL`, `PRED_CMD`, `FLUSH_CMD` passthrough and the speculation mitigations on VM exit (retpoline/IBRS/eIBRS, L1D flush, VERW for MDS, RSB stuffing).
7. **M6 – Interrupt virtualization (4–8 weeks, then a long tail).** External-interrupt exiting; event injection with the interruptibility-state rules; NMI and interrupt-window exiting; virtual LAPIC timer. Prefer hardware help from day one: APICv (virtual-interrupt delivery, APIC-register virtualization, posted interrupts) on Intel, AVIC/x2AVIC on AMD, with x2APIC-only guests and MSI-only devices so no virtual IOAPIC/PIC/PIT is needed (the Firecracker/microvm style minimal platform).
   - Pitfalls: lost or double-injected interrupts around `IDT-vectoring info` on exits that interrupt delivery; EOI handling; priority (PPR/TPR) bugs that only show under load; timer drift when a vCPU is descheduled.
8. **M7 – SMP guests (3–6 weeks).** INIT/SIPI emulation or a paravirtual AP start; IPIs via x2APIC ICR MSR (fast-path this exit); wait-for-SIPI activity state; pause-loop exiting; per-vCPU TSC offset; PV spinlock/IPI/TLB-flush hints (KVM or Hyper-V enlightenment ABIs so stock Linux uses them).
9. **M8 – virtio devices (6–12 weeks).** virtio-mmio or virtio-pci (needs a minimal ECAM host bridge), console, block, net, rng, balloon/free-page-reporting, virtio-fs or 9p for workspace sharing; MMIO instruction decode avoided by using EPT-misconfig "fast MMIO" or hypercall/ioeventfd-style notification.
10. **M9 – IOMMU and passthrough (6–12 weeks).** VT-d/AMD-Vi: DMA remapping for all devices from boot (default-deny), interrupt remapping (required for x2APIC with many CPUs and for safe passthrough), RMRR/unity-map handling, MSI-X table trapping, PCI config-space filtering, device reset (FLR), ACS and isolation groups.
    - Pitfalls: RMRRs that make devices unassignable; firmware leaving DMA enabled across handoff; devices behind non-ACS bridges; SR-IOV VF enumeration requiring the PF driver (which lives in whoever owns the PF).
11. **M10 – Production hardening (open-ended; this is most of the project).** Machine check and NMI handling, microcode loading, CPU hotplug/offline, memory error containment, live update or kexec of the hypervisor, crash dumps, watchdogs, performance counters, ARM64 port (EL2, stage-2, GICv3/v4 with ITS, SMMUv3, PSCI, generic timers; a different platform layer but the same upper layers).

ARM64 note: bring-up is simpler per feature (no real mode, no VMCS consistency checks, standard PSCI/GIC/SMMU), but server platform variance (ACPI vs DT, SMMU quirks, GIC ITS for MSIs, RAS via firmware-first) replaces x86's legacy as the main cost.

### Gaps
- I did not retrieve Karvandi's later "Hypervisor From Scratch" parts, the memN0ps or Gamozo material, or OSDev wiki pages in this session, so the pitfall list above is uncited engineering knowledge.
- No sourced data on how long each milestone took real teams.

---

## 4. The host I/O problem (NIC and NVMe without writing drivers)

### Takeaway
Every small hypervisor that reached production avoided native drivers: Xen (dom0), ACRN (Service VM), Hyper-V/OpenHCL (root partition or Linux-based paravisor), Jailhouse (Linux root cell), and AWS Nitro (hardware offload cards). A privileged or semi-privileged Linux driver VM with device passthrough, serving virtio/vhost-user backends to sandboxes, is the only strategy with broad evidence behind it; SR-IOV VF passthrough is the performance escape hatch but costs density, fast boot and memory overcommit.

### Cited Findings
- ACRN keeps the hypervisor small by putting "the bulk of the Device Model implementation" in a Linux Service VM, which manages user VMs and does I/O emulation; an upstream Linux kernel module (HSM) is the interface. — [ACRN introduction](https://projectacrn.github.io/3.3/_sources/introduction/index.rst.txt); [kernel.org ACRN docs](https://docs.kernel.org/virt/acrn/introduction.html) (snippets)
- OpenHCL takes the same approach inside a paravisor: "a Linux-based kernel layer hosts the VMM and device drivers in usermode". — [OpenHCL announcement](https://techcommunity.microsoft.com/blog/windowsosplatform/openhcl-the-new-open-source-paravisor/4273172) (snippet)
- AWS's answer was to offload I/O to hardware: once networking and storage moved to Nitro cards, the hypervisor "is really only responsible for dividing up memory and dividing the cores among the guests"; the Nitro hypervisor is a "custom-developed, minimized hypervisor based on KVM". — [Cadence write-up of Liguori's Hot Chips talk](https://community.cadence.com/cadence_blogs_8/b/breakfast-bytes/posts/the-aws-nitro-project); [AWS Nitro security whitepaper](https://docs.aws.amazon.com/whitepapers/latest/security-design-of-aws-nitro-system/the-nitro-system-journey.html) (snippets)
- Xen driver-domain cost, historical: Santos et al. (USENIX ATC 2008) profiled per-packet cost in the driver-domain model and reduced receive-path cost by 56%, approaching direct I/O with multi-queue NICs. — [Santos et al., USENIX 2008](https://www.usenix.org/legacy/events/usenix08/tech/full_papers/santos/santos_html/) (snippet) **[dated]**
- Xen's network guide lists a netback thread pinned near 100% of a dom0 vCPU as the usual bottleneck symptom, with more dom0 vCPUs/netback threads and SR-IOV as remedies. — [Xen Network Throughput Guide](https://wiki.Xen.org/wiki/Network_Throughput_Guide) (snippet) **[dated]**
- Anecdote (xen-devel 2020): PV guest via dom0 backend reached about 9.39 Gbit/s on a 10 GbE NIC; the same through a passthrough driver domain reached only 2.4–4.5 Gbit/s, with the setup questioned by a maintainer. Not a controlled result. — [xen-devel thread](https://lore.kernel.org/xen-devel/a06fdec5-9e9e-2319-e7f7-d68fdb48ffba@suse.com/) (snippet)
- SR-IOV reached 10 GbE line rate for TX and RX in 2009 Xen Summit slides, 3.04x over PV in one workload. — [Xen Summit 2009 slides](https://pt.slideshare.net/xen_com_mgr/xensummit2009-io-virtualization-performance) (snippet) **[dated]**
- Xen dom0less/Hyperlaunch: the design doc describes extending dom0less (multiple domains constructed at boot) to x86 as Hyperlaunch. An April 2026 xen-users post reports conflicting information on whether x86 Hyperlaunch is usable in 4.20. Xen 4.21 notes mention virtio-pci and parallel boot for dom0less (Arm-oriented); Xen 4.22 (2026-07-30) x86 notes mention FRED but nothing found on x86 dom0less. — [Hyperlaunch design doc](https://xenbits.xen.org/docs/latest/designs/launch/hyperlaunch.html); [xen-users April 2026](https://lists.xen.org/archives/html/xen-users/2026-04/msg00016.html); [Xen 4.21 release](https://www.linuxfoundation.org/press/xen-project-delivers-xen-4.21-a-modernized-hypervisor-with-broader-architecture-support-and-improved-performance); [Xen 4.22 release](https://www.linuxfoundation.org/press/xen-project-releases-xen-4.22-strengthening-open-source-virtualization-for-cloud-embedded-and-automotive-systems-1) (snippets)
- Jailhouse requires a Linux root cell to boot the machine and launch other cells; Bao boots standalone but is static partitioning with passthrough only. — [Bao discussion 85](https://www.github.com/bao-project/bao-hypervisor/discussions/85) (snippet)

### Inferences
Options ranked by fit for "many small, short-lived Linux sandboxes" (my assessment):

| Strategy | Hypervisor work | I/O performance | Density / boot time | Main risk |
|---|---|---|---|---|
| A. Linux driver/service VM with NIC+NVMe passthrough; virtio backends in that VM (vhost-user style over a hypervisor-provided shared-memory + event channel transport) | IOMMU, passthrough, an inter-VM shared-memory and notification primitive, and a virtio transport that lets a backend in one VM serve another | Good: comparable to vhost-user designs, one extra world switch per notification unless polled | Best: sandboxes see only virtio, boot fast, memory can be shared/ballooned | Service VM is a large TCB slice and a single point of failure unless split (net VM, storage VM) and made restartable |
| B. SR-IOV VF per sandbox (NIC VFs; NVMe namespaces or SR-IOV NVMe) | Same passthrough plumbing; still need something to own the PF | Near native | Poor: VF count limits (tens to a few hundred per port), guest memory must be pinned so no overcommit or CoW snapshots, slower create/destroy, guest needs vendor VF driver | Kills snapshot/restore and memory density, which are the main levers for agent sandboxes |
| C. One or two native drivers in the hypervisor or a Rust driver process (e.g. NVMe plus one NIC family) | NVMe is a tractable spec-driven driver; a modern NIC (ice, mlx5) is large, firmware-coupled, and still needs a network stack/switch behind it | Potentially best | Good | Hardware lock-in; years of quirk-chasing; you still need switching, overlay, firewalling that Linux gives for free |
| D. Hardware offload (DPU/SmartNIC presenting virtio-net/NVMe functions) | Passthrough only | Near native | Good if the DPU supports many functions (scalable IOV / SIOV-like) | Cost and vendor dependency; not "commodity" |

- Option A is effectively what Xen dom0, ACRN and Hyper-V do. The new design work is making the service VM *less* privileged than dom0: it should own devices and serve backends but not be able to map arbitrary guest memory (grant-style or pre-shared virtqueue regions, or virtio with restricted DMA/swiotlb in the guest).
- A cheap first version of A: service VM gets the NIC and NVMe via passthrough; each sandbox gets virtio-net and virtio-blk whose backends are vhost-user daemons from rust-vmm's `vhost-device`/Cloud Hypervisor code in the service VM. The hypervisor only needs memory sharing and doorbells.
- When developing nested on KVM, the hypervisor needs no native drivers at all if the service VM uses virtio from L0; the `virtio-drivers` crate covers the case where the hypervisor itself wants a console or block device early.

### Gaps
- No recent (post-2020) controlled measurement of driver-domain vs in-host vhost vs VF passthrough overhead was found; the Xen numbers are 2008–2020.
- No source located quantifying the size of a minimal NVMe or NIC driver in Rust for hypervisor use.
- Status of x86 dom0less/Hyperlaunch in Xen 4.22 is unresolved.
- Hyper-V root-partition/direct-virtualization details and their performance were not researched.

---

## 5. Scheduler and resource management for many small VMs

### Takeaway
I found little sourced evidence specific to "thousands of small VMs"; what is documented is that SMT-safe core scheduling exists in Xen with small measured overhead in its author's tests but workload-dependent regressions elsewhere. The scheduler and memory manager are areas where a from-scratch hypervisor has to re-earn years of Linux tuning, which is a core argument for the KVM-based alternative.

### Cited Findings
- Xen core scheduling: boot option `sched-gran=core` (or `socket`) schedules sibling threads as a unit so that two domains never share a core, motivated by L1TF/MDS cross-thread leaks. — [Xen core scheduling patch series](https://patchew.org/Xen/20190528103313.1343-1-jgross@suse.com/); [Xen Summit 2019 slides](https://de.slideshare.net/xen_com_mgr/xpdds19-core-scheduling-in-xen-jrgen-gro-suse) (snippets) **[dated: 2019]**
- Author's credit2 measurements for one benchmark: 116.10 (cpu granularity) vs 114.04 (core) with no competing guest; 202.30 (cpu) vs 207.24 (core) with a competing 4-vCPU guest, i.e. within a few percent. A Phoronix comparison on a NUMA host found core granularity slower than both thread granularity and SMT-off in several tests. Runtime SMT switching is disabled when core scheduling is active. — [patch cover letter](https://lore.kernel.org/r/20190809145833.1020-1-jgross@suse.com); [openbenchmarking result](https://openbenchmarking.org/result/1904105-SP-1904100DA38&grr&sro) (snippets) **[dated]**
- Credit2 has been considered fully hyperthread-aware since Xen 4.8. — [Xen wiki: Hyperthreading](https://wiki.Xen.org/wiki/Hyperthreading) (snippet) **[dated]**
- AWS Nitro's hypervisor role is described as only dividing memory and cores among guests, i.e. static pinning rather than time-sharing for mainline EC2. — [Cadence write-up](https://community.cadence.com/cadence_blogs_8/b/breakfast-bytes/posts/the-aws-nitro-project) (snippet)
- Firecracker deliberately left scheduling, memory management and block I/O to the Linux kernel by keeping KVM and replacing only QEMU. — [Firecracker NSDI'20](https://www.usenix.org/conference/nsdi20/presentation/agache); [Brooker blog](https://brooker.co.za/blog/2020/02/19/firecracker.html) (snippets)

### Inferences
- A reasonable v1 design for agent sandboxes (bursty, mostly idle, 1–4 vCPUs, oversubscribed): per-CPU run queues with work stealing; a virtual-runtime fair scheduler with weights and caps (credit2/CFS/EEVDF-like) rather than EDF; tickless idle using TSC-deadline; a short default timeslice with wake-up preemption for I/O-bound vCPUs; halt-polling for a few tens of microseconds before descheduling; core scheduling keyed by tenant/trust domain so SMT can stay on.
- Memory: buddy allocator over 4K/2M/1G with per-CPU free-page caches; guest RAM backed by 2M pages where possible with the ability to split for CoW and ballooning; NUMA-local allocation with vCPU soft affinity. The features that matter most for sandbox economics are not in a "minimal" hypervisor and each is substantial: snapshot/restore with CoW from a shared base image, page sharing of identical kernel/rootfs pages, free-page reporting/ballooning, and memory hot-(un)plug (virtio-mem). Linux/KVM gets these from the host mm (userfaultfd, KSM, THP, memfd); a new hypervisor must build them.
- Real-time-oriented schedulers (seL4 MCS, NOVA's priority round-robin with donation) are aimed at latency isolation and are not obviously suited to heavy oversubscription.

### Gaps
- No sourced comparison of credit2 vs CFS/EEVDF vs BVT for high-density microVM hosts.
- No sourced data on memory-allocator design in NOVA, seL4, or Xen for many small VMs; not researched this session.
- No data on NUMA behaviour of small hypervisors.

---

## 6. Control plane and API design

### Takeaway
Deployed designs converge on a tiny hypercall or capability interface in the hypervisor plus a rich API in a less-trusted place: per-VM deprivileged VMMs (NOVA), a Service VM with a kernel shim and userspace device model (ACRN), or a daemon in the control domain (Edera's Rust replacement for the Xen toolstack). Firecracker's per-VM process with a REST socket is the model for the user-facing API.

### Cited Findings
- NOVA runs one deprivileged user-level VMM per VM, so "exploitation of a bug in the platform virtualization code only compromises the VMM"; Hedron exposes a capability-based authorization model. — [NOVA EuroSys 2010](https://HyperVisor.org/eurosys2010.pdf); [Hedron README](https://github.com/cyberus-technology/hedron) (first is snippet)
- ACRN: userspace device model in the Service VM talks to the hypervisor through the HSM kernel module (ioctl → hypercall). — [kernel.org ACRN docs](https://docs.kernel.org/virt/acrn/introduction.html) (snippet)
- Edera/krata: a Rust daemon (about 8K SLoC) implements the Xen control plane and manages OCI-container-style workloads in VMs. — [krata repo](https://www.github.com/edera-dev/krata); [lib.rs](https://lib.rs/crates/krata-daemon) (snippets)
- Firecracker built "a new VMM, device model, and API for managing and configuring MicroVMs" on KVM. — [Firecracker NSDI'20](https://www.usenix.org/conference/nsdi20/presentation/agache) (snippet)
- OpenVMM is designed around an internal message-passing architecture so the same VMM can be embedded in different environments, and runs as multiple user-mode processes in OpenHCL. — [OpenVMM project post](https://techcommunity.microsoft.com/blog/WindowsOSPlatform/the-openvmm-project/4547237) (snippet)

### Inferences
- Recommended split:
  1. Hypervisor: a small, versioned hypercall ABI (create/destroy VM, add vCPU, map/unmap/share memory, set vCPU state, run/exit reasons, inject interrupt, assign device, scheduling parameters), authorized by per-VM handles or capabilities rather than "caller is dom0". Shape it like the KVM ioctl surface or MSHV's so existing VMM code maps onto it.
  2. Per-sandbox VMM: either a process in the service VM (reuses Linux and rust-vmm unchanged; quickest) or a ring-3 process on the hypervisor (better isolation, NOVA-style; requires a user-mode runtime, IPC, and a loader, which is effectively a microkernel).
  3. Fleet API: REST/gRPC daemon in the service VM, Firecracker-like (create from snapshot, pause/resume, exec/attach, network policy).
- Starting with VMMs in the service VM and exposing a `/dev/<hv>` device with KVM-like ioctls (as ACRN's HSM and Microsoft's `/dev/mshv` do) is the lowest-effort path to reusing Cloud Hypervisor or Firecracker device models.

### Gaps
- No primary read of the MSHV root-partition ioctl ABI, Xen's domctl/XSM model, or seL4 Microkit's system description for comparison.
- No evidence gathered on control-plane latency (VM create/destroy rates) for these designs.

---

## 7. Development workflow, CI, and test reuse

### Takeaway
Day-to-day development can be done entirely in emulation (QEMU/KVM with nested VMX/SVM, Bochs for VM-entry diagnostics), and CI with KVM is now easy on hosted runners and non-metal cloud instances. Real hardware is still required for IOMMU, interrupt remapping, APICv/AVIC, SR-IOV, firmware quirks and performance, so a small physical lab or rented bare metal is needed from roughly milestone M9.

### Cited Findings
- GitHub-hosted Linux runners expose hardware acceleration (KVM): announced for larger runners in Feb 2023 and extended to standard 2-vCPU runners in April 2024. GitHub's docs still say nested virtualization is "technically possible" but "not officially supported". `/dev/kvm` permissions may need a udev/chmod step. — [GitHub changelog 2024-04-02](https://github.blog/changelog/2024-04-02-github-actions-hardware-accelerated-android-virtualization-now-available/); [GitHub-hosted runners docs](https://docs.github.com/en/actions/concepts/runners/github-hosted-runners); [WarpBuild docs](https://warpbuild.com/docs/ci/nested-virtualization) (snippets)
- Third-party runner services (WarpBuild, RunsOn, Namespace, Ubicloud) advertise nested-virtualization runners. — [RunsOn docs](https://runs-on.com/docs/runners/capabilities/nested-virtualization/); [WarpBuild docs](https://warpbuild.com/docs/ci/nested-virtualization) (snippets)
- kvm-unit-tests is not KVM-only: a KVM Forum 2020 talk ("When 'KVM' doesn't mean KVM") lists TCG, Hypervisor.framework, WHPX, s390x LPAR/z/VM, x86 bare metal and VMware as targets. Nadav Amit's 2019 series got most x86 tests (vmx, apic, pmu, eventinj) running on bare metal. There is an x86_64 UEFI build (`./configure --target-efi`) producing `.efi` test binaries, originally motivated by SEV-ES and tested in VMs. — [KVM Forum 2020 slides](https://kvm-forum.qemu.org/2020/kvm-unit-tests_%20When%20KVM%20doesn%27t%20mean%20_KVM_.pdf); [Amit series](https://lore.kernel.org/kvm/20190628203019.3220-1-nadav.amit@gmail.com/t/); [x86/efi README](https://github.com/kvm-unit-tests/kvm-unit-tests/blob/master/x86/efi/README.md) (snippets) **[dated: 2019–2022]**
- Bochs gives explicit VMX failure reasons and a debugger; used in a Rust hypervisor course. — [Hypervisor-101-in-Rust](https://tandasat.github.io/Hypervisor-101-in-Rust/hypervisor-setup-and-operation-cycle/testing-with-bochs.html) (snippet)
- Example of a fast emulated test loop in a Rust hypervisor: 34 suites in about 5 s under QEMU. — [users.rust-lang.org post](https://users.rust-lang.org/t/30k-lines-of-no-std-rust-a-bare-metal-arm64-hypervisor-that-replaces-googles-hafnium-spmc/139497)
- OpenVMM keeps integration tests in-repo (`vmm_tests/`, "petri" framework); Hedron uses Catch2 unit tests of hypervisor logic on the host plus Nix-driven builds. — [openvmm PR 2221 contributor notes](https://github.com/microsoft/openvmm/pull/2221/files) (snippet, informal); [Hedron README](https://github.com/cyberus-technology/hedron)
- Oxide's Propolis "works best (and its CI tests run) on AMD hosts", showing that even well-funded teams test on a narrow hardware matrix. — [oxidecomputer/propolis](https://github.com/oxidecomputer/propolis) (snippet)

### Inferences
- Suggested loop: (1) host-side unit tests for all logic that does not touch hardware (page-table code, allocators, instruction decoder, virtio queues, scheduler) with `cargo test`, Miri and fuzzing; (2) QEMU TCG or KVM with `-cpu host,+vmx` / `+svm` booting the hypervisor from a UEFI image, serial to stdout, `isa-debug-exit` for pass/fail, QEMU's gdbstub for the hypervisor itself; (3) Bochs runs for VMX consistency failures; (4) kvm-unit-tests EFI binaries and Linux selftests as guests; (5) nightly real-hardware runs via BMC/Redfish + PXE/HTTP boot + serial-over-LAN.
- Caveat on nested development: L0 KVM's nested VMX/SVM emulation is more forgiving and feature-different than hardware (e.g. which secondary controls are exposed, posted interrupts, no real IOMMU unless using an emulated vIOMMU). Bugs will hide until bare metal. QEMU's emulated `intel-iommu`/`amd-iommu`/`virtio-iommu` let you develop the DMAR/IVRS code path but not validate it.
- Minimum hardware matrix for x86: one recent Intel server generation and one recent AMD EPYC generation, each with the target NIC and NVMe; add one ARM64 server (Ampere-class) only once x86 is stable. Bare-metal rental (Hetzner, OVH, Latitude, Vultr, PhoenixNAP, AWS `.metal`) covers this without a lab; Equinix Metal was announced for sunset, so do not plan around it (not verified this session).
- GitHub runners sit on Azure VMs, so a hypervisor under test there runs as L1 on Hyper-V with an L2 guest: fine for functional CI, not for performance.

### Gaps
- No verified statement on whether GitHub's standard runners expose nested VMX to an L1 guest *of QEMU/KVM* (i.e. three levels: Hyper-V → KVM → your hypervisor → guest). This needs a quick experiment; it is the configuration a hypervisor CI actually needs.
- Current bare-metal provider landscape (Equinix Metal successor options) was not researched.
- Did not confirm kvm-unit-tests EFI support has been validated on real bare metal.

---

## 8. Effort calibration: size, time, team, and failure modes

### Takeaway
The ring-0 core of a minimal hypervisor is small (roughly 10–40 KLOC across NOVA, Bao, ACRN) and a small team can write one; the total system is not, because the device model, toolstack, I/O path, memory features and hardware qualification dominate. Every large operator whose reasoning I could find (AWS, Oxide, and Firecracker's authors) chose to reuse an existing kernel-level hypervisor and write only the user-space VMM.

### Cited Findings
Code size
- NOVA: 9 KLOC microhypervisor + 7 KLOC user environment + 20 KLOC VMM = about 36 KLOC. — [NOVA FOSDEM 2013](https://hypervisor.org/nova-fosdem-2013.pdf); [arXiv 2112.06875](https://arxiv.org/pdf/2112.06875) (snippets) **[dated]**
- ACRN: about 40K LOC hypervisor (target under 40K), with device model outside it in the Service VM. — [ACRN introduction](https://projectacrn.github.io/3.3/_sources/introduction/index.rst.txt) (snippet)
- Bao: under 20K SLOC (second-hand citation). — [arXiv 2106.04514](https://arxiv.org/pdf/2106.04514.pdf) (snippet)
- Xen: 79,161 SLOC for the hypervisor vs 18,313,464 for Linux, per Edera's own `sloc` run; Edera calls the comparison a "naive first pass" and prefers comparing ring-0 executable pages. A 2008 list thread mentions about 300K LOC for Xen 3.3 on a different basis. These conflict; the Edera number likely counts a subset. — [Edera blog](https://edera.dev/stories/edera-reduces-the-hypervisor-attack-surface-with-rust-and-xen); [xen-devel 2008](https://lists.xenproject.org/archives/html/xen-devel/2008-09/msg00353.html) (snippets; vendor source)
- Firecracker: about 50 KLOC of Rust per the NSDI'20 paper summaries; another write-up says about 81K; a 2026 third-party count says about 83K (Firecracker) and about 106K (Cloud Hypervisor). QEMU is cited as over 1M LOC. — [the morning paper summary](https://blog.acolyer.org/2020/03/02/firecracker/); [arXiv 2005.12821](https://arxiv.org/pdf/2005.12821); [rywalker.com](https://rywalker.com/research/cloud-hypervisor) (snippets; later figures low-confidence)
- Solo Rust ARM64 hypervisor/SPMC: 30K lines, 230 KB binary. — [users.rust-lang.org](https://users.rust-lang.org/t/30k-lines-of-no-std-rust-a-bare-metal-arm64-hypervisor-that-replaces-googles-hafnium-spmc/139497)
- krata control plane: daemon about 8K SLoC, guest agent about 3K SLoC (own code; dependencies about 1M SLoC). — [lib.rs krata-daemon](https://lib.rs/crates/krata-daemon) (snippet)

Choices by experienced builders
- Oxide (RFD 26): considered only KVM/Linux and bhyve/illumos seriously. Xen was rejected for its "Large and complicated (by dom0) codebase"; emerging VMMs were excluded because they "Haven't been proven in production"; the guiding principle was to "choose software that is already close to being what we need" and pick "a stable foundation". Writing a new hypervisor was not even evaluated. They wrote a new Rust userspace (Propolis) on an existing kernel hypervisor (bhyve). bhyve weaknesses they accepted: mostly Intel-tested, limited passthrough, no nested virt. — [Oxide RFD 26](https://rfd.shared.oxide.computer/rfd/26)
- Firecracker (AWS Lambda/Fargate): "chose to keep KVM, but entirely replace QEMU". — [Firecracker NSDI'20](https://www.usenix.org/conference/nsdi20/presentation/agache) (snippet)
- AWS EC2 Nitro: moved from Xen to a minimized KVM-based hypervisor once I/O was offloaded to cards; no formal public statement of why KVM over Xen was found; press speculated about NVMe support and KVM's larger community. — [AWS Nitro whitepaper](https://docs.aws.amazon.com/whitepapers/latest/security-design-of-aws-nitro-system/the-nitro-system-journey.html); [The Register 2017](https://www.theregister.com/2017/11/07/aws_writes_new_kvm_based_hypervisor_to_make_its_cloud_go_faster/); [TechTarget](https://www.techtarget.com/searchaws/blog/AWS-Cloud-Cover/AWS-using-KVM-and-Xen-but-users-may-not-feel-any-impact) (snippets) **[dated: 2017]**
- Edera (startup, type-1 for container/AI isolation): reused the Xen core and rewrote only the toolstack in Rust. — [krata repo](https://www.github.com/edera-dev/krata) (snippet)
- Microsoft: new Rust work (OpenVMM/OpenHCL, Hyperlight) is all VMM-level on top of the existing Microsoft Hypervisor or KVM, not a new ring-0 hypervisor. — [microsoft/openvmm](https://github.com/microsoft/OpenVMM); [Hyperlight blog](https://opensource.microsoft.com/blog/2024/11/07/introducing-hyperlight-virtual-machine-based-security-for-functions-at-scale/) (snippets)

Signals of stall or narrow scope
- Hedron after about 10 years of commercial development: Intel only, no AMD. — [Hedron README](https://github.com/cyberus-technology/hedron)
- seL4 x86 VMM work was reported on hold in 2025 over kernel prerequisites; still needs a development SDK in 2026. — [seL4 devel list](https://lists.sel4.systems/hyperkitty/list/devel@sel4.systems/thread/FHQYU2A3RYSZKCJAR5EXEDLSAEJDF2VO/); [libvmm README](https://github.com/au-ts/libvmm) (snippets)
- pKVM (Google, well-resourced): protected guest support "has been somewhat languishing upstream" per Will Deacon in Jan 2026; arm64 protected guests landed in Linux 7.1 as "very experimental". — [linux-arm-kernel Jan 2026](https://lists.infradead.org/pipermail/linux-arm-kernel/2026-January/1091563.html); [Phoronix on Linux 7.1 KVM](https://phoronix.com/news/linux-7.1-kvm) (snippets)
- AxVisor: multi-arch Rust type-1, but only QEMU for x86 and ARM dev boards for hardware; no releases. — [AxVisor repo](https://github.com/arceos-hypervisor/axvisor)

### Inferences
- Planning estimate for a from-scratch Rust type-1 with a Linux service VM for I/O (my judgement, no source): 3–5 strong systems engineers; about 3–6 months to a single-vCPU Linux guest on QEMU and one real machine; about 9–15 months to SMP guests, virtio via service VM, IOMMU, two CPU vendors, and a usable API; about 2–3 years to something you would trust with untrusted multi-tenant workloads on a fleet (security review, speculation mitigations, live update, failure handling, snapshotting, observability). ARM64 adds roughly 30–50% to the platform work.
- Expected code size on that path: 20–40 KLOC ring-0 Rust per architecture family, plus 30–80 KLOC of VMM/device/control-plane code, most of which can be derived from Cloud Hypervisor/Firecracker/rust-vmm if the ABI is KVM-shaped.
- Where such projects stall, in my assessment: (1) the I/O story (drivers or an untidy dom0); (2) long-tail hardware and firmware variance; (3) interrupt/timer virtualization bugs that appear only under load; (4) continuous security work: each new CPU vulnerability class needs mitigations the Linux/Xen communities ship for free; (5) feature parity pressure (snapshots, live migration, hotplug, GPU passthrough, confidential computing); (6) operational tooling (crash dumps, tracing, perf) that nobody budgets for; (7) loss of the one or two people who understand the VMX/SVM core.
- The strongest counter-evidence to "it's too hard" is that small teams and individuals do ship cores of 10–40 KLOC. The strongest evidence against starting one is that no operator with a choice has done so for general Linux guests in the last decade.

### Gaps
- Team sizes and elapsed time for NOVA, ACRN, Bao, Bareflank/MicroV, Jailhouse, xvisor, Hyperlight, OpenVMM and Firecracker were **not** found; do not quote any.
- No source for Google's hypervisor rationale (KVM with a custom user-space VMM) was retrieved.
- No Bryan Cantrill statement beyond RFD 26 was retrieved; the "On the Metal"/blog material was not read.
- Xen's true current hypervisor LOC needs a direct `cloc` on the tree.

---

## 9. Build vs reuse: the five options

### Takeaway
For the stated purpose, option 4 (minimal Linux + KVM + a Rust VMM) is the low-risk baseline that the evidence favours, and option 3 (Xen core + Rust toolstack, as Edera does) is the proven way to get a real type-1 boundary. A from-scratch core (option 1) is feasible for a small team only if it copies the service-VM I/O model and a KVM-shaped ABI so the VMM is reused; options 2 and 5 are not ready on x86 servers.

### Cited Findings
- Option 2 status: Hedron is GPLv2, Intel-only, mirror-only. seL4 libvmm x86 needs a non-mainline Microkit SDK. — [Hedron README](https://github.com/cyberus-technology/hedron); [libvmm README](https://github.com/au-ts/libvmm) (second is snippet)
- Option 3 precedent: Edera runs Xen's core with a Rust control plane; x86_64 complete, aarch64 partial; its authors claim PV Xen reduces total trusted code by 66% vs KVM when drivers and guest OS are counted (vendor claim). Oxide rejected Xen for dom0 complexity. x86 dom0less (Hyperlaunch) status is unclear in 2026. — [krata repo](https://www.github.com/edera-dev/krata); [Edera blog](https://edera.dev/stories/edera-reduces-the-hypervisor-attack-surface-with-rust-and-xen); [Oxide RFD 26](https://rfd.shared.oxide.computer/rfd/26); [xen-users April 2026](https://lists.xen.org/archives/html/xen-users/2026-04/msg00016.html) (snippets except RFD 26)
- Option 4 precedent: Firecracker (keep KVM, replace QEMU, about 50 KLOC Rust at publication), Cloud Hypervisor (KVM and MSHV, SEV-SNP on KVM as of v52), AWS Nitro (minimized KVM). — [Firecracker NSDI'20](https://www.usenix.org/conference/nsdi20/presentation/agache); [Phoronix on CH 52](https://phoronix.com/news/cloud-hypervisor-52); [AWS Nitro whitepaper](https://docs.aws.amazon.com/whitepapers/latest/security-design-of-aws-nitro-system/the-nitro-system-journey.html) (snippets)
- Option 5 status: pKVM is arm64 only upstream; protected guests merged in Linux 7.1 as "very experimental"; kernel docs call pKVM "an experimental, development feature and subject to breaking changes"; basic "pKVM state isolation" appeared in an October 2026 pull. My search found **no** x86 pKVM/deprivileged-host work upstream. — [Phoronix on Linux 7.1](https://phoronix.com/news/linux-7.1-kvm); [kernel docs pKVM](https://docs.kernel.org/next/virt/kvm/arm/pkvm.html) (snippets)
- Licences as read: Hedron GPLv2; AxVisor Apache-2.0. — [Hedron README](https://github.com/cyberus-technology/hedron); [AxVisor repo](https://github.com/arceos-hypervisor/axvisor)

### Inferences
Comparison (my assessment; licences other than the two cited above are from background knowledge and should be re-checked):

| Option | Time to first useful sandbox host | TCB / isolation story | I/O | Density features (snapshots, overcommit) | Main risks | Licence |
|---|---|---|---|---|---|---|
| 1. New Rust type-1 + Linux service VM + reused Rust VMM | 9–15 months to credible prototype; 2–3 years to production | Smallest ring-0 if kept minimal; you own every bug and every CPU-vulnerability mitigation | Service VM passthrough + virtio backends | Must build all of it | Schedule, hardware long tail, security maintenance, bus factor | Your choice (Apache-2.0/MIT typical); free to take Apache/BSD code from rust-vmm, CH, Firecracker; cannot copy GPL code from Xen/Linux/Hedron |
| 2a. NOVA/Hedron + own Rust user-level VMM | 6–12 months if Intel-only is acceptable | Proven microhypervisor design, capability model | Needs a driver VM or own user-level drivers | Must build | Intel-only (Hedron), thin community, C++ core you must maintain | GPLv2 core; a separate user-level VMM over the syscall boundary is generally treated as a separate work, but get legal advice |
| 2b. seL4 + Microkit + libvmm | Long on x86 | Verified kernel, but x86 VT-x paths are outside the proof (unverified this session) | Driver VMs / sDDF drivers | Static system model fits poorly with dynamic VM churn | x86 immaturity, dynamic creation | Kernel GPLv2; userland BSD-2 (unverified) |
| 3. Xen core (PVH guests, minimal or disaggregated dom0) + Rust toolstack | 3–6 months (Edera shows the path; krata is open source) | Real type-1; about 80 KLOC+ C hypervisor with a long XSA history and an active security process | dom0/driver domain with Linux drivers; mature PV and passthrough | Weaker than KVM: no CoW snapshot/restore culture, limited memory sharing; ballooning and PoD exist | C codebase, dom0 still privileged unless you invest in disaggregation, smaller community than KVM, x86 dom0less uncertain | GPLv2 hypervisor; toolstack over the hypercall ABI can be your own licence (unverified) |
| 4. Minimal Linux + KVM + Firecracker/Cloud Hypervisor ("type-1 in practice") | Weeks | Large ring-0 (Linux), mitigated by jailer/seccomp, minimal kernel config, immutable image; best-audited virtualization path in the industry | All Linux drivers, vhost, io_uring, VFIO, SR-IOV | Best: snapshots, userfaultfd, balloon, virtio-mem, hugepages, KSM | Host kernel attack surface; less differentiated | Linux GPLv2 (kernel unmodified or patches published); VMMs Apache-2.0 |
| 5. pKVM-style deprivileged host | Not available on x86 | Attractive in theory: small EL2/root-mode core, Linux demoted to a service role | Linux drivers | Constrained by protection model | arm64-only and experimental | GPLv2 |

- A staged strategy that keeps options open: ship on option 4 now with a VMM whose hypervisor interface is abstracted (as Cloud Hypervisor's is), then, if the type-1 boundary is a real requirement, bring up option 1 or 3 underneath the same VMM and control plane. The service VM in option 1 is then literally the option-4 host image, minus KVM.
- Option 1 is essentially "ACRN/pKVM architecture, written in Rust, for servers". It is worth checking ACRN itself (BSD-3-Clause to my knowledge, x86, service-VM model, about 40K LOC) as a fork base or at least as the design reference before committing to a clean slate; its client/IoT focus (static configuration, limited VM counts) is the likely mismatch.
- Licence implication summary: permissive-licensed Rust VMM code (rust-vmm, Firecracker, Cloud Hypervisor, OpenVMM under MIT) is the main reuse pool for a proprietary or permissive new hypervisor; GPLv2 cores (Xen, Hedron, seL4, Linux/KVM) can be used as-is with source obligations for modifications, with your value-add kept in separately licensed user-level components.

### Gaps
- Licences of NOVA upstream, seL4 userland, ACRN, Bao, OpenVMM, Xen toolstack boundary were not verified this session.
- No independent (non-vendor) comparison of Xen vs KVM attack surface or XSA/CVE rates was gathered.
- No data on Edera's team size or time to product.

---

## 10. Can it run on cloud VMs? Nested virtualization and hardware requirements

### Takeaway
As of 2026 a custom hypervisor can in principle be developed and CI-tested on ordinary cloud VMs on all three major clouds on Intel (and some AMD on Azure and possibly GCP), but AWS officially supports only KVM and Hyper-V as the L1 hypervisor, nested costs 10% or more on CPU and more on I/O per Google, and no nested option exposes a real IOMMU or ARM virtualization. Production for an I/O-passthrough design means bare metal.

### Cited Findings
- AWS: nested virtualization on non-metal instances launched February 2026 on C8i, M8i, R8i; a June 2026 announcement extended it to C7i, M7i, R7i, C8id/M8id/R8id, the i7/i8 flex families, I7i and X8i, plus GovCloud. Only KVM and Hyper-V are supported L1 hypervisors. Graviton is not supported. No extra charge. AWS still recommends bare metal for performance-sensitive or low-latency virtualization workloads. The English user guide may lag and list only C8i/M8i/R8i. — [AWS What's New Feb 2026](https://aws.amazon.com/about-aws/whats-new/2026/02/amazon-ec2-nested-virtualization-on-virtual); [AWS What's New June 2026](https://aws.amazon.com/about-aws/whats-new/2026/06/nested-virtualization-intel-us-gov-cloud/); [EC2 user guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/amazon-ec2-nested-virtualization.html); [The Register](https://www.theregister.com/2026/02/17/nested_virtualization_aws_ec2/) (snippets)
- GCP: only Linux KVM is supported as the L1 hypervisor; docs warn nested VMs "might experience a 10% or greater decrease in performance for workloads that are CPU-bound, and possibly more for input/output-bound workloads". AMD and Arm VMs are listed as unsupported; one cached page version says N4D (AMD) is an exception. Conflict unresolved. — [GCP nested virtualization overview](https://docs.cloud.google.com/compute/docs/instances/nested-virtualization/overview) (snippet)
- Azure: nested virtualization is listed as supported on Intel Dv5/Dsv5 and AMD Dasv5/Dadsv5 (EPYC 7763v) series pages; originally introduced with Dv3/Ev3. — [Azure Dasv5 page](https://learn.microsoft.com/sv-se/AZURE/virtual-machines/dasv5-dadsv5-series); [Azure Dv5 page](https://learn.microsoft.com/en-us/azure/virtual-machines/dv5-dsv5-series); [Azure blog](https://azure.microsoft.com/en-us/blog/nested-virtualization-in-azure/) (snippets)
- Example hardware floors chosen by small hypervisors: Hedron requires ACPI, XSAVE, FSGSBASE and Intel; bhyve requires VT-x with EPT. — [Hedron README](https://github.com/cyberus-technology/hedron); [Oxide RFD 26](https://rfd.shared.oxide.computer/rfd/26)

### Inferences
- "Supported L1 hypervisors: KVM and Hyper-V" on AWS/GCP is a support statement; a custom hypervisor using the same VMX features that KVM uses will likely run, but anything it touches that KVM-as-L1 does not (unusual VMCS controls, posted interrupts, EPT features such as mode-based execute, PML, VMFUNC, SPP) may be absent or buggy, and you have no support recourse. Design the feature-detection path to degrade to the common subset.
- Under any cloud L0 there is no VT-d/AMD-Vi exposed, so a service-VM-with-passthrough I/O design cannot run as designed; the hypervisor would need a "nested mode" where the service VM uses the cloud's virtio/ENA/NVMe devices via... passthrough without an IOMMU (only safe if the service VM is trusted) or where the hypervisor trusts L0. This is a real design fork: a cloud-VM deployment target effectively requires option 4-like trust in the service VM.
- Suggested minimum bare-metal requirements for a clean-slate design (my recommendation): x86-64-v3 or newer; Intel with EPT, unrestricted guest, VPID, APICv with posted interrupts, VT-d with interrupt remapping, TSC-deadline, invariant TSC, x2APIC; or AMD Zen 3+ with NPT, AVIC/x2AVIC (Zen 4+), AMD-Vi; UEFI-only boot; MSI-X capable devices only. Requiring this floor removes shadow paging, real-mode emulation, PIC/PIT/IOAPIC emulation and most legacy code, which is the same trade bhyve and Hedron made.
- ARM64 cannot be developed nested on mainstream clouds in 2026 (no Graviton nested support; GCP excludes Arm); use QEMU TCG with `virtualization=on`, or bare-metal Ampere-class servers. Arm NV/NV2 support in KVM as an L0 exists upstream to my knowledge but I did not verify its 2026 state.

### Gaps
- No measured nested-virtualization overhead numbers for AWS's 2026 offering.
- GCP AMD (N4D) nested support is unconfirmed.
- No official statement from any cloud on running a non-KVM, non-Hyper-V L1 hypervisor.
- ARM64 nested virtualization availability on any cloud and in upstream KVM was not verified.

---

> Scope update (2026-11-26): ARM64 is a **confirmed** target, not optional. Sections 11–15 cover the AArch64 path at the same depth as x86-64. Where sections 1–10 call ARM64 an add-on or say "add an ARM server only once x86 is stable", sections 11–15 supersede them.

## 11. AArch64 language, toolchain and crates

### Takeaway
The Rust bare-metal AArch64 stack is in better shape than the x86 one for hypervisor work because Google maintains a set of `no_std` crates for exactly this (pKVM/Android firmware): `smccc` (SMCCC + PSCI), `arm-gic`, `aarch64-paging`, `aarch64-rt`, plus the community `aarch64-cpu` register crate and `fdt` parsers. The one real codegen trap is floating point/SIMD: use a soft-float or FP-disabled build for EL2 code.

### Cited Findings
- `smccc` (Google) is `no_std`, covers Arm SMC Calling Convention 1.4 and PSCI 1.1 on aarch64, and can issue calls by either HVC or SMC (0.2.x); newer versions also cover aarch32. — [smccc on docs.rs](https://docs.rs/crate/smccc/0.2.1); [google/psci repo](https://github.com/google/psci) (snippets)
- Google's `osdemo` example combines `arm-gic` (Generic Interrupt Controller driver), `smccc`, `aarch64-paging` and `aarch64-rt` as a reference for bare-metal aarch64 OS development in Rust. — [osdemo on lib.rs](https://lib.rs/crates/osdemo) (snippet)
- `aarch64-cpu` (v11.x) is a `no_std` crate for system-register access, originating in the rust-raspberrypi-OS-tutorials project. — [aarch64-cpu docs.rs](https://docs.rs/aarch64-cpu/11.1.0/src/aarch64_cpu/lib.rs.html) (snippet)
- Real-project evidence: a 30K-line `no_std` Rust EL2 hypervisor/SPMC names `fdt` as its only external crate and uses `alloc` with a bump allocator. Its worst bug was a debug build turning `read_volatile` into NEON instructions that EL3 firmware trapped (silent hang, hours in GDB); advice is `opt-level = 1` from the start. — [users.rust-lang.org post](https://users.rust-lang.org/t/30k-lines-of-no-std-rust-a-bare-metal-arm64-hypervisor-that-replaces-googles-hafnium-spmc/139497)
- `limine` crate 0.6.3 has an aarch64 MP module whose `MpInfo` exposes processor ID and MPIDR, so Limine can be the UEFI loader and can start secondary CPUs on Arm too. `uefi-rs` supports aarch64. — [limine 0.6.3 aarch64 MP source](https://docs.rs/crate/limine/0.6.3/source/src/mp/aarch64.rs); [uefi-rs](https://github.com/ben-krieger/uefi-rs) (snippets)
- `virtio-drivers` (no_std, used in Android's pKVM `vmbase`/Trusty) gives guest-side virtio block/net/console for when the hypervisor itself runs on QEMU `virt`. — [virtio-drivers docs.rs](https://docs.rs/crate/virtio-drivers/0.7.5); [Android vmbase README](https://android.googlesource.com/platform/packages/modules/Virtualization/+/2e2f48a9/vmbase/README.md) (snippets)

### Inferences
- Targets (background knowledge, not verified this session): `aarch64-unknown-none` (hard-float ABI, uses FP/SIMD registers) and `aarch64-unknown-none-softfloat` (no FP/SIMD use). Build the EL2 core with the soft-float target so the hypervisor never touches guest FP/SVE/SME state and can lazily switch it; this also avoids the NEON trap above. Both are prebuilt rustup targets, so stable Rust works without `build-std`.
- Crates the hypervisor still has to write or heavily extend: `aarch64-cpu` covers common EL1/EL2 registers, but expect gaps for GICv3 hypervisor interface (`ICH_*`), newer features (FGT, NV, SVE/SME traps, PAuth keys), stage-2 specific formats; `arm-gic` is a *physical* GIC driver, not a vGIC; no reusable SMMUv3 driver or ACPI IORT parser for hypervisor use that I know of (the `acpi` crate's coverage of MADT GICC/GICD/GICR/ITS and IORT entries needs checking).
- Device tree and ACPI are both required (see section 12), so budget for `fdt` (parse host DT on boards) plus an FDT *writer* for guest DTs (`vm-fdt` from rust-vmm does this; `no_std` status unverified).

### Gaps
- `arm-gic`, `aarch64-paging`, `aarch64-rt`, `vm-fdt`, `acpi` crate: versions, `no_std` status, GICv3 hypervisor-interface and IORT coverage not verified.
- Tier status of `aarch64-unknown-none*` not checked this session.

---

## 12. AArch64 bare-metal bring-up and ordered milestones (EL2, stage-2, vGIC, Linux at EL1)

### Takeaway
Reaching a Linux guest is quicker on Arm than on x86: firmware hands over at EL2 in a defined state, PSCI starts secondary CPUs, there is no VMCS-style consistency checking, and GICv3 has built-in virtual CPU interface support. The cost moves to platform variance: servers boot with UEFI+ACPI (MADT GIC structures, GTDT, IORT, MCFG), boards boot with devicetree, and IOMMU/MSI plumbing (SMMUv3 + ITS) is harder than VT-d.

### Cited Findings
- Entry state: Linux's arm64 boot protocol states the CPU "must be in non-secure state, either in EL2 (RECOMMENDED in order to have access to the virtualisation extensions), or in EL1"; a hypervisor loaded the same way by UEFI/U-Boot gets the same handover. With GICv3, system-register access must be enabled by higher levels; with GICv2-compat mode `ICC_SRE_EL2.SRE` must be 0 and DT/ACPI must describe a GICv2. — [Booting AArch64 Linux](https://www.kernel.org/doc/html/v7.1/arch/arm64/booting.html) (snippet)
- Server platform contract: Arm SystemReady (SR band) certifies BSA + SBSA hardware and SBBR firmware (UEFI, ACPI, SMBIOS), and the requirements cover SMCCC and PSCI calls and secondary-core boot. SBSA v7.2 is dated Nov 2024, BBR v2.2 June 2025. The IR band (embedded) is BSA + EBBR with devicetree. — [Arm SystemReady](https://arm.com/systemready); [UEFI Forum SystemReady slides](https://uefi.org/sites/default/files/resources/15.%20Integrate%20Arm%20SystemReady_Arm_Wang_FINAL.pdf) (snippets; clause-level GICv3/SMMUv3 requirements not confirmed from an official page)
- PSCI 1.1 via SMC/HVC is available from the `smccc` crate (CPU_ON, CPU_OFF, SYSTEM_RESET etc.). — [smccc docs.rs](https://docs.rs/crate/smccc/0.2.1) (snippet)
- QEMU `virt` gives a full emulated target: `virtualization=on` exposes EL2 (default off); `gic-version=3|4|max` (under TCG `max` is 4 when virtualization is on); `iommu=smmuv3` creates a machine-wide SMMUv3 for the PCIe topology. An older doc revision notes ITS is not modeled under TCG; check the release in use. — [QEMU virt docs](https://www.qemu.org/docs/master/system/arm/virt) (snippet)
- Scope-cutting precedent: KVM/arm64 restricted nested support to FEAT_NV2 only, dropping plain FEAT_NV, because all hardware with NV has NV2. A new hypervisor can similarly require a recent architecture baseline. — [linux-arm-kernel, Nov 2023](https://lists.infradead.org/pipermail/linux-arm-kernel/2023-November/882044.html) (snippet)
- AxVisor runs Linux guests on four real ARM64 boards (RK3588, RK3568, Phytium) from a Rust type-1 code base, i.e. the Arm path is the more hardware-proven one in the Rust hypervisor ecosystem. — [AxVisor repo](https://github.com/arceos-hypervisor/axvisor)

### Inferences
Ordered milestones (background engineering knowledge, not verified against sources this session; effort for 2–3 experienced engineers is my estimate).

1. **A0 – Boot at EL2 and print (about 1 week).** Entry as a UEFI application (PE/COFF with the arm64 Image header, or via Limine) on servers; flat image + DTB pointer in x0 on boards/QEMU `-kernel`. Verify `CurrentEL == EL2`; set `VBAR_EL2`, stack, BSS; PL011 or 16550 UART (address from ACPI SPCR/DBG2 or DT `/chosen/stdout-path`).
   - Pitfalls: firmware entering at EL1 (cloud VMs, some boards; nothing can be done); FP/SIMD traps (`CPTR_EL2`) and Rust emitting NEON; alignment faults from `SCTLR_EL2.A`; relocating a position-independent image; UEFI leaving the MMU and caches on with its own tables.
2. **A1 – EL2 MMU and memory (1–2 weeks).** Decide **nVHE vs VHE** now: with `HCR_EL2.E2H=1` (VHE, Armv8.1+) EL2 gets two translation regimes and EL1-like registers, which suits a hypervisor with EL0 user-mode VMMs; nVHE is simpler for a pure EL2 kernel. Stage-1 tables for EL2, `MAIR/TCR/SCTLR_EL2`, memory map from UEFI or DT `/memory` + reserved-memory.
   - Pitfalls: break-before-make rules when changing live mappings; cache maintenance before enabling the MMU; 4K vs 16K/64K granule support (`ID_AA64MMFR0_EL1`); physical address size up to 52 bits (LPA/LPA2).
3. **A2 – Platform discovery (2–3 weeks).** Two front ends feeding one internal model: ACPI (MADT GICC/GICD/GICR/ITS entries, GTDT for timer PPIs, MCFG for ECAM, IORT for SMMU/ITS ID mappings, PPTT for topology, SPCR) and FDT. No AML needed if PCI uses MSI only.
   - Pitfalls: IORT ID-mapping chains (RID → StreamID → DeviceID); firmware that only gets one of ACPI/DT right; reserved regions (RMR nodes in IORT).
4. **A3 – GICv3 host + timers + SMP (2–3 weeks).** Distributor and per-CPU redistributor init, `ICC_SRE_EL2`, priority mask, EL2 physical timer (`CNTHP_*`) for the scheduler tick; secondaries via PSCI `CPU_ON` with an EL2 entry stub; SGIs for IPIs; per-CPU data in `TPIDR_EL2`.
   - Pitfalls: redistributor wake sequence (`GICR_WAKER`); MPIDR affinity vs linear CPU index; PSCI conduit (SMC on servers, sometimes HVC); errata-dependent CPU features on big.LITTLE boards (RK3588: A76+A55 differ in features, so vCPU feature set must be the intersection or vCPUs must be pinned per cluster).
5. **A4 – First guest code at EL1 (1–2 weeks).** `HCR_EL2` (VM, IMO/FMO/AMO, TSC, TWI/TWE, RW, and friends), `VTCR_EL2`/`VTTBR_EL2` with VMID, stage-2 tables, `SPSR_EL2`/`ELR_EL2` then `ERET`; world switch saving EL1 system registers, GPRs and (lazily) FP/SVE; exception vector for lower-EL sync/IRQ; decode `ESR_EL2` (HVC, SMC, data/instruction abort, sysreg trap, WFx).
   - Pitfalls: far fewer than x86 at this step. Common ones: wrong `VTCR_EL2` T0SZ/SL0 vs table levels, forgetting `ISB`, stage-2 permission/access-flag faults, VMID rollover and TLB invalidation, not context-switching `CNTVOFF_EL2`/`CNTKCTL_EL1`, PAuth keys and SVE vector length per vCPU.
6. **A5 – Linux to an initramfs shell, 1 vCPU (3–5 weeks).** Load `Image` at a 2 MiB-aligned address with a generated guest DTB (memory, `psci` node with `method = "hvc"`, GICv3 node, `arm,armv8-timer`, UART, `chosen`); emulate PSCI for the guest (at least `PSCI_VERSION`, `CPU_ON`, `CPU_OFF`, `SYSTEM_OFF/RESET`, `FEATURES`) and SMCCC arch calls (version, `ARCH_FEATURES`, Spectre workaround discovery); ID-register sanitisation (trap with `HCR_EL2.TID3` and present a filtered view: hide SVE/SME/MTE/PMU/SPE/AMU/RAS features not context-switched); virtual timer passthrough (`CNTV_*` with `CNTVOFF_EL2`), trapping or emulating the EL1 physical timer; emulated PL011 by stage-2 data-abort MMIO using `ESR_EL2.ISV` syndrome (no instruction decoder needed for normal accesses).
   - Pitfalls: aborts without valid syndrome (`ISV=0`, e.g. load-pair to MMIO) need a decoder or must be rejected; guest cache-maintenance and `HCR_EL2.TVM`-style handling while the guest MMU is off; DTB placement and size limits; speculative-execution mitigations exposed through SMCCC.
7. **A6 – Virtual GIC (4–6 weeks, long tail).** GICv3 only: guest CPU interface is handled in hardware through `ICH_LR<n>_EL2` list registers once `ICH_HCR_EL2.En` is set; the hypervisor emulates the distributor and redistributor MMIO frames, maintains pending/active state, fills list registers on entry, handles maintenance interrupts, and links physical to virtual IRQs for passed-through devices (HW bit) and for the virtual timer PPI.
   - Pitfalls: list-register overflow and priority ordering; level-triggered resampling; SGI delivery between vCPUs (trap `ICC_SGI1R_EL1`); migrating a vCPU with active interrupts; EOI mode with deactivation split. This is the Arm equivalent of the x86 interrupt-injection long tail, and it is where most Arm hypervisor bugs live.
8. **A7 – SMP guests (2–3 weeks).** PSCI `CPU_ON` creates/starts vCPUs; WFI trapping and blocking; vSGIs; per-vCPU `VMPIDR_EL2`; PV time (stolen-time SMCCC interface) for fair scheduling.
9. **A8 – virtio (shared with x86).** virtio-mmio described in the guest DT is the least-effort transport; virtio-pci needs an emulated ECAM host bridge and a vITS or GICv2m-style MSI frame. Backends live wherever the x86 design puts them.
10. **A9 – SMMUv3, ITS and passthrough (8–14 weeks).** SMMUv3 stage-2 with stream table, command/event queues, per-device STEs sharing the VM's stage-2 tables or separate ones; default-deny for all StreamIDs; physical ITS driver (device/collection/ITT tables, LPI configuration and pending tables) because PCIe MSIs on servers arrive as LPIs; a virtual ITS, or GICv4/4.1 direct vLPI injection, for passed-through devices.
    - Pitfalls: ITS is a substantial driver with command queues and memory tables shared with hardware; SMMU vendor quirks and firmware RMRs; devices sharing StreamIDs; no interrupt-remapping equivalent to lean on beyond the ITS itself; GICv4 doorbell and vPE scheduling complexity (skip GICv4 for v1).
11. **A10 – Server hardening.** RAS (SError routing, firmware-first error handling via SDEI/APEI GHES), CPU errata workarounds table (Linux's `cpu_errata.c` is the reference list), SVE/SME state management and vector-length policy, PAuth/BTI/MTE exposure policy, PMU virtualisation, CPU hotplug and idle states through PSCI `CPU_SUSPEND`, kexec/live update.

Comparison with the x86 list in section 3: A0–A5 total roughly 9–16 weeks versus M0–M5 at roughly 13–24 weeks; the vGIC (A6) is comparable to M6; SMMUv3+ITS (A9) is larger than VT-d (M9).

### Gaps
- Official SBSA clause text for mandatory GICv3/SMMUv3 levels not retrieved.
- No sourced evidence on how well SystemReady-certified servers (Ampere Altra/AmpereOne, Graviton metal, Cobalt) behave for a non-Linux EL2 payload; firmware quality claims are untested.
- No sourced timings for Arm bring-up by comparable teams.

---

## 13. Reusable Arm hypervisor code and designs

### Takeaway
There is more open, small, readable Arm hypervisor code than x86 code, but almost all of it targets embedded boards with devicetree and static partitioning; none is a ready base for dynamic, high-density sandboxes on ACPI servers. Use these as references (and Xen-on-Arm or KVM as fallbacks), and expect to write the vGIC, SMMUv3/ITS and ACPI front end yourself.

### Cited Findings
- pKVM: upstream arm64 only; protected guests landed in Linux 7.1 as "very experimental"; kernel docs call it "an experimental, development feature and subject to breaking changes"; an October 2026 pull describes basic "pKVM state isolation". Protected-guest upstreaming was described by Will Deacon in January 2026 as "somewhat languishing". GPLv2 as part of Linux. — [Phoronix on Linux 7.1](https://phoronix.com/news/linux-7.1-kvm); [kernel docs pKVM](https://docs.kernel.org/next/virt/kvm/arm/pkvm.html); [linux-arm-kernel Jan 2026](https://lists.infradead.org/pipermail/linux-arm-kernel/2026-January/1091563.html) (snippets)
- Hafnium: now the reference S-EL2 Secure Partition Manager Core in the TF-A stack (FF-A), AArch64 only, with Arm FVP as the only platform supported in the TF-A reference stack. It has moved away from being a general normal-world hypervisor. — [Hafnium SPM docs](https://git.trustedfirmware.org/plugins/gitiles/hafnium/hafnium.git/+/49b95f03f91885ce871d5313f941f8d274c9b935/docs/secure-partition-manager/secure-partition-manager.rst); [TF-A SPM docs](https://trustedfirmware-a.readthedocs.io/en/v2.4/components/secure-partition-manager.html) (snippets)
- Bao: clean-slate static partitioning hypervisor for Armv8-A and RISC-V, under 20K SLOC (second-hand); listed boards are embedded (Xilinx ZCU102/ZCU104, Ultra96, HiKey 960, i.MX8, QEMU virt); boots standalone without a Linux root cell. — [Bao paper, OASIcs NG-RES 2020](https://drops.dagstuhl.de/entities/document/10.4230/OASIcs.NG-RES.2020.3); [Bao README (fork)](https://github.com/ninolomata/bao-hypervisor/blob/main/README.md); [arXiv 2106.04514](https://arxiv.org/pdf/2106.04514.pdf) (snippets; platform list may be dated)
- A comparative study covers Jailhouse, Xen dom0less, Bao and the seL4 CAmkES VMM on Arm for code size, performance, interrupt latency, inter-VM communication and boot time. — [Martins & Pinto, arXiv 2303.11186](https://arxiv.org/abs/2303.11186v2) (snippet; figures not retrieved)
- seL4 + Microkit + libvmm: AArch64 is the mature path (libvmm 0.2.0, Aug 2026, "runs on AArch64 and x86-64"; x86 still needs a development SDK). — [Trustworthy Systems](https://www.trustworthy.systems/projects/virtualisation/); [au-ts/libvmm](https://github.com/au-ts/libvmm) (snippets)
- Xen on Arm: dom0less (multiple domains built at boot from devicetree, no dom0 required: "true static partitioning") is an established Arm feature; Xen 4.21 added virtio-pci and parallel boot work for dom0less systems. — [Xen blog: true static partitioning with dom0less](https://xenproject.org/blog/true-static-partitioning-with-xen-dom0-less/); [Xen 4.21 release](https://www.linuxfoundation.org/press/xen-project-delivers-xen-4.21-a-modernized-hypervisor-with-broader-architecture-support-and-improved-performance) (snippets)
- Rust-Shyper: Rust AArch64 embedded type-1 with VM migration and hypervisor live-update (JSA 2023 paper); an openEuler project. — [openEuler write-up](https://www.openeuler.org/en/blog/20230428-rust/20230428-rust); [Gitee repo](https://gitee.com/openeuler/rust_shyper) (snippets) **[dated]**
- AxVisor: Apache-2.0 Rust type-1, one code base for x86_64/aarch64/RISC-V, Linux guests on RK3588/RK3568/Phytium boards and QEMU; 118 commits, no releases. — [AxVisor repo](https://github.com/arceos-hypervisor/axvisor)
- Other small Rust AArch64 type-1 projects exist as learning references (e.g. a 30K-line SPMC/hypervisor validated against TF-A + pKVM). — [project page](https://willamhou.github.io/hypervisor/); [users.rust-lang.org post](https://users.rust-lang.org/t/30k-lines-of-no-std-rust-a-bare-metal-arm64-hypervisor-that-replaces-googles-hafnium-spmc/139497)
- Edera/krata's Xen-based stack lists aarch64 at 10–30% complete, so the "Xen core + Rust toolstack" route is not yet proven on Arm by its main proponent. — [krata repo](https://www.github.com/edera-dev/krata) (snippet)

### Inferences
- Best reading order for an implementer: Bao (smallest complete Armv8 + GICv3 static hypervisor in C) → KVM/arm64 `arch/arm64/kvm/hyp/nvhe` and `vgic/` (the production reference for world switch, vGIC-v3, ID-register sanitisation, errata; GPLv2, so study but do not copy into a permissively licensed code base) → Xen `xen/arch/arm` for ACPI + SMMUv3 + ITS on servers → AxVisor/Rust-Shyper for Rust structuring.
- On Arm, "option 5" (pKVM-style) is the one place where a deprivileged-Linux design actually exists upstream. If the Arm product could ship on pKVM while x86 uses something else, the two architectures would diverge in architecture and TCB story; that undercuts the single-codebase goal and pKVM's experimental status makes it a poor foundation in 2026.
- Xen on Arm is a credible fallback for Arm servers (ACPI, SMMUv3, ITS, dom0less all exist), stronger relative to the alternatives than Xen is on x86.
- Licences from background knowledge (verify): Bao Apache-2.0; Hafnium BSD-3-Clause; Rust-Shyper MulanPSL-2.0; Xen and pKVM GPLv2; seL4 kernel GPLv2.

### Gaps
- Bao's GICv3/SMMU support level, current platform list and licence not confirmed from the official repo.
- No LOC or team-size data for Rust-Shyper, Hafnium, Xen/Arm.
- Whether any of these has been run on Ampere/Graviton-class ACPI servers (other than Xen and KVM) is unknown.

---

## 14. AArch64 development workflow: emulation, nested virtualization, hardware

### Takeaway
QEMU `virt` with `virtualization=on` under TCG is the universal dev and CI target and needs no special host, but it is slow and emulation-accurate rather than hardware-accurate. Hardware-accelerated nested development needs FEAT_NV2 silicon on bare metal (AmpereOne-class or Neoverse V2-class), is not offered by any cloud VM I could confirm, and its upstream KVM state in late 2026 was not verified; so plan on cheap EL2-capable boards plus rented Arm bare metal from early in the project.

### Cited Findings
- QEMU `virt`: `virtualization=on` emulates the Arm Virtualization Extensions (EL2); GIC version selectable up to 4 with virtualization on; `iommu=smmuv3` adds a machine-wide SMMUv3; these let the full A0–A9 path run in emulation. ITS-under-TCG support varies by QEMU version. — [QEMU virt docs](https://www.qemu.org/docs/master/system/arm/virt) (snippet)
- Nested virtualization on Arm requires FEAT_NV2 hardware on a bare-metal host with recent kernel and QEMU; Arm's own learning path states it "isn't available inside a standard VM". — [Arm learning path: nested virtualization on an Arm server](https://learn.arm.com/learning-paths/servers-and-cloud-computing/nested-virtualization/) (snippet)
- KVM/arm64 nested support history: groundwork merged in 6.3 (2023); restricted to FEAT_NV2 only (Nov 2023 series); NV_frac handling (Feb 2025); "Recursive NV" series v4 (May 2025). No 2026 status confirmation found. — [linux-arm-kernel Nov 2023](https://lists.infradead.org/pipermail/linux-arm-kernel/2023-November/882594.html); [Feb 2025](https://lists.infradead.org/pipermail/linux-arm-kernel/2025-February/1003138.html); [May 2025](https://lists.infradead.org/pipermail/linux-arm-kernel/2025-May/1028107.html) (snippets)
- Ampere states AmpereOne supports nested virtualization via NV2 (vendor claim). — [Ampere blog "Unlocking layers"](https://amperecomputing.com/blogs/unlocking-layers) (snippet)
- AWS: Graviton is explicitly not supported for the 2026 nested-virtualization feature; older re:Post answers say Graviton2 (Armv8.2) lacks NV. GCP lists Arm VMs as unsupported for nested virtualization. — [EC2 nested virtualization guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/amazon-ec2-nested-virtualization.html); [AWS re:Post](https://www.repost.aws/questions/QUEoabj2ZERq2P5QFL6d6-RQ/nested-virtualization-on-graviton); [GCP overview](https://docs.cloud.google.com/compute/docs/instances/nested-virtualization/overview) (snippets)
- GCP Axion and Graviton4 both use Neoverse V2 cores (Debian list post). — [debian-arm May 2025](https://lists.debian.org/debian-arm/2025/05/msg00018.html) (snippet)
- Dev boards with proven EL2 hypervisor use: RK3588 (Orange Pi 5 Plus, EVM3588), RK3568 and Phytium Pi run AxVisor with Linux guests. — [AxVisor repo](https://github.com/arceos-hypervisor/axvisor)
- Raspberry Pi 5: seL4 developers note seL4 needs EL2 to act as hypervisor there and that the Pi 5 CPU cannot run 32-bit code at EL1 or above. — [seL4 devel list, Oct 2025](https://go.mail-archive.com/devel@sel4.systems/msg04575.html) (snippet)
- A realistic QEMU-based loop: 34 test suites in about 5 s, plus end-to-end tests against real TF-A firmware. — [users.rust-lang.org post](https://users.rust-lang.org/t/30k-lines-of-no-std-rust-a-bare-metal-arm64-hypervisor-that-replaces-googles-hafnium-spmc/139497)

### Inferences
- Tiered workflow:
  1. Every commit: host unit tests + QEMU TCG `-M virt,virtualization=on,gic-version=3 -cpu max` (runs on any x86 CI runner, including GitHub-hosted; a Linux guest boot under double emulation takes tens of seconds to minutes). Add `iommu=smmuv3` and UEFI (EDK2 `QEMU_EFI.fd`, which can provide ACPI tables) variants to exercise the ACPI and SMMU paths.
  2. Daily: boards on the desk with UART and network boot. RK3588 boards (8 cores, GICv3, up to 32 GB, about USD 100–200) are the best value and have big.LITTLE to shake out asymmetric-feature bugs; Raspberry Pi 4 is GICv2 (BCM2711 GIC-400), so it only helps if you carry a GICv2 path, which a server-focused design should not; Raspberry Pi 5 enters at EL2 and I believe has no GICv3 either (unverified). Boards are devicetree-only and have no SMMUv3/ITS comparable to servers.
  3. Nightly/weekly: real servers. Ampere Altra/AmpereOne bare metal (Hetzner's Arm dedicated line, Oracle Cloud Ampere bare-metal shapes, Equinix-successor hosts) and AWS Graviton `.metal` instances; these are the only places to validate UEFI+ACPI, SMMUv3, ITS, RAS, and 64–192 core SMP. Provider availability not verified this session.
- Nested acceleration (your hypervisor as an L1 guest of KVM on Arm) would speed the loop but requires owning or renting NV2-capable bare metal and a kernel with working NV; treat it as optional. Unlike x86, there is no Bochs-equivalent need: Arm has no entry-check failure class, and QEMU TCG with `-d int,guest_errors` plus gdbstub is adequate.
- Arm FVP (Fixed Virtual Platform) models are the reference for newer architecture features (SME, GICv4.1, RME) when QEMU lags; free Base FVPs exist (unverified this session).
- kvm-unit-tests has arm64 tests (gic, timer, psci, pmu) usable as guest payloads; not verified this session.

### Gaps
- Whether mainline KVM/arm64 nested virtualization is enabled and usable by default in late-2026 kernels: not confirmed.
- Azure Cobalt and GCP Axion nested-virtualization status: nothing found. Graviton4 metal NV2 availability to customers: nothing found.
- Raspberry Pi 5 interrupt controller type and EL2 handover details: not confirmed.
- Current Arm bare-metal provider list and pricing: not researched.

---

## 15. One codebase for x86-64 and AArch64: arch abstraction and bring-up order

### Takeaway
Dual-arch hypervisors are normal (Xen, KVM, Bao, seL4, AxVisor all do it), and the evidence suggests about half or more of a small hypervisor plus essentially all of the VMM/control plane can be architecture-neutral if the seam is drawn at "vCPU run loop, stage-2 page tables, interrupt controller, platform discovery". My recommendation is to bring up AArch64 first on QEMU to shape the portable core, start x86 no later than the point the Arm guest boots Linux, and never let either architecture go more than a milestone ahead.

### Cited Findings
- AxVisor explicitly uses "the same codebase to support three architectures: x86_64, Arm (aarch64), and RISC-V" with functionality split into independent components behind standard interfaces; its source tree has a `src/hal/arch/` layer. — [AxVisor repo](https://github.com/arceos-hypervisor/axvisor); [docs.rs axvisor source tree](https://docs.rs/crate/axvisor/0.5.4/source/src/hal/arch/) (second is snippet)
- Bao was designed clean-slate for both Armv8 and RISC-V and stays under about 20K SLOC. — [Bao paper](https://drops.dagstuhl.de/entities/document/10.4230/OASIcs.NG-RES.2020.3); [arXiv 2106.04514](https://arxiv.org/pdf/2106.04514.pdf) (snippets)
- Cloud Hypervisor already abstracts hypervisor back ends (KVM, MSHV) and `vm-memory` lists x86_64, ARM64 and RISCV64 support, so the reusable Rust VMM layer is multi-arch today. — [Phoronix on CH 26](https://www.phoronix.com/news/Cloud-Hypervisor-26.0); [rust-vmm/vm-memory](https://github.com/rust-vmm/vm-memory)
- Counter-examples where the second architecture lagged for years: Hedron/NOVA lineage is x86-only (Intel-only for Hedron); Edera lists aarch64 at 10–30%; seL4 libvmm's x86 support trails its Arm support; Oxide's Propolis is tested on AMD only. — [Hedron README](https://github.com/cyberus-technology/hedron); [krata repo](https://www.github.com/edera-dev/krata); [au-ts/libvmm](https://github.com/au-ts/libvmm); [oxidecomputer/propolis](https://github.com/oxidecomputer/propolis) (last three snippets)
- Limine and `uefi-rs` cover both architectures, so one boot path (UEFI application) can serve both. — [limine crate 0.6.3](https://docs.rs/crate/limine/0.6.3); [uefi-rs](https://github.com/ben-krieger/uefi-rs) (snippets)

### Inferences
Suggested structure (my design recommendation):

- `hv-core` (arch-neutral, `no_std`): physical frame allocator, heap, VM and vCPU objects and lifecycle state machines, scheduler, timers (deadline abstraction), guest-physical address-space manager (regions, CoW, sharing, ballooning policy), hypercall/control ABI, inter-VM shared memory and notification channels, virtio-mmio transport if any is in-hypervisor, logging/tracing, device-assignment policy. Target: no `cfg(target_arch)` inside it.
- `hv-arch-{x86_64,aarch64}` implementing a small set of traits:
  - `Vcpu`: create, load/save state, `run() -> Exit` with a **portable exit enum** (MMIO read/write with decoded address/size/data, hypercall, halt/WFI, shutdown/reset, plus an `ArchSpecific` variant), inject-interrupt, pending-work kick.
  - `Stage2`: map/unmap/protect at 4K/2M/1G, dirty/access tracking, TLB invalidation by VM; EPT, NPT and Arm stage-2 all fit one interface (three formats, same algorithm; a generic page-table walker parameterised by entry format works).
  - `HostCpu`: per-CPU init, idle, IPI, context for the hypervisor itself, feature detection, errata/mitigation hooks.
  - `IrqChip` host side (x2APIC vs GICv3) and `VirtIrqChip` guest side (APICv/AVIC-assisted LAPIC vs vGICv3). **Do not try to unify the virtual interrupt controllers**; expose only "assert/deassert guest interrupt line N / send MSI" upward.
  - `Iommu`: attach/detach device to VM address space, default-deny, MSI routing (VT-d/AMD-Vi vs SMMUv3+ITS).
  - `Platform`: discovery producing one internal description (CPUs, memory, interrupt controllers, timers, PCI segments, IOMMU units, console) from ACPI on both architectures plus FDT on Arm; share the ACPI table walker, keep table-specific parsers (MADT LAPIC vs GIC entries, DMAR/IVRS vs IORT) per arch.
  - `GuestBoot`: x86 `boot_params`/PVH vs Arm `Image` + generated FDT.
- Shared outside ring 0/EL2: VMM/device models, control plane, API, image/snapshot tooling, test harness. Guest-visible platform should be made as similar as possible on both: virtio-only devices, PSCI-like/ACPI-reduced minimal platform, no legacy.
- Things that will not abstract cleanly and need per-arch owners: world switch and exit handling; CPU feature/ID filtering (CPUID+MSRs vs ID registers); FP/vector state (XSAVE/AMX vs SVE/SME); speculation mitigations and errata; virtual interrupt controller; nested virt if ever needed; x86 instruction emulation for MMIO (Arm mostly avoids it through syndrome info).

Which to bring up first:

- **Arm first for the first guest, x86 immediately after** is my recommendation:
  - Arm's EL2 model is the cleaner specification of what a hypervisor needs (explicit trap bits, syndrome-decoded MMIO, PSCI, hardware vGIC CPU interface), so an abstraction designed against Arm and then stretched to cover VMX and SVM tends to stay small; one designed around VMCS concepts tends to leak x86 notions (exit-qualification style details, instruction decode, APIC assumptions) into the portable core.
  - The Rust crate support (Google's aarch64 crates) and small readable references (Bao, AxVisor, Rust-Shyper) are better on Arm; QEMU TCG gives full EL2 without nested-virt host requirements.
  - x86 actually has three back ends (Intel VMX, AMD SVM, and their differing interrupt virtualisation), so it is where most of the arch-specific code and calendar time will go; it must not start late.
- Arguments for x86 first, which may outweigh the above depending on the team: the commodity server fleet, bare-metal rental and cloud nested-virt CI are overwhelmingly x86; x86 development can use hardware-accelerated nested KVM today while Arm development is stuck on slow TCG or physical boards; and the commercially important I/O path (VT-d passthrough to a service VM) is easier to validate on x86.
- Either way, the rule that matters is parallelism: put the second architecture's "boot, print, enter guest, exit" (M3/A4 level) in CI before the first architecture's virtio or IOMMU work begins, and gate every `hv-core` change on both builds. The counter-examples above show that a second architecture added later tends to stay at 10–30% for years.
- Staffing implication: at least one engineer who owns each architecture; a 3-person team doing both is at the low end. Dual-arch adds, in my estimate, 40–60% to hypervisor-core effort (not 100%), concentrated in vCPU/irqchip/IOMMU/platform code, and little to the VMM/control-plane effort. This replaces the "30–50%" figure in section 8.

### Gaps
- No measured shared-vs-arch-specific LOC split for Xen, KVM, Bao or AxVisor was retrieved; the "about half" figure is my estimate.
- No published experience report comparing Arm-first vs x86-first bring-up order was found.
- AxVisor's actual trait/crate boundaries (axvcpu, axvm, x86_vcpu, arm_vcpu, axaddrspace, axdevice) were not confirmed from the repo page.
