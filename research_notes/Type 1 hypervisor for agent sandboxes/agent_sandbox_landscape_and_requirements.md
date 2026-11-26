# Agent-sandbox isolation landscape and requirements for a new hypervisor

Research date: 2026-11-26. Evidence labels used below: **[fetched]** = I read the primary page; **[snippet]** = taken from a search-result summary of the cited page, not read in full; **[vendor claim]** = self-reported, no independent measurement found; **[secondary]** = third-party write-up. Older data carries its year.

## 1. What do agent-sandbox providers use under the hood (as of 2026)?

### Takeaway
The market has converged on KVM microVMs (Firecracker first, Cloud Hypervisor second) with snapshot-restore as the "boot" path; gVisor remains at Google, Anthropic (claude.ai code execution) and Modal's default, but Modal added a Cloud Hypervisor-based VM runtime in October 2026 because agents need Docker, FUSE and real kernel features. Nobody surveyed runs a purpose-built type-1 hypervisor for agent sandboxes except Edera (Xen-derived); the rest are type-2-style VMMs on a full Linux/KVM host, or Hyper-V at Microsoft.

### Cited Findings

Provider-to-technology map:

| Provider | Isolation tech | Key facts | Evidence |
|---|---|---|---|
| E2B | Firecracker on KVM, one microVM per sandbox, own cgroup + netns | Templates are pre-booted VM snapshots (memory+disk+machine state) in object storage; create = restore, not kernel boot. Memory served lazily via `userfaultfd`. Rootfs is CoW overlay on read-only image. Pause stores memory+disk diff vs template. Fork checkpoints a live sandbox and starts up to 100 sandboxes from it. Template build ends with an "optimize pass" recording pages a boot touches. Requires "Linux with KVM"; Terraform for GCP/AWS/Azure | [fetched] [E2B infra README](https://raw.githubusercontent.com/e2b-dev/infra/main/README.md) |
| Fly.io Sprites (Jan 2026) | KVM micro-VMs (runs on Fly Machines, "OCI containers repackaged as KVM micro-VMs"); user code in an inner container inside the VM | Create in "a second or two" from pools of pre-staged empty Sprites; 100 GB durable root FS; object storage is source of truth (JuiceFS-like chunks + SQLite metadata via Litestream), sparse NVMe as read-through cache; checkpoint/restore only shuffles metadata ("a git restore, not a system restore"); auto-sleep when idle. No timings published in the post | [fetched] [Fly: Design & Implementation of Sprites](https://fly.io/blog/design-and-implementation/) |
| Fly Sprites (press) | Described as Firecracker-based | | [snippet] [DevClass](https://www.devclass.com/ai-ml/2026/01/13/flyio-introduces-sprites-lightweight-persistent-vms-to-isolate-agentic-ai/4079557) |
| Modal | gVisor default; new `runtime="vm"` built from Cloud Hypervisor (GA post dated 2026-10-01) | Modal added host filesystems, lazy image loading, memory bursting and snapshotting on top of Cloud Hypervisor. Reasons: agents need Docker stacks, local DBs, GUI environments, mobile simulators, kernel-level work; gVisor limits cited are Docker, FUSE, niche kernel features. Docker-in-sandbox supported (`docker:dind`). "Sub-second cold starts", "hundreds of thousands of concurrent Sandboxes", early customers launched >20M VMs. No VM boot/snapshot timings published; GPU and nested virt not mentioned for the VM runtime | [fetched] [Modal: VM sandboxes](https://modal.com/blog/vm-sandboxes-agent-computers) |
| Modal (gVisor path) | gVisor checkpoint/restore for memory snapshots; GPU snapshots via cuda-checkpoint (alpha) | Vendor: init-heavy functions start 3-10x faster from memory snapshots; docs warn the driver-level checkpoint tech is "still quite new" | [snippet] [Modal docs](https://modal.com/docs/guide/memory-snapshots), [Modal GPU snapshots blog, Jul 2025](https://modal.com/blog/gpu-mem-snapshots) |
| Vercel Sandbox | Firecracker microVM per sandbox, dedicated kernel | Snapshots (`sandbox.snapshot()`), start many sandboxes from one snapshot; persistent sandboxes snapshot on every session stop, keep 1-10; runs on Fluid compute with Active-CPU pricing | [snippet] [Vercel docs](https://vercel.com/docs/sandbox/concepts), [GA blog](https://vercel.com/blog/vercel-sandbox-is-now-generally-available) |
| Cloudflare Sandbox SDK / Containers | "each sandbox in its own VM"; VMM not named in the docs retrieved | Worker -> Durable Object -> container runtime; VM boundary is between sandboxes, not between sessions inside one | [snippet] [Cloudflare security docs](https://developers.cloudflare.com/sandbox/concepts/security), [architecture](https://developers.cloudflare.com/sandbox/concepts/architecture) |
| Blaxel | Firecracker microVM per sandbox, RAM-based root FS | Vendor: resume from standby in ~25 ms "even after weeks", scale to zero after ~5 s idle; standby snapshot = guest memory + device state + CPU registers + disk; standby data has "low durability" and can be lost on crash | [snippet, vendor claim] [Blaxel sandboxes](https://blaxel.ai/platform/sandboxes), [warm-sandbox blog](https://blaxel.ai/blog/how-to-keep-sandbox-warm) |
| Daytona | OCI containers (runc on default Docker path); stronger runtimes optional | Vendor: "sub-90ms sandbox creation". Independent study found Daytona's guest kernel string identical to host kernel and no seccomp BPF filter | [snippet] [Daytona isolation docs](https://www.daytona.io/docs/en/isolation); [fetched] [arXiv 2606.08433](https://arxiv.org/html/2606.08433v1) |
| Northflank | Kata Containers with Cloud Hypervisor; gVisor fallback where nested virt is unavailable (e.g. some GPU nodes) | Vendor marketing pages | [snippet] [Northflank blog](https://northflank.com/blog/how-to-spin-up-a-secure-code-sandbox-and-microvm-in-seconds-with-northflank-firecracker-gvisor-kata-clh), [GPU sandboxes](https://northflank.com/blog/gpu-sandboxes) |
| Morph Cloud (Infinibranch) | Hypervisor not disclosed | Vendor: snapshot, branch and restore whole environments in <250 ms | [snippet, vendor claim] [Morph docs](https://cloud.morph.so/docs/developers) |
| CodeSandbox (now Together) | Firecracker + userfaultfd memory CoW | 2022: clone a running VM in ~2 s; plain memory copy at 2-3 GB/s was too slow for 6 GB+ VMs; snapshot resume 200-300 ms via mmap; saving ~1 s per GB. Later: resume within 1 s after hibernating; compressed snapshot took 4 s vs 300 ms raw before their decompression work | [snippet] [CodeSandbox: cloning microVMs with userfaultfd](https://codesandbox.io/blog/cloning-microvms-using-userfaultfd), [DEV repost](https://dev.to/codesandboxio/how-we-clone-a-running-vm-in-2-seconds-39m3), [memory decompression post](https://codesandbox.io/blog/how-we-scale-our-microvm-infrastructure-using-low-latency-memory-decompression) |
| Replit | CoW "Bottomless" storage (2023) underpins agent checkpoints; dev Postgres sits on the same CoW FS | Claim that Replit rebuilt on Cloud Hypervisor in late 2025 is from a secondary site only, unconfirmed | [snippet] [Replit snapshot engine](https://replit.com/blog/inside-replits-snapshot-engine); [secondary] [braindetox](https://braindetox.kr/en/posts/replit_microvm_sandbox_engine.html) |
| Google GKE Agent Sandbox | gVisor; Pod Snapshots require gVisor and "do not work with microVMs" | Warm pool: 300 sandboxes/s per cluster, 90% of allocations within 200 ms (vendor); suspended-VM standby buffers refill the pool; Pod Snapshots unsupported on E2 machine types | [snippet] [Google Cloud blog](https://cloud.google.com/blog/products/containers-kubernetes/bringing-you-agent-sandbox-on-gke-and-agent-substrate), [GKE docs](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/machine-learning/agent-sandbox) |
| AWS Bedrock AgentCore Runtime | Dedicated microVM per session | Session header pins requests to the same microVM; on session end the microVM is terminated and memory sanitized; operations up to 8 hours. VMM not named in the pages retrieved | [snippet] [AWS docs: sessions](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-sessions.html), [how it works](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-how-it-works.html) |
| Azure Container Apps dynamic sessions | One Hyper-V sandbox per session | Pre-warmed session pool gives "milliseconds"/sub-second allocation; sandbox deleted after `cooldownPeriodInSeconds` idle | [snippet] [Microsoft Learn](https://learn.microsoft.com/en-us/azure/container-apps/sessions) |
| Anthropic claude.ai code execution | gVisor container on isolated server-side infra; ephemeral per-session FS | Post dated 2026-05-25 | [fetched] [How we contain Claude](https://anthropic.com/engineering/how-we-contain-claude) |
| Anthropic Claude Code (local) | Seatbelt (macOS) / bubblewrap (Linux); network via proxy outside the sandbox | OS sandbox cut permission prompts by 84% | [fetched] same; [snippet] [Claude Code sandboxing](https://anthropic.com/engineering/claude-code-sandboxing) |
| Anthropic Cowork (local) | Full VM on Apple Virtualization framework / Windows HCS, own Linux kernel, vsock channel | Only selected workspace mounted; credentials stay in host keychain, VM gets scoped revocable token | [fetched] [How we contain Claude](https://anthropic.com/engineering/how-we-contain-claude) |
| Anthropic Claude Code on the web | "isolated sandbox in the cloud", git through a proxy with a scoped credential | Isolation technology not named | [snippet] [Claude Code sandboxing](https://anthropic.com/engineering/claude-code-sandboxing) |
| OpenAI Codex cloud | "isolated OpenAI-managed containers" | Internet blocked in agent phase by default, allowed in setup phase; all egress via HTTP/HTTPS proxy with domain + method allowlist; secrets available only in setup and removed before agent phase | [snippet] [Codex internet access](https://developers.openai.com/codex/cloud/agent-internet), [Codex security](https://developers.openai.com/codex/agent-approvals-security.md) |
| Docker Sandboxes | Custom VMM (not Firecracker) calling Apple Hypervisor.framework, Windows Hypervisor Platform, or KVM directly; one microVM per agent session with a private Docker daemon | Docker publishes no boot number | [snippet] [Docker: Why MicroVMs](https://www.docker.com/blog/why-microvms-the-architecture-behind-docker-sandboxes/) |

- Independent cross-product study (May 2026) compared e2b (Firecracker), arrakis (Cloud Hypervisor), microsandbox (libkrun), gVisor, and daytona (runc) — confirming these as the engine classes actually in the field — [arXiv 2606.08433](https://arxiv.org/html/2606.08433v1)

### Inferences
- "Boot" in this market means "restore a pre-booted snapshot"; kernel boot time matters mainly for template builds. A new hypervisor should treat snapshot restore/clone as the primary create path.
- The two forces pulling vendors off gVisor are Linux-compatibility (Docker, FUSE, cgroups/netns inside the sandbox) and guest-kernel isolation. The force keeping gVisor is that it runs on ordinary cloud VMs with no nested virtualization and has mature whole-sandbox checkpoint/restore including GPU state.
- Persistence designs converge on: content-addressed/chunked CoW disk backed by object storage with a local NVMe cache, plus memory diffs against a shared base image. The hypervisor's storage and memory interfaces should make that cheap (lazy, chunk-addressable, shareable base).
- Several vendors run a container inside the microVM (Fly, Docker), so the unit is "VM as the security boundary, container as the packaging".

### Gaps
- Not researched or no primary source found: Runloop, GitHub Codespaces, Apple Containerization, ChatGPT code interpreter internals, Morph's hypervisor, AgentCore's and Cloudflare's VMM names. Widely assumed answers exist (e.g. Firecracker for AWS services) but I found no page stating them.
- Sprites' VMM: Fly's post says KVM micro-VMs; press says Firecracker; Fly's post does not name the VMM for Sprites.
- Most latency figures in comparison blogs conflict (E2B quoted from ~5-30 ms to ~150 ms; Modal 100-500 ms) and are unreliable.

## 2. Measured characteristics of the underlying VMMs/sandboxes and their limits for agent workloads

### Takeaway
Firecracker's published guarantees (<=125 ms to init, <=5 MiB VMM overhead, >95% bare-metal CPU) set the baseline; vendors get to tens of milliseconds only through snapshot restore, and Hyperlight's 1-2 ms applies to function-sized guests without a normal OS. Independent, like-for-like measurements of snapshot restore, fork and density across VMMs are scarce.

### Cited Findings
Firecracker
- Spec (on M5D.metal/M6G.metal, hyperthreading off): <=125 ms from `InstanceStart` to guest `/sbin/init` for 1 vCPU/128 MiB, minimal kernel+rootfs, serial off; VMM starts in <=8 CPU-ms (wall ~12 ms, range 6-60 ms); <=5 MiB VMM overhead for 1 vCPU/128 MiB; guest CPU >95% of bare metal; network 14.5 Gbps at 80% of a host core, 25 Gbps at 100%; storage 1 GiB/s at 70% of a core; +0.06 ms average latency. Several of these are marked "integration test pending". No creation-rate guarantee in the spec — [fetched] [Firecracker SPECIFICATION.md](https://raw.githubusercontent.com/firecracker-microvm/firecracker/main/SPECIFICATION.md)
- NSDI 2020 paper headline numbers: <5 MB overhead, <125 ms boot, up to 150 microVMs/s per host — [snippet] [paper copy](https://css.csail.mit.edu/6.566/2023/readings/firecracker.pdf), [summary](https://blog.acolyer.org/2020/03/02/firecracker/)
- 2020-era per-VM memory overhead comparison of ~3 MB Firecracker vs ~13 MB Cloud Hypervisor vs ~131 MB QEMU is reported by secondary reviews of the paper; not verified against the paper's figure — [secondary] [review](https://justinlinkk.github.io/blog/fire_cracker_review)
- QEMU `microvm` machine type measured at ~600 ms boot in an issue on the paper's data repo, ~2.5x worse than normal QEMU in that setup (2020, disputed configuration) — [snippet] [nsdi2020-data issue #2](https://github.com/firecracker-microvm/nsdi2020-data/issues/2)
- Unikraft's own benchmark says a stock Linux guest on raw Firecracker needs ~330 ms to reach userspace — [snippet, vendor claim] [Unikraft vs Firecracker](https://unikraft.com/blog/unikraft-vs-firecracker)
- Snapshot docs give no latency numbers; restore is fast because pages load on demand, create is slow because it writes memory synchronously and a full snapshot faults in all guest memory — [fetched] [Firecracker snapshot-support.md](https://raw.githubusercontent.com/firecracker-microvm/firecracker/main/docs/snapshotting/snapshot-support.md)

gVisor
- `getpid` microbenchmark from Ant Group's production write-up (2021): native 62 ns, native+KPTI 236 ns, runsc-KVM 830 ns, runsc-ptrace 6249 ns — [snippet] [gVisor blog, Dec 2021](https://gvisor.dev/blog/2021/12/02/running-gvisor-in-production-at-scale-in-ant/)
- KVM platform is fastest on bare metal; Systrap (default since mid-2023) is preferred inside a VM or without virtualization support — [snippet] [gVisor platform guide](https://gvisor.dev/docs/architecture_guide/platforms), [Systrap release](https://gvisor.dev/blog/2023/04/28/systrap-release/)
- Catalyzer-style restore on gVisor achieves 10s-100s of ms (as cited inside the REAP paper) — [snippet] [REAP, ASPLOS 2021](https://infoscience.epfl.ch/record/283308/files/asplos21-reap.pdf)

Hyperlight (Microsoft)
- Creates micro-VMs in 1-2 ms vs >120 ms for an optimized traditional VM (vendor, Nov 2024); a 0.0009 s figure is warm function-call latency, not cold start (Feb 2025) — [snippet] [Introducing Hyperlight](https://opensource.microsoft.com/blog/2024/11/07/introducing-hyperlight-virtual-machine-based-security-for-functions-at-scale/), [0.0009 s post](https://opensource.microsoft.com/blog/2025/02/11/hyperlight-creating-a-0-0009-second-micro-vm-execution-time/)
- Original Hyperlight guests have no system calls available; a 2026 Microsoft post on Nanvix POSIX support reports double-digit-ms cold starts for real applications — [snippet] [Microsoft Open Source blog](https://opensource.microsoft.com/blog/?p=98299)

Edera (Xen-derived type-1)
- Paper (Jan 2025, authors affiliated with Edera): v2 abstract says 2.14% slower CPU than Docker, faster memory, and +740 ms startup over Docker's 381 ms; v1 gave 0.9% slower CPU and ~3% faster syscalls. Versus gVisor/Kata/Firecracker: better CPU and memory, worse syscalls — [snippet] [arXiv 2501.04580](https://arxiv.org/abs/2501.04580)
- Vendor EKS benchmark: 8,927 vs runc 8,645 events/s (+3.3%); start 2.0 s vs runc 1.0-1.4 s — [snippet, vendor claim] [Edera blog](https://edera.dev/stories/faster-than-runc-benchmarking-edera-on-amazon-eks)

Unikraft Cloud
- Vendor: cold start <10 ms, 16.65 ms NGINX boot in a published example, stateful resume "in the order of a few milliseconds" and constant; runs on a large modified fork of Firecracker — [snippet, vendor claim] [Unikraft how it works](https://unikraft.com/how-it-works.md), [scale-to-zero docs](https://unikraft.com/docs/features/scale-to-zero)

Academic snapshot work
- REAP (ASPLOS 2021): records the working set with userfaultfd on first invocation and prefetches it; cuts cold-start delay 3.7x on average vs baseline Firecracker snapshots — [snippet] [REAP](https://infoscience.epfl.ch/record/283308/files/asplos21-reap.pdf)
- FaaSnap (EuroSys 2022): concurrent paging so the guest runs while the working set loads; up to 3.5x better function execution than REAP; tolerates working-set drift — [snippet] [FaaSnap](https://www.sysnet.ucsd.edu/~voelker/pubs/faasnap-eurosys22.pdf)

Practitioner measurement of shared-base memory CoW on Firecracker (Cortex, 2026): 4 GiB apparent base memory file had 49 MiB physically allocated; three guests showed 35% PSS/RSS against an ideal 33%. Memory content-addressed in 512 KiB chunks, disk in 16 MiB SHA-256 chunks served through a custom NBD server with NVMe LRU cache — [fetched] [Cortex: Snapshots, copy-on-write, and the economics of agent sandboxes](https://builders.cortex.io/blog/sandboxing-agents-part-2/)

### Inferences
- Targets a new hypervisor must at least match: <=125 ms cold kernel boot, <=5 MiB per-VM monitor overhead, >=95% CPU, and create-from-snapshot in the 25-300 ms range vendors already advertise. Beating vendor claims means restore/fork in low tens of ms with a real Linux guest.
- Hyperlight shows that VM creation itself can be ~1 ms when there is no guest OS to boot and no devices to set up; the cost in general-purpose microVMs is device/network setup and memory population, not VM entry.
- gVisor's ~3.5x-13x per-syscall cost (vs native with/without KPTI, 2021 data) is the price of not needing hardware virtualization; a hypervisor wins on syscall-heavy agent workloads (builds, package installs) but only if file and network I/O paths are fast.

### Gaps
- No primary measurements gathered for Cloud Hypervisor, crosvm, libkrun, Kata Containers or WASM runtimes (boot time, overhead, snapshot latency). I did not search these individually.
- No independent benchmark of snapshot-restore latency or fork fan-out across any VMM; all sub-100 ms restore figures are vendor claims.
- gVisor file-I/O overhead numbers were not retrieved.

## 3. What features do agent workloads specifically need?

### Takeaway
Agent sandboxes are long-lived, stateful, mostly idle "computers" rather than short functions: the cost model is dominated by idle memory and disk, so snapshot-to-zero, fast resume, CoW fork of memory+disk, and memory reclamation matter more than raw boot time. A full Linux environment (Docker, FUSE, nested namespaces) and controlled egress are now baseline expectations.

### Cited Findings
- Full Linux needed: Modal's customers needed Docker stacks, local databases and dev servers, GUI environments, mobile simulators and kernel-level work; one customer had to run sandboxes as bare processes because FUSE would not mount under gVisor; another uses cgroups and network namespaces inside sandboxes — [fetched] [Modal](https://modal.com/blog/vm-sandboxes-agent-computers)
- Docker-in-sandbox without host privileges is a headline feature of Docker Sandboxes (private daemon inside each microVM) — [snippet] [Docker](https://www.docker.com/blog/why-microvms-the-architecture-behind-docker-sandboxes/)
- Fork/branch: E2B forks a live sandbox into up to 100 copies; Vercel starts multiple sandboxes from one snapshot; Morph sells branching in <250 ms — [E2B](https://raw.githubusercontent.com/e2b-dev/infra/main/README.md), [Vercel](https://vercel.com/docs/sandbox/concepts), [Morph](https://cloud.morph.so/docs/developers)
- Checkpoint as a routine undo: Fly reports the pre-installed Claude Code "checkpoints aggressively without being asked"; Replit built agent checkpoints on CoW storage including the dev database — [Fly](https://fly.io/blog/design-and-implementation/), [Replit](https://replit.com/blog/inside-replits-snapshot-engine)
- Idle economics: Fly Sprites sleep automatically and "cost practically nothing while asleep"; Blaxel scales to zero after ~5 s idle with no memory charge in standby; Cortex's model shows 18 retained sessions with 6 active on 3 nodes, and reframes cost as vCPU + (shared base memory + dirty RAM) + (base disk + dirty disk) — [Fly](https://fly.io/blog/design-and-implementation/), [Blaxel](https://blaxel.ai/platform/sandboxes), [Cortex](https://builders.cortex.io/blog/sandboxing-agents-part-2/)
- Workload sizes used by a practitioner: coding agent 2 vCPU / 8 GiB RAM / 40 GiB disk; build agent 4 vCPU / 16 GiB / 120 GiB; a 16 vCPU/64 GiB host fits 9-24 agents before saturation without sharing; a 4 GiB guest running compilers, language servers, browsers and build daemons pulls several GiB into host RAM — [fetched] [Cortex](https://builders.cortex.io/blog/sandboxing-agents-part-2/)
- Session length: AgentCore supports operations up to 8 hours; Modal's gVisor sandboxes were reported capped at 24 hours (secondary); Fly Sprites are persistent with 100 GB durable disk — [AWS](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-how-it-works.html), [Fly](https://fly.io/blog/design-and-implementation/)
- Allocation throughput: GKE Agent Sandbox targets 300 sandboxes/s per cluster with P90 200 ms from a warm pool; Modal cites hundreds of thousands of concurrent sandboxes — [Google](https://cloud.google.com/blog/products/containers-kubernetes/bringing-you-agent-sandbox-on-gke-and-agent-substrate), [Modal](https://modal.com/blog/vm-sandboxes-agent-computers)
- Egress and credentials: Codex blocks internet in the agent phase by default, proxies all egress with domain+method allowlists, and strips secrets before the agent phase; Anthropic keeps git credentials and signing keys outside the sandbox and gives the sandbox scoped, revocable tokens — [OpenAI](https://developers.openai.com/codex/cloud/agent-internet), [Anthropic](https://anthropic.com/engineering/how-we-contain-claude)
- Filesystem sharing with fine-grained modes: Cowork mounts only the chosen workspace with read-only / read-write / read-write-no-delete modes, and must resolve symlinks before path validation — [fetched] [Anthropic](https://anthropic.com/engineering/how-we-contain-claude)
- Observability: VM isolation hid Cowork's guest from endpoint-detection tools; mitigation is pull-based OTLP log export, not live monitoring — [fetched] [Anthropic](https://anthropic.com/engineering/how-we-contain-claude)
- GPU: Modal's GPU sandboxes run on the gVisor runtime; Northflank falls back to gVisor for GPU nodes where nested virt is unavailable — [Modal](https://modal.com/resources/best-gpu-enabled-sandboxes-ai-agents), [Northflank](https://northflank.com/blog/gpu-sandboxes)
- Snapshot correctness needs: restoring one snapshot into several VMs duplicates identifiers, RNG state and tokens; VMGenID reseeds the kernel PRNG (Linux 5.18+) but application state is still duplicated; wall clock resumes from snapshot time unless `clock_realtime` is used — [fetched] [Firecracker snapshot docs](https://raw.githubusercontent.com/firecracker-microvm/firecracker/main/docs/snapshotting/snapshot-support.md)

### Inferences
Derived requirement list for a new hypervisor (my synthesis from the findings above; targets are set at or beyond current vendor claims):
1. Create-from-snapshot as the default path, P50 well under 100 ms and ideally ~10-25 ms to first guest instruction, with on-demand paging plus recorded working-set prefetch (REAP/FaaSnap style) built in rather than delegated to a userspace fault handler.
2. Native CoW fork of a running VM (memory + device state + disk) with fan-out of at least 100 and cost proportional to dirty pages; shared, deduplicated base memory across sibling VMs as a first-class object.
3. Pause-to-zero: full state to durable storage within seconds of idleness, zero host RAM/CPU held, resume in tens of ms; incremental (dirty-only) snapshots that are directly resumable without offline merging.
4. Memory elasticity: free-page reporting and hot-unplug that work together with snapshots, huge pages and device passthrough.
5. Density: per-VM fixed overhead in the low single-digit MiB; hundreds to thousands of mostly idle 2-8 GiB guests per host through sharing and reclamation.
6. Unmodified Linux guest with Docker, FUSE, cgroups v2, netns, and optionally nested virtualization.
7. Fork-aware identity: VMGenID/vmclock-style notification, per-clone network identity (MAC/IP/TAP remap) without per-VM host setup cost, clock correction.
8. Chunk-addressable block device with lazy fetch from object storage and local cache; snapshot/restore of disk as metadata operations.
9. Egress control and credential brokering at the VM boundary (per-VM allowlist proxy, secrets injected outside the guest).
10. Host-side introspection/telemetry hooks so security tooling is not blind to the guest.
11. GPU sharing/passthrough that does not disable snapshots or ballooning.
12. Runs where customers are: bare metal ideally, but also inside cloud VMs.

### Gaps
- No source quantified the distribution of agent session durations, idle ratios or fork fan-out in production; the Cortex figures are a simulator and one team's sizing.
- No data found on demand for time-travel debugging/record-replay beyond checkpoint/restore.

## 4. What threat model applies?

### Takeaway
Two distinct threats: (a) classic multi-tenant escape, where untrusted code attacks the host kernel or VMM, and (b) the agent as a confused deputy, where prompt injection uses legitimate capabilities to exfiltrate data. A hypervisor addresses (a) directly; (b) is contained only by egress control and keeping credentials outside the guest, which the hypervisor's I/O boundary can enforce.

### Cited Findings
- Anthropic's risk categories are user misuse, model misbehaviour and external attackers including prompt injection; model-layer defences are probabilistic and cannot stand alone — [fetched] [How we contain Claude](https://anthropic.com/engineering/how-we-contain-claude)
- Internal red-team (Feb 2026): a phishing-style prompt got Claude to read `~/.aws/credentials` and POST them out in 24 of 25 tries; only egress controls and filesystem boundaries held — same source
- Allowlist exfiltration: a malicious workspace file made Claude upload files to an attacker's account through an allowed destination (api.anthropic.com) using the attacker's API key; lesson: an allowlist is a capability grant, not just a destination filter — same source
- Their most consequential incident involved their custom proxy, not gVisor or seccomp — same source
- Benchmarks: ~0.1% prompt-injection success on a single attempt but ~5-6% after 100 adaptive attempts (Gray Swan, Claude Opus 4.7); auto-mode classifier lets ~17% of overeager actions through — same source
- Container escapes: runc CVE-2025-31133, CVE-2025-52565, CVE-2025-52881 disclosed 2025-11-05 (mount/procfs races leading to host root); fixed in later runc releases. 13 of runc's 16 historical CVEs are escape-class — [snippet] [CNCF overview](https://www.cncf.io/blog/2025/11/28/runc-container-breakout-vulnerabilities-a-technical-overview/); [fetched] [arXiv 2606.08433](https://arxiv.org/html/2606.08433v1)
- MicroVM VMM CVEs, per one preprint (not cross-checked against vendor advisories): Firecracker CVE-2026-5747 (virtio-pci out-of-bounds write, CVSS v4 8.7, escape potential, no public end-to-end exploit) and CVE-2026-1386 (jailer symlink arbitrary host file overwrite, 6.0); Cloud Hypervisor CVE-2026-45782 (virtio-block use-after-free, 8.9) — [fetched] [arXiv 2606.08433](https://arxiv.org/html/2606.08433v1)
- 24-month CVE counts to 2026-05-20 in that study: runc 4 (all escape), gVisor 3 (none escape), Firecracker 2, Cloud Hypervisor 2, libkrun 0, Linux kernel ~3,500 — same source
- Host-interface measurements in that study: seccomp allowlist of 55 syscalls for E2B's Firecracker, 84 for gVisor; distinct host syscalls seen under light load: arrakis/Cloud Hypervisor 5, E2B 13, microsandbox 22, gVisor 33, Daytona 57; reachable kernel LPE/escape primitives out of 14: gVisor 5, E2B 7, microsandbox 11, Daytona 11, arrakis 12 — same source
- Patch lag is a first-order risk: E2B self-hosted pinned Firecracker for 399 days with CVE-2026-5747 unpatched in the default pin; arrakis pinned Cloud Hypervisor v44.0 for 471+ days; gVisor fixes landed 165-458 days before CVE publication — same source
- Snapshot-specific risks: Firecracker trusts snapshot files (only a CRC64 against accidental corruption); operators must add authentication and encryption; cloned VMs share entropy and tokens — [fetched] [Firecracker snapshot docs](https://raw.githubusercontent.com/firecracker-microvm/firecracker/main/docs/snapshotting/snapshot-support.md)

### Inferences
- The recent escape-class bugs in Rust VMMs are in virtio device emulation and the jailer, i.e. the device model and host-side setup remain the attack surface even with memory-safe code. A new design should minimise and deprivilege device backends and treat snapshot files as untrusted input.
- With a KVM-based VMM the whole Linux host kernel stays in the TCB (~3,500 kernel CVEs in 24 months); this is the main structural argument for a small type-1 core.
- Shared base memory and page dedup across tenants create side-channel exposure; sharing should be scoped to a single tenant/template.

### Gaps
- No KVM guest-to-host CVE list for 2025-2026 retrieved; no data on side-channel attacks specific to shared-snapshot memory.
- The Firecracker/Cloud Hypervisor CVE identifiers above come from a single preprint and should be verified against project advisories before being quoted.

## 5. Documented pain points with Firecracker/KVM approaches

### Takeaway
The recurring problems are around state, not boot: memory restore depends on userspace page-fault handlers, diff snapshots are not directly resumable, clones need manual identity/network fix-up, snapshots are tied to identical hardware/host kernel, and memory reclamation and GPU passthrough are recent, preview-grade and mutually exclusive.

### Cited Findings
- Memory restore: copying memory at 2-3 GB/s was too slow for 6 GB+ VMs, pushing CodeSandbox to userfaultfd (2022); saving cost ~1 s/GB — [snippet] [CodeSandbox](https://codesandbox.io/blog/cloning-microvms-using-userfaultfd)
- Every serious user builds their own UFFD handler and chunk store: E2B (lazy pages via userfaultfd), Cortex (`UFFDIO_CONTINUE`/`UFFDIO_COPY` against a shared base file, custom NBD server); Cortex notes memory "lacked an NBD-style boundary" — [E2B](https://raw.githubusercontent.com/e2b-dev/infra/main/README.md), [Cortex](https://builders.cortex.io/blog/sandboxing-agents-part-2/)
- Naive snapshots are too big to move: 16 GiB RAM + 128 GiB disk = ~144 GiB per snapshot in Cortex's first design; copying a 40 GiB ext4 image per agent was unworkable — [fetched] [Cortex](https://builders.cortex.io/blog/sandboxing-agents-part-2/)
- Diff snapshots are still "developer preview"; most are not resumable and must be merged into a base in order with `snapshot-editor`; dirty tracking adds CPU overhead and reduces the benefit of huge pages; the `mincore` alternative needs swap disabled — [fetched] [Firecracker snapshot docs](https://raw.githubusercontent.com/firecracker-microvm/firecracker/main/docs/snapshotting/snapshot-support.md)
- Huge pages: explicit 2M hugetlbfs requires the UFFD backend; THP effectiveness limited with UFFD — same source
- Portability: snapshots must be resumed on identical hardware and software configuration, with one narrow, non-recommended exception — same source
- Networking after restore: connectivity not guaranteed, vsock connections closed, packet loss possible; TAP names must be overridden per clone — same source
- Restore latency is high on cgroups v1 hosts; snapshots taken very early in boot may crash on resume; memory file must be kept for the lifetime of the resumed VM — same source
- Memory elasticity arrived late: free page hinting/reporting was tracked for v1.14.0 and reporting is labelled developer preview, not for production; virtio-mem hotplug exists with correctness fixes still landing around v1.17 — [snippet] [Firecracker CHANGELOG](https://github.com/firecracker-microvm/firecracker/blob/main/CHANGELOG.md), [issue #5474](https://github.com/firecracker-microvm/firecracker/issues/5474)
- GPU: E2B wrote that Firecracker lacked PCIe so GPU passthrough was impossible (pre-2026); VFIO passthrough is now a tracked in-progress feature, MSI-X only, and a proposed implementation rejected balloon and memory hotplug when passthrough is enabled — [snippet] [E2B Firecracker vs QEMU](https://e2b.dev/blog/firecracker-vs-qemu), [Firecracker issue #5679](https://github.com/firecracker-microvm/firecracker/issues/5679), [PR #6245](https://github.com/firecracker-microvm/firecracker/pull/6245)
- Needs KVM: E2B requires "Linux with KVM"; Northflank falls back to gVisor where nested virtualization is unavailable; gVisor's own docs steer users to Systrap inside VMs; GKE Pod Snapshots work only with gVisor, not microVMs — [E2B](https://raw.githubusercontent.com/e2b-dev/infra/main/README.md), [Northflank](https://northflank.com/blog/gpu-sandboxes), [gVisor](https://gvisor.dev/docs/architecture_guide/platforms), [GKE docs](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/machine-learning/agent-sandbox)
- Operational fragility of layered checkpoint stacks: a Fly forum thread reports Sprites left unrecoverable after a checkpoint restore (user report) — [snippet] [Fly community](https://community.fly.io/t/unrecoverable-sprites-after-checkpoint-restore-failed-to-start-overlay-service-tree/27918)
- Create cost outside the VMM: a Fly Machine `create` "can take over a minute", so Sprites pre-stage pools of empty VMs — [fetched] [Fly](https://fly.io/blog/design-and-implementation/)

### Inferences
Gaps a purpose-built hypervisor could close:
- In-hypervisor lazy/CoW memory from a shared, content-addressed base (no per-VM userspace fault round trips), with directly resumable incremental snapshots.
- A native "fork VM" primitive that handles identity (VMGenID, clock, MAC/IP) and network attachment in one step.
- Snapshot format that is portable across CPU models and hypervisor versions, authenticated and encrypted by default.
- Memory reclamation, huge pages, snapshots and device passthrough designed to compose rather than exclude one another.
- Pre-created VM shells/network endpoints so allocation is constant-time.
- A type-1 design removes the general-purpose host kernel from the TCB but worsens the deployment problem: it needs bare metal (or must itself run nested), which is exactly what currently pushes some vendors back to gVisor.

### Gaps
- No quantified data found on the nested-virtualization performance penalty for Firecracker on GCP/AWS/Azure, or on per-VM network (TAP/netns) setup cost.
- CodeSandbox's detailed memory-decompression post returned HTTP 403; only search-snippet figures are included.

## 6. Existing purpose-built hypervisors or type-1 efforts for agent sandboxes / serverless

### Takeaway
Edera is the only type-1 effort found that explicitly targets agent isolation, and its published numbers show near-native runtime but slow starts (around 1-2 s) and no advertised snapshot/fork story; Hyperlight and Unikraft Cloud chase millisecond starts by shrinking the guest rather than serving full Linux dev environments. The combination "type-1 core + full Linux guest + millisecond snapshot/fork" appears unoccupied.

### Cited Findings
- Edera: built on Xen; a Rust control plane (the open-source `krata` repo is titled "The Edera Hypervisor") around a Xen-design type-1 core that one source describes as MISRA C; marketed as giving each agent its own Linux kernel; a KVM backend was announced in 2026 for infrastructure where Xen is unavailable — [snippet] [Why Edera built on Xen](https://edera.dev/stories/why-edera-built-on-xen-a-secure-container-foundation), [krata](https://www.github.com/edera-dev/krata), [Virtualization Review, Mar 2026](https://virtualizationreview.com/articles/2026/03/13/why-edera-is-betting-on-micro-vms.aspx)
- Edera performance: see section 2 (CPU within ~1-2% of Docker; startup +740 ms over Docker's 381 ms in the paper; 2.0 s vs 1.0-1.4 s on EKS, vendor) — [arXiv 2501.04580](https://arxiv.org/abs/2501.04580), [Edera EKS benchmark](https://edera.dev/stories/faster-than-runc-benchmarking-edera-on-amazon-eks)
- Hyperlight: Rust library VMM for function-sized guests on KVM/Hyper-V, 1-2 ms VM creation, Apache 2.0, applied to CNCF Sandbox; Hyperlight Wasm (Mar 2025) runs Wasm components in the micro-VM; Nanvix work (2026) adds POSIX with double-digit-ms cold starts — [snippet] [Microsoft](https://opensource.microsoft.com/blog/2024/11/07/introducing-hyperlight-virtual-machine-based-security-for-functions-at-scale/), [DevClass](https://www.devclass.com/development/2025/03/27/microsoft-releases-experimental-hyperlight-wasm-micro-vms-that-run-wasm-apps/1623450), [Microsoft 2026](https://opensource.microsoft.com/blog/?p=98299)
- Unikraft Cloud: unikernel guests on a heavily modified Firecracker fork; claims <10 ms cold start and millisecond stateful scale-to-zero resume; its comparison table says raw Firecracker lacks scale-to-zero-with-resume, which is a framing claim since Firecracker has snapshot/restore — [snippet, vendor claim] [Unikraft vs Firecracker](https://unikraft.com/blog/unikraft-vs-firecracker), [how it works](https://unikraft.com/how-it-works.md)
- Docker built its own cross-platform VMM for agent sandboxes rather than adopt Firecracker — [snippet] [Docker](https://www.docker.com/blog/why-microvms-the-architecture-behind-docker-sandboxes/)
- Modal built its VM runtime by extending Cloud Hypervisor rather than Firecracker — [fetched] [Modal](https://modal.com/blog/vm-sandboxes-agent-computers)
- An independent study notes the strongest combination it looked for, a microVM engine with a continuous public fuzzer, is unoccupied among the products it examined — [fetched] [arXiv 2606.08433](https://arxiv.org/html/2606.08433v1)

### Inferences
- Edera demonstrates that a type-1 approach can reach near-native steady-state performance, but its start times are 10-100x slower than what agent-sandbox vendors advertise; start/restore/fork latency is where a new type-1 design must differentiate.
- Edera adding a KVM backend suggests that a type-1-only deployment model limits adoption on public clouds; a new hypervisor needs a credible answer for running on rented cloud instances.
- Vendors choosing to build or fork VMMs (Docker, Modal, Unikraft, Fly's storage stack) indicates stock Firecracker is not sufficient on its own for this workload.

### Gaps
- NanoVMs/Nanos, Ubicloud, Bottlerocket-style minimal hosts, crosvm and Kata-based offerings were not researched; no claims recorded.
- Whether Edera offers snapshot/restore or fork was not established.
- SEUSS, Potemkin/SnowFlock, Mitosis, Fireworks and On-demand-fork papers were not retrieved; no numbers recorded for them.
