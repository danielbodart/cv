# Daniel Worthington-Bodart -- Portfolio of Recent Work

A comprehensive catalogue of open source contributions, side projects, and technical achievements (primarily 2025-2026; updated October 2026). This document captures the raw material for job applications -- interesting facts, technical depth, and what was actually built.

---

## Agent Safety Stack

Five open-source projects, named after letterpress parts, that let AI coding agents (Claude Code, Codex) run on NixOS with real tool access but no credentials, no root and no unmediated egress. **685 commits, 348 releases** across Zig, Go and Nix; runs daily across my own machines via nix-config.

**Threat model:** the agent is untrusted, including one steered by prompt injection. Every approval it might need -- a credential, an outbound request, an SSH signature, a sudo -- moves out of its reach to a host daemon that decides by policy or asks a person.

**How it fits together:** **chase** picks a sandbox tier for the checkout and launches the agent in a **flong** container. Its traffic is steered by the kernel to **frisket**, which injects credentials and enforces policy. Anything only a person should approve goes to **quoin**, which asks through **galley**.

### flong -- Ephemeral Rootless Containers
**Repo:** [danielbodart/flong](https://github.com/danielbodart/flong) | **Language:** Zig, Nix | **153 commits, 112 releases**

Rootless, daemonless containers declared with NixOS's own `containers.<name>` option; each run gets a fresh in-memory overlay that is discarded on exit.
- **~10ms launch vs ~120ms for Docker**, with no images to pull -- packages are the Nix store paths already on the host
- No capabilities, `no_new_privs`, nested user namespaces -- not even control over its own network namespace
- 4-filter seccomp stack compiled at build time, proven byte-for-byte stricter than systemd-nspawn's (and found a 64-bit-compare hole in nspawn's own filter)
- Single static, libc-free Zig launcher with NixOS VM tests and "proof" builds that fail if a security property doesn't hold

### frisket -- Credential-Injecting Egress Proxy
**Repo:** [danielbodart/frisket](https://github.com/danielbodart/frisket) | **Language:** Go, Nix | **186 commits, 79 releases**

The sandbox holds only a placeholder token; frisket swaps in the real credential on the wire, checks each request against policy and logs everything.
- **Kernel steering (TPROXY), not proxy variables** -- tools that ignore proxies have nowhere else to go
- DNS allowlist with rebinding protection; private and metadata ranges always refused
- Per-session, name-constrained CA for TLS termination; the key never leaves the daemon
- Rules by method and path, git operation, GraphQL mutation and SSH command -- each request allowed, refused or held for a person
- "Record" mode proposes the minimal grant an agent actually needed

### chase -- Agent Policy for NixOS
**Repo:** [danielbodart/chase](https://github.com/danielbodart/chase) | **Language:** Go, Nix | **228 commits, 131 releases**

Decides which sandbox a checkout gets, which apps run in it, and which credentials they use.
- 5 tiers (host, ops, trusted, work, strict) chosen by checkout path -- the one thing an agent can't forge
- **5,400+ API operations** (Cloudflare, GitHub, Hugging Face) plus 394 Google Cloud APIs classified read/write/guarded from vendor API descriptions
- Per-project grants signed with a TPM key; an unsigned or edited grant does nothing
- Apps for claude, codex, git, gcloud, rootless Docker, SSH and dev shells, each with scoped state

### quoin -- Host Gatekeepers
**Repo:** [danielbodart/quoin](https://github.com/danielbodart/quoin) | **Language:** Go, Zig | **85 commits, 12 releases**

Small daemons, each its own system user, guarding what only a person should approve: an SSH key, root, and a project's rules.
- **TPM-backed SSH agent:** keys never leave the chip; signs silently for a human's terminal, asks about an agent's `git push`
- **sudo gate:** a sudo plugin written in Zig -- typed sudo is untouched, an agent's sudo waits for human approval
- **Remote approval:** `ssh -A host sudo` approved by your own TPM key on your own screen; works on macOS too
- Formal threat model backed by spikes in NixOS VMs

### galley -- Stacking Approval Dialogs
**Repo:** [danielbodart/galley](https://github.com/danielbodart/galley) | **Language:** Go, GJS (GTK 4) | **33 commits, 14 releases**

A zenity drop-in that puts every question in one shared queue window, so bursts of approvals from parallel agents don't each steal focus.
- Full zenity command-line compatibility, verified against zenity over thousands of invocations
- ~3ms per question once warm; a killed caller withdraws its question
- Hardened against spoofed answers (peer credential checks, no exported answer actions)

---

## Graph Database & Agent Memory

### mogwai-db -- TinkerPop 4 Graph Database on SQLite
**Repo:** [bodar/mogwai-db](https://github.com/bodar/mogwai-db) | **Language:** TypeScript (Bun) | **2,501 commits, 306 releases**

A TinkerPop 4 graph database with SQLite as its engine. Any unmodified TinkerPop client talks to it in Gremlin over the standard wire. One graph is one SQLite database, so per-user, per-tenant or per-agent graphs are isolated by design and cost almost nothing when idle.

**Technical highlights:**
- **Compiles each whole Gremlin traversal into one SQL statement** -- no step-by-step interpreter; anything it can't compile is refused, never answered wrongly
- **Four runtimes from one core:** in the browser (SQLite WASM on OPFS), Cloudflare Durable Objects (scales to zero), single-file binaries for Linux/macOS/Windows, and Docker
- **~82% of the official TinkerPop conformance suite** via the unmodified client, enforced as a CI ratchet within an 8-level test ladder (including property-based testing)
- **Stable plans without statistics:** join order fixed at compile time took a 1-hop lookup from ~9.8s to ~19ms
- Full-text search, graph algorithms, cross-graph federation and CouchDB-style replication -- all in SQL; built largely through parallel Claude Code agent workflows

**Direction: agent memory** (design stage, not yet shipped)

Positions mogwai-db as per-agent memory that is physically isolated and free when idle. Thesis: *"the most efficient agent-memory API is not an API -- it's a query language plus a code sandbox, running next to the data."*
- MCP server over Gremlin with typed schema discovery, rather than a bag of tools
- Graph model for short-term conversation, long-term entity knowledge and reasoning traces
- Fused retrieval at the edge: embed, nearest-neighbour search, graph expansion, ranked by recency, salience and similarity

---

## Side Projects

### Capsper -- Local Voice Dictation and Meeting Transcription
**Repo:** [danielbodart/capsper](https://github.com/danielbodart/capsper) | **Language:** Zig, C, Objective-C, TypeScript, Nix | **410 commits, 254 releases**

Fully local push-to-talk dictation for Linux and macOS: hold CapsLock, speak, release -- text is typed into the focused window. No cloud, no GPU required. Also records and transcribes both sides of a meeting.

**Technical highlights:**
- **Single Zig binary** for the whole pipeline: keyboard interception, audio capture, mel spectrogram, streaming RNNT inference (NVIDIA Nemotron Speech 600M), detokenisation and keystroke injection
- **Efficient on-device inference:** ~8x realtime on CPU at ~1.4 cores only while speaking (Linux, ONNX Runtime INT8); 93% Apple Neural Engine utilisation, ~15.5x realtime (macOS, CoreML)
- **Meeting capture:** virtual PipeWire sink records near and far sides into one stereo Opus file with a WebVTT transcript -- speakers labelled by construction, WebRTC echo cancellation, VAD-gated inference
- **Robust streaming decoder:** guards against RNNT stall and runaway-token failure modes, validated by a regression suite of real recordings with coverage/WER thresholds
- **Production delivery:** Sigstore-signed releases, atomic self-update with crash rollback, NixOS flake with NixOS and home-manager modules, built-in web console

**Spin-off projects:** [nemotron-speech-600m-onnx](https://huggingface.co/danielbodart/nemotron-speech-600m-onnx), [nemotron-speech-600m-coreml](https://huggingface.co/danielbodart/nemotron-speech-600m-coreml), [ten-vad-ggml](https://huggingface.co/danielbodart/ten-vad-ggml) (see Hugging Face Model Ports), and [fulcio-codesign](https://github.com/danielbodart/fulcio-codesign) -- a Zig binary for Sigstore/Fulcio macOS code signing via Security.framework SPI.

---

### Zerocast -- Low-Latency Application Sharing
**Repo:** [danielbodart/zerocast](https://github.com/danielbodart/zerocast) | **Language:** C, Zig, TypeScript, Objective-C | **192 commits, 124 releases**

Remote application and terminal sharing with GPU-direct capture, hardware encode and peer-to-peer WebRTC streaming to a browser. ~200us prepare + 1.6ms encode at 1080p30.

**Technical highlights:**
- **Zero CPU copy pipeline:** GPU capture -> hardware AV1/HEVC encode -> WebRTC; pixels never leave the GPU
- **Cross-platform capture:** NVIDIA (NvFBC), Wayland (embedded wlroots compositor + VA-API), macOS (ScreenCaptureKit + VideoToolbox + virtual displays)
- Forked libdatachannel to add capture-time and playout-delay RTP extensions for true end-to-end latency measurement and zero jitter buffer
- Multi-cursor collaboration and terminal sharing (PTY to xterm.js) over WebRTC data channels
- Cloudflare Durable Objects signalling with WebSocket Hibernation (no cost while streaming)

---

### Talebrary -- Interactive Fiction Platform
**Repo:** danielbodart/talebrary (private) | **Language:** TypeScript | **558 commits** | [talebrary.com](https://talebrary.com/)

2,000+ interactive fiction games playable in the browser, with AI-generated cover art and suggested actions. Built on Cloudflare Workers, D1, R2 and Workflows.

**Technical highlights:**
- Single codebase runs on Bun locally and Cloudflare Workers in production -- no framework, just function composition
- Portable durable-workflow abstraction matching Cloudflare's Workflows API
- AI image pipeline with a 3-step model fallback chain, plus rate limiting and AI spend caps
- **LLM eval suite built from scratch:** scorers for JSON validity, schema match, tree structure and vision-based judging across multiple models
- In-browser player running WasiGlk interpreters in a Web Worker; spin-off [dullahan](https://github.com/bodar/dullahan) is a headless text-adventure engine on JSR

---

### WasiGlk -- Interactive Fiction Interpreters in WebAssembly
**Repo:** [bodar/wasiglk](https://github.com/bodar/wasiglk) | **Language:** Zig, TypeScript, C | **127 commits**

Runs classic interactive fiction interpreters in the browser by compiling them to WebAssembly with Zig.

**Technical highlights:**
- **17 C interpreters compiled to wasm32-wasi** (Glulxe, TADS 2/3, Hugo, Alan, Level9, Magnetic and more)
- **67-87% smaller binaries than Emscripten** (Glulxe 1.68MB -> 222KB)
- Full Zig implementation of the Glk I/O layer, including graphics and transcript record/replay
- JSPI for native browser async instead of Asyncify
- TypeScript client published to JSR as `@bodar/wasiglk`

---

## Hugging Face Model Ports

Published under [huggingface.co/danielbodart](https://huggingface.co/danielbodart) -- taking research-grade models and making them deployable on specific hardware.

### nemotron-speech-600m-onnx
**[HF](https://huggingface.co/danielbodart/nemotron-speech-600m-onnx)** | **[GitHub](https://github.com/danielbodart/nemotron-speech-600m-onnx)** | 212 downloads

NVIDIA's Nemotron Speech 600M streaming ASR exported to ONNX.
- 4 precision variants: FP32, FP16, INT8 dynamic, INT8 static (2.3GB -> 799MB encoder)
- Static INT8 made to work with **warm-cache streaming calibration** -- naive calibration produced all-blank output

### nemotron-speech-600m-coreml
**[HF](https://huggingface.co/danielbodart/nemotron-speech-600m-coreml)** | **[GitHub](https://github.com/danielbodart/nemotron-speech-600m-coreml)** | 34 downloads

The same model converted to CoreML for Apple Silicon.
- **93% of encoder ops on the Apple Neural Engine**, ~15.5x realtime
- 3-level numerical validation against the PyTorch reference

### ten-vad-ggml
**[HF](https://huggingface.co/danielbodart/ten-vad-ggml)** | **[GitHub](https://github.com/danielbodart/ten-vad-ggml)**

The only GGML implementation of TEN-VAD (voice activity detection).
- ONNX-to-GGML converter plus a complete Zig inference implementation (STFT, conv, LSTM)
- Verified against the native library to within 0.001

---

## ML Experiments

### Masked Diffusion Text -- Discrete Text Diffusion from Scratch
**Repo:** [danielbodart/masked-diffusion-text](https://github.com/danielbodart/masked-diffusion-text) | **Language:** Python

Fine-tunes ModernBERT-large to generate text by iterative unmasking, inspired by [MDLM](https://arxiv.org/abs/2406.07524). Because generation is non-autoregressive, it can work in reverse -- given an answer, generate a valid question.
- Cosine, log-linear and Fibonacci unmasking schedules with Gumbel sampling
- Trained on synthetic arithmetic reasoning (90k examples)

---

## Open Source Libraries

### bodar.ts -- TypeScript Library Ecosystem
**Repo:** [bodar/bodar.ts](https://github.com/bodar/bodar.ts) | **543 commits** | Published to JSR | [dataflow.bodar.com](https://dataflow.bodar.com/)

TypeScript successors to my long-running Java libraries, plus new work:
- **@bodar/dataflow** -- reactive dataflow for plain HTML, inspired by Observable: add one attribute to `<script>` tags and they form a live dependency graph. Works at build time, at the edge or in a service worker. Includes its own JS AST walker, JSX compiler and scope analyser
- **@bodar/totallylazy** -- functional programming: composable predicates, transducers, parser combinators, lazy evaluation
- **@bodar/lazyrecords** -- type-safe SQL from functional predicates; PostgreSQL, SQLite and DuckDB (native and WASM)
- **@bodar/jsx2dom** -- JSX straight to native DOM calls, in browser, edge, server and tests
- **@bodar/yadic** -- tiny lazy dependency injection container

### Security Finding
- **Mosaic DuckDB Go Server:** found and reported 3 vulnerabilities -- an unwired function blocklist, a validation bypass and cross-tenant data leaks ([uwdata/mosaic#958](https://github.com/uwdata/mosaic/issues/958))

---

## Third-Party Contributions

### Pull Requests (20+)

**Merged (7):**
| Repository | PR | What |
|---|---|---|
| testcontainers/testcontainers-node | [#977](https://github.com/testcontainers/testcontainers-node/pull/977) | Major refactor: replaced callbacks with async, separated timeout/log concerns, 20% less code |
| cloudflare/workers-types | [#70](https://github.com/cloudflare/workers-types/pull/70) | Fixed latitude/longitude type from `number` to `string` matching runtime behavior |
| asdf-community/asdf-deno | [#39](https://github.com/asdf-community/asdf-deno/pull/39) | Windows bash (MSYS) support |
| adaltas/node-csv-docs | [#65](https://github.com/adaltas/node-csv-docs/pull/65) | Fixed import example to use public API |
| anthropics/claude-code | [#17057](https://github.com/anthropics/claude-code/pull/17057) | Trunk-based development workflow (later extracted to plugin) |
| apache/tinkerpop | [#3511](https://github.com/apache/tinkerpop/pull/3511) | Export the GraphBinary serializers from `gremlin/io` (needed by mogwai-db) |
| mike-lischke/antlr4ng | [#109](https://github.com/mike-lischke/antlr4ng/pull/109) | Fix DFA state-table hash collisions (found via mogwai-db's Gremlin parser) |

**Open/In Review (8):**
| Repository | PR | What |
|---|---|---|
| ggml-org/whisper.cpp | [#3677](https://github.com/ggml-org/whisper.cpp/pull/3677) | Streaming VAD: `whisper_vad_detect_speech_no_reset()` + explicit state reset to preserve LSTM temporal continuity across 32ms chunks |
| ggml-org/whisper.cpp | [#3661](https://github.com/ggml-org/whisper.cpp/pull/3661) | Fix misleading VAD timing log (cumulative labeled as per-call) |
| ggml-org/whisper.cpp | [#3660](https://github.com/ggml-org/whisper.cpp/pull/3660) | Cross-attention accessor for AlignAtt streaming -- expose `[n_tokens x n_audio_ctx x n_heads]` tensor |
| chrender/libfizmo | [#3](https://github.com/chrender/libfizmo/pull/3) | WASI platform guards to compile Z-machine interpreter to WebAssembly |
| ZimNovich/mxn-jsx-ast-transformer | [#1](https://github.com/ZimNovich/mxn-jsx-ast-transformer/pull/1), [#2](https://github.com/ZimNovich/mxn-jsx-ast-transformer/pull/2), [#3](https://github.com/ZimNovich/mxn-jsx-ast-transformer/pull/3) | TemplateLiteral support, AST range support, member method factory support |
| observablehq/framework | [#2032](https://github.com/observablehq/framework/pull/2032) | Custom code extensions in markdown (positive engagement from Observable team) |
| BohdanTkachenko/waydriver | [#71](https://github.com/BohdanTkachenko/waydriver/pull/71) | Nix package for the MCP server on a private D-Bus session -- fixes at-spi needing systemd, a 25s xdg-desktop-portal GDK stall, and at-spi2-registryd corrupting the MCP stdio stream |

**Other Notable:**
| Repository | PR | What |
|---|---|---|
| ufal/SimulStreaming | [#33](https://github.com/ufal/SimulStreaming/pull/33) | Fix server crash on missing keys + bound unbounded token growth in 8hr+ sessions |
| hotdata-dev/duckdb_extension_parser_tools | [#12](https://github.com/hotdata-dev/duckdb_extension_parser_tools/pull/12) | Fix subquery extraction for multi-tenant access control validation (+139 lines) |
| miniupnp/miniupnp | [#714](https://github.com/miniupnp/miniupnp/pull/714) | IPv4 address mask with IPv6 disabled at runtime (OpenWRT) |
| GameServerManagers/LinuxGSM | [#4467](https://github.com/GameServerManagers/LinuxGSM/pull/4467) | Fix Minecraft Bedrock monitor constantly restarting server |

### Bug Reports & Design Feedback (17+)

**Security findings:**
- [uwdata/mosaic#958](https://github.com/uwdata/mosaic/issues/958) -- DuckDB Go Server: function-blocklist dead code + exec bypasses validation + cross-tenant data leak

**Architecture/spec influence:**
- [WebAssembly/WASI#799](https://github.com/WebAssembly/WASI/issues/799) -- Questioned why WASI HTTP needs 2 separate handler interfaces (citing fetch, http4k, Finagle). WASI spec authors agreed; simplification planned for 0.3.

**Infrastructure bugs:**
- [cloudflare/containers#147](https://github.com/cloudflare/containers/issues/147) -- WebSocket connections don't renew activity timeout (container sleeps despite active connections)
- [oven-sh/bun#25551](https://github.com/oven-sh/bun/issues/25551) -- `Bun.build()` ESPIPE on second call with `--watch` (confirmed, fix created by another contributor)

**Other notable reports across:** whisper.cpp, SimulStreaming, DuckDB parser_tools, Bun, jsdelivr, acorn, sniffnet, testcontainers-node, LinuxGSM, miniupnp, nom (Rust parser), crates.io, parcel, esbuild, BishopFox/raink, and the WASI spec.

**Technology breadth of contributions:** C, C++, Go, JavaScript, TypeScript, Python, Rust, Shell, WebAssembly/WASI

---

## Established Open Source Libraries (Java)

Functional programming for Java **before Java 8 Streams existed**, all published to Maven.

### TotallyLazy -- Functional Programming for Java
**Repo:** [bodar/totallylazy](https://github.com/bodar/totallylazy) | **318 stars, 32 forks**
- Lazy evaluation across iterables, sequences, strings, dates and numbers
- Persistent data structures (sets, maps, sorted maps, lists)
- Functors, applicatives, multi-method dispatch and pattern matching
- Infinite sequences and concurrent mapping
- Ported to Objective-C (OCTotallyLazy) and TypeScript

### UtterlyIdle -- REST Framework for Java
**Repo:** [bodar/utterlyidle](https://github.com/bodar/utterlyidle) | **96 stars, 18 forks**
- **~1ms startup**, configuration in code, no static state
- In-memory container for fast tests, plus Servlets, Jetty, Undertow and more
- One uniform API for client and server

### JCompilo -- Java Compiler and Build Tool
**Repo:** [bodar/jcompilo](https://github.com/bodar/jcompilo) | **41 stars**
- **20-80% faster** than other Java build tools; no build file needed
- Bytecode post-processing, including tail-call optimisation via `@tailrec`
- Zero-copy JAR creation

### YatSpec -- Self-Documenting Tests
**Repo:** [bodar/yatspec](https://github.com/bodar/yatspec) | **59 stars, 24 forks**
- Human-readable HTML specs generated from plain JUnit tests -- full IDE refactoring support
- **Automatic sequence diagrams** for microservice interactions

### Yadic, LazyRecords, ShavenMaven
- **[Yadic](https://github.com/bodar/yadic.java)** (12 stars) -- 12KB DI container, 1M resolves in 0.28s
- **[LazyRecords](https://github.com/bodar/lazyrecords)** (8 stars) -- LINQ for Java across SQL, Lucene, memory and more, with cross-source joins
- **[ShavenMaven](https://github.com/bodar/shavenmaven)** (31 stars) -- fast, transitive-free dependency resolver

---

## 10second.build -- Startup

**Website:** [10second.build](https://10second.build) | **GitHub:** [10secondbuild](https://github.com/10secondbuild)

A CI service bringing super-fast builds to everyone, built on JCompilo and Docker.
- Docker pipeline: checkout -> build -> release
- Automated SSL, DNS, bare-metal provisioning and containers
- Grew out of the 10-second build work at Sky (45 min -> 3 min), which won the **2012 Agile Award** and a **QCon talk**

---

## Cross-Language Ecosystem

The bodar libraries form a coherent, layered ecosystem ported across languages over 15+ years:

```
                    Java (2010s)              TypeScript (2018-now)        Other
                    -----------               -------------------         -----
FP Core:            totallylazy (318*)        @bodar/totallylazy (JSR)    OCTotallyLazy (Obj-C)
Data Access:        lazyrecords (8*)          @bodar/lazyrecords (JSR)    + DuckDB adapters
DI Container:       yadic.java (12*)          @bodar/yadic (JSR)
HTTP/REST:          utterlyidle (96*)         (http-handler pattern)      http-handler.rust
Testing/BDD:        yatspec (59*)             --
Build/Compiler:     jcompilo (41*)            --
Dep Resolution:     shavenmaven (31*)         --
Templates:          funclate                  --
Parsing:            lazyparsec                parser combinators in TS
Reactive UI:        --                        @bodar/dataflow (JSR)
JSX runtime:        --                        @bodar/jsx2dom (JSR)
```

**Publication platforms:** Maven (repo.bodar.com), JSR (jsr.io/@bodar), npm (@bodar/totallylazy), deno.land/x (bodar)

---

## Developer Tooling & AI Agents

### Dragoman -- Codex as a Native Claude Code Subagent
**Repo:** [danielbodart/dragoman](https://github.com/danielbodart/dragoman) | **Language:** TypeScript (Bun) | **171 commits, 34 releases**

A Claude Code plugin that runs OpenAI Codex as a background peer agent, with its approvals, progress and permissions feeling built into Claude -- and Claude's guardrails applying to Codex.
- **Bridges two protocols:** Codex's app-server JSON-RPC on one side, an MCP server on the other (run, status, steer, cancel, continue, review)
- **Native approvals:** Codex's approval requests appear as prompts in Claude's own UI instead of failing
- **Config compiler:** Claude's permission modes, allow/deny rules, sandbox paths and network domains compiled into an equivalent Codex policy
- Steer a run mid-flight, continue a thread, or run Codex's code review over a diff

### browser-agent-os -- WebAssembly Microkernel in the Browser
**Repo:** [danielbodart/browser-agent-os](https://github.com/danielbodart/browser-agent-os) | **Language:** Zig, TypeScript | exploratory

A browser-resident microkernel for capability-scoped, multi-process AI agents: WASM is the sandbox, Web Workers are processes, OPFS is the disk.
- WASI kernel in TypeScript, running in the browser or on Bun
- Zig shell with pipes and redirects, plus 16 Zig coreutils compiled to WebAssembly

### Claude Code Plugins
**Repo:** [danielbodart/claude-code-plugins](https://github.com/danielbodart/claude-code-plugins) | 3 stars | 76 commits
- **url-allowlist:** WebFetch limited to URLs found via search, blocking data exfiltration through query strings; red-team tested against 19 exfiltration vectors
- **commit-push-trunk:** trunk-based workflow with worktree support and CI monitoring
- **code-review-local:** parallel-agent code review for local, trunk-based work
- **statusline:** context usage, quota bars and effort level
- **graphical-sudo**, **web-researcher**, **notification-sound**

### Claude Code Sandbox
**Repo:** [danielbodart/claude-code-sandbox](https://github.com/danielbodart/claude-code-sandbox)

Settings-level deny rules blocking remote-mutating CLI commands and reads of credentials and secrets. The lightweight precursor to the [Agent Safety Stack](#agent-safety-stack).

---

## Infrastructure / Ops: Nix

### nix-config -- Every Machine in the House as One Flake
**Repo:** danielbodart/nix-config (private) | **Language:** Nix | **548 commits**

One flake declares 10 machines: NixOS desktop, laptop and server; a nix-darwin Mac; an OpenWrt router and 4 Wi-Fi access points; and a Raspberry Pi. One rebuild command applies system and home config on any of them, and any bad change is one reboot from rollback.
- **Every platform from one place:** NixOS, nix-darwin, OpenWrt images and Raspberry Pi OS images, with features (e.g. Capsper, the SSH signer) declared once across all of them
- **Hardware-rooted security:** Secure Boot, disk encryption unlocked by the TPM, and SSH and secrets keys that never leave the TPM
- **Offline bare-metal reinstall** from one encrypted USB key holding the machine's whole prebuilt system
- **Safe for agents:** Claude Code and its MCP servers pinned as Nix packages with read-only config, and the Agent Safety Stack wired in across every machine
- **Self-updating server:** upgrades itself hourly from trunk, waiting until nobody is playing

### Game Server Fleet -- Hands-Off Hosting for Me and My Kids

Self-hosted Minecraft Bedrock, Valheim and Hytale servers, built as a small open-source toolchain so they look after themselves. New game versions flow in automatically and nobody gets kicked mid-game. **Every tool runs both on Docker and natively on systemd, with a NixOS module in its flake.**
- **[Chandlery](https://github.com/danielbodart/chandlery):** game servers as Docker images (the tag *is* the game version) or as confined systemd/NixOS services built from the same image -- graceful stop, real health checks, a console either way; NixOS VM-tested
- **[Tidewaiter](https://github.com/danielbodart/tidewaiter)** (Go): updates Docker containers and NixOS systemd units only once no one is using them; health-gated swap with rollback
- **[Treesinger](https://github.com/danielbodart/treesinger)** (Go): Hytale sign-in broker -- sign in once and every server shares it; in Docker or socket-activated under systemd
- **[Portical](https://github.com/danielbodart/portical)** (Go, 26 stars): UPnP port forwarding from a single Docker label or systemd unit key
- **Hands-off pipeline:** hourly CI bumps game versions in the flake, the server rebuilds itself, and Tidewaiter swaps each game in once its players leave

### Nix Packaging
- **[nix-raspios-imagebuilder](https://github.com/danielbodart/nix-raspios-imagebuilder):** reproducible Raspberry Pi OS images built by Nix, with a read-only root and A/B upgrades that fall back automatically
- **[claude-desktop](https://github.com/danielbodart/claude-desktop) / chatgpt-desktop:** Nix flakes for the official Linux apps, pinned to the vendors' signed repos and self-updating
- **[glim](https://github.com/danielbodart/glim)** (Zig): Razer Blade keyboard backlight as a standard Linux LED, with no kernel module

---

## Key Metrics & Facts

**Agents, safety & infrastructure (Jul-Oct 2026):**
- **Agent Safety Stack:** 5 projects (flong, frisket, chase, quoin, galley), **685 commits and 348 releases in ~6 weeks**, ~41k lines Zig, ~128k Go, ~24k Nix
- **flong containers start in ~10ms vs Docker's ~120ms**; seccomp stack proven byte-for-byte stricter than systemd-nspawn's
- **5,400+ API operations** classified read/write/guarded for agent policy (Cloudflare, GitHub, Hugging Face) plus 394 Google Cloud APIs
- **mogwai-db:** 2,501 commits and 306 releases in under 4 months; ~82% of the official TinkerPop Gherkin suite (1,878/2,286); 4 runtimes from one core; 9.8s -> 19ms query plan fix
- **nix-config:** 10 machines (NixOS desktop/laptop/server, nix-darwin Mac, OpenWrt router + 4 APs, Raspberry Pi) in one flake, 548 commits in ~8 weeks; Secure Boot, TPM-sealed LUKS, TPM-resident SSH and age keys
- **Dragoman:** 171 commits, 34 releases bridging Codex app-server JSON-RPC to Claude Code MCP with native approval relay
- **Capsper:** 410 commits, 254 releases; 28x CPU reduction (28 -> 1.4 cores); meeting capture with 12.6 dB echo cancellation

**Recent (2025-2026):**
- **4 major side projects** built in ~10 weeks (capsper, zerocast, talebrary, wasiglk) -- all production-grade with automated releases
- **17 interpreters** rebuilt from x86 to WebAssembly, with 67-87% binary size reductions
- **3 model formats** mastered: ONNX (4 precision variants), CoreML (93% ANE), GGML (custom Zig implementation)
- **20+ PRs** across third-party repos spanning C, C++, Go, JS, TS, Python, Rust, Shell (incl. merged apache/tinkerpop and antlr4ng PRs)
- **17+ bug reports** including security vulnerabilities and WASI spec influence
- **254 automated releases** of capsper alone (CI/CD via GitHub Actions)
- **5+ difficult bugs/race conditions** solved (mmap corruption, HEVC POC reset, warm-cache quantization calibration, Bun ESPIPE, testcontainers race condition)
- **7 TypeScript packages** published to JSR (functional programming ecosystem)
- **~3,170 commits** across bodar org repos (2,501 mogwai-db + 543 bodar.ts + 127 wasiglk)
- **1 LLM eval suite** built from scratch (talebrary evals)
- **Languages used:** Zig, Go, C, C++, TypeScript, Nix, Python, Objective-C, Rust, Shell, Scala
- **Platforms:** Linux (Wayland, X11, evdev, PipeWire), macOS (CoreML, CoreAudio, ScreenCaptureKit, TCC), WebAssembly (WASI, JSPI), Cloudflare Workers (D1, R2, Durable Objects, Workflows, AI), NixOS / nix-darwin / OpenWrt, TPM 2.0, seccomp/user namespaces

**Established (2011-present):**
- **565+ combined GitHub stars** across bodar Java libraries
- **7 Java libraries** published to Maven, all in production use
- **1 QCon presentation** and **1 Agile Award** (2012 Most Valuable Agile Innovation)
- **1 startup** (10second.build) built on the library ecosystem
- **15+ year FP evolution** across Java -> Objective-C -> TypeScript -> Rust -> Zig
- **Build time reduction:** 45 minutes -> 3 minutes (Sky, production system)

---

## Narrative Themes

### Making Machines Safe for Agents
The through-line of 2026: if coding agents are going to act autonomously, the machine has to be built for it. Start with an OS that is reproducible, declarative and rollback-able (NixOS, every machine in one flake), then move every approval out of the agent's reach -- credentials swapped in on the wire by a host proxy (frisket), sandboxes chosen by unforgeable checkout path (chase) in ~10ms ephemeral rootless containers (flong), and anything only a person should approve (SSH signatures, sudo, project policy) held by TPM-backed host daemons (quoin) that ask through one stacking approval queue (galley). Agents are first-class users of the system, not a risk bolted onto it -- and the stack itself was built with them.

### Graphs as Agent Memory
mogwai-db compiles Gremlin to SQL so that a graph can be one SQLite database: in a browser tab, a Durable Object, a binary or a container. The direction is per-agent memory that is physically isolated and costs nothing when idle, queried through a language plus a code sandbox next to the data rather than a bag of tools.

### Agent-Driven Engineering at Scale
Not just using agents but engineering how they work: mogwai-db's 2,501 commits came through parallel Claude Code agent workflows under a strict CI ladder; Dragoman makes Codex a native peer inside Claude with the same guardrails; plugins, skills and Nix-pinned MCP servers make the tooling reproducible.

### Systems Programming Depth
Not surface-level: understands GPU memory models (DMA-BUF, CUDA interop, VA-API surfaces), codec internals (HEVC POC, AV1 intra refresh), Wayland protocol (wlroots, xkbcommon), and macOS platform internals (TCC, Security.framework SPI, CGVirtualDisplay private API).

### ML Inference Optimization (Not Training)
Focus on making models run efficiently on-device: static INT8 quantization with warm-cache calibration, CoreML ANE targeting at 93% utilization, GGML format porting with custom Zig inference, streaming ASR with LSTM state management.

### Full-Stack but Depth-First
TypeScript for web frontends and Cloudflare Workers, but the majority of effort is C/Zig systems code. Can go from GPU driver integration to JSX rendering in the same project.

### Upstream-First Mentality
Multiple PRs to Apache TinkerPop, antlr4ng, waydriver, whisper.cpp, libfizmo, testcontainers-node, Observable Framework, DuckDB tools. Forked wlroots and libdatachannel only when upstream changes were too specialized.

### Security Consciousness
Formal threat models for agent sandboxing (5 actor classes, trusted computing base, accepted limits), TPM 2.0 for disk unlock, SSH keys, age secrets and policy signing, Secure Boot with lanzaboote, seccomp parity proofs against systemd-nspawn (and a 64-bit-compare hole identified in its NETLINK_AUDIT rule), systematic red-team testing of AI agent security (19 exfiltration vectors), DuckDB Go Server security audit, multi-tenant access control analysis, Fulcio/Sigstore code signing.

### Functional Programming Heritage
15+ years of FP practice across languages: created TotallyLazy for Java (318 stars) before Java 8 Streams existed, built a compiler extension (JCompilo) that adds tail-call optimization via bytecode transformation, then ported the entire ecosystem to TypeScript. Deep understanding of: lazy evaluation, persistent data structures, parser combinators, transducers, applicative functors, pattern matching, and self-describing composable predicates. The same FP principles appear in the systems work -- the Zig code in capsper and zerocast uses similar compositional patterns.

### Presentation & Award
- **2012 Agile Award Winner:** "Most Valuable Agile Innovation" for The 10-Second Build
- **QCon London:** ["Crazy Fast Build Times, or When 10 Seconds Starts to Make You Nervous"](http://www.infoq.com/presentations/Crazy-Fast-Build-Times-or-When-10-Seconds-Starts-to-Make-You-Nervous) (slides and video)
- The work at Sky reduced build times from 45 minutes to 3 minutes, changed Sky's corporate policy to actively encourage open source development, and inspired the 10second.build startup
