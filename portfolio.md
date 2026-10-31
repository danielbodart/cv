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
**Repo:** [bodar/mogwai-db](https://github.com/bodar/mogwai-db) | **Language:** TypeScript (Bun) | **2,501 commits, 306 releases** (Jul-Oct 2026)

A TinkerPop 4 graph database that uses SQLite as its engine and runs natively in the browser, at the edge, on bare metal and in Docker. Any unmodified TinkerPop 4 client talks to it in Gremlin over the standard HTTP/GraphBinary wire. Built for many small-to-medium isolated graphs (per-user, per-tenant, per-agent) where one graph is one SQLite database, so isolation comes free and an idle graph costs roughly nothing. Pre-alpha, designed in the open.

**Technical highlights:**
- **Compiles each whole Gremlin traversal into one parameterised SQL statement** (a CTE chain) instead of interpreting it a step at a time. Pipeline: ANTLR parser generated from TinkerPop's own `Gremlin.g4` -> own IR -> strategy normalisation -> typed Stream lowering -> SQL, encoded to GraphBinary only at the root. Writes become `INSERT ... SELECT ... RETURNING` chains, so statement count scales with steps, not rows
- **Four runtimes from one core:** browser (official SQLite WASM on OPFS in a Worker behind a Service Worker, so an unmodified client works inside the tab), Cloudflare Durable Objects (one DO = one graph, created on first access, scales to zero), single `bun build --compile` binaries for Linux/macOS/Windows on arm64/x64, and a multi-arch Docker image. One contract suite runs against every backend over the real wire
- **Conformance:** 100% of the 2,395 canonical Gremlin traversals parse; **1,878 of 2,286 official TinkerPop Gherkin scenarios (~82%)** pass through the unmodified `gremlin` JS client, enforced as a ratchet that can only go up. Anything it can't compile is refused with a clear error -- never a plausible wrong answer
- **Eight-level test ladder:** SQL-contract tests, a Gherkin addendum of 120 feature files, property-based testing (differential oracle plus metamorphic laws, seeded from HEAD), and an L8 run of the whole suite under `PRAGMA reverse_unordered_selects` that catches hidden row-order dependence, gated at zero failures in CI
- **Plans that stay stable without statistics:** join order and index seeks fixed at compile time, because Durable Object SQLite refuses `PRAGMA analysis_limit`. A mid-size graph's 1-hop filtered lookup went from **~9.8s to ~19ms**, identical with or without `sqlite_stat1`; `EXPLAIN QUERY PLAN` assertions forbid full scans
- **Bounded vs unbounded `repeat()`:** the TinkerPop corpus asks for 2.5x10^15 traversers over 808 vertices. A bounded `times(n)` is unrolled at compile time with traverser counts compressed per step; an unbounded one becomes a recursive CTE
- **Search and analytics in SQL:** FTS5 trigram full-text index behind `tinker.search` and the TextP predicates; graph algorithms compiled as set-based SQL passes (PageRank, ArticleRank, HITS, betweenness/closeness/harmonic centrality, weak and strong connected components, peer-pressure clustering, k-core, node similarity, triangle count, shortest path)
- **Distributed features:** cross-graph `federate()` pushdown and CouchDB-style replication (revisions, by-seq change feed, tombstones, resumable pull/push, conflict preservation) across Bun, Cloudflare and the browser
- **Built for agents to drive:** graph lifecycle is REST on the same `/gremlin/{graph}` path; the server describes itself through OpenAPI 3.1 (`/openapi.json`, interactive `/docs`)
- ~300ms cold start and ~1ms warm queries for the compiled binary; parsing is 2.7% of compile time
- **Agent-driven development at scale:** built through parallel Claude Code agent workflows (hundreds of worktree branches, ~650 agent-authored commits) under a strict CI gate: type check, lint, architecture/bind/SQL hygiene checks and the full test ladder

**Direction: agent memory** (design and phased plan; not yet shipped)

A 400-line design doc positions mogwai-db as the engine for agent memory that is physically isolated per agent and costs nothing when idle. Thesis: *"the most efficient agent-memory API is not an API -- it's a query language plus a code sandbox, running next to the data."*
- An MCP server over the Gremlin endpoint with exact typed schema and enum discovery from the catalog, bound parameters, chunked streaming and honest errors -- a better version of Apache's official `gremlin-mcp`
- A three-part memory model (short-term conversation, long-term entity knowledge graph with typed relationships kept typed, reasoning traces) expressed as a graph schema plus query recipes, not a bag of ~20 tools
- On Cloudflare, a Code Mode sandbox with Vectorize and Workers AI for fused retrieval: embed -> KNN -> graph expansion -> server-side ranking by recency, salience and similarity, with decay/salience scoring later pushed into the compiled SQL
- One SQLite database per agent (Durable Object facets under a per-customer Workers for Platforms boundary), costed at ~$0.02-0.03 per tenant per month against $45-125/month floors for incumbents (Neo4j, Zep, Weaviate, Mem0)

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

A developer-focused remote application and terminal sharing tool with GPU-direct capture, hardware encode, and P2P WebRTC streaming to a browser viewer. Measured as low as ~200us prepare + 1.6ms encode at 1080p30.

**Technical highlights:**
- **Zero CPU copy pipeline:** NvFBC GPU texture capture -> CUDA interop (zero-copy) -> NVENC hardware encode (AV1/HEVC) -> WebRTC P2P. Pixels never leave the GPU.
- Dual codec: AV1 (preferred, RTX 40-series+) with HEVC fallback (GTX 950+), auto-detected at startup
- Multi-platform capture:
  - Linux/NVIDIA: NvFBC + headless Xorg (per-app isolated X server)
  - Linux/Wayland: embedded wlroots-based headless compositor + VA-API encode (Intel/AMD) with DMA-BUF frame capture and damage tracking
  - macOS: ScreenCaptureKit + VideoToolbox HEVC + CGVirtualDisplay (in-process, eliminating IPC)
- Forked [wlroots](https://github.com/danielbodart/wlroots) to filter CCS modifiers from GBM allocation for VA-API compatibility
- Forked [libdatachannel](https://github.com/danielbodart/libdatachannel) adding `abs-capture-time` RTP extension (true capture-to-render latency measurement) and `playout-delay` extension (zero jitter buffer)
- Multi-cursor collaboration: SVG overlay with per-viewer colored cursors. Binary wire protocol over unreliable/unordered WebRTC data channels (UDP semantics)
- Terminal sharing: PTY over WebRTC data channel to xterm.js with 256KB replay buffer and asciinema recording
- Cloudflare Worker signaling: Durable Objects with WebSocket Hibernation API (zero cost during active streaming). STUN for ~85% of connections, Cloudflare TURN as fallback
- Remote input injection: XTEST on Linux, CGEvent on macOS, with platform-independent keycode mapping
- Latency telemetry: end-to-end, server, decode, jitter buffer, render, and network breakdown in viewer stats panel

---

### Talebrary -- Interactive Fiction Platform
**Repo:** [danielbodart/talebrary](https://github.com/danielbodart/talebrary) | **Language:** TypeScript | **558 commits** | repo private, site behind sign-in since Aug 2026

A web application serving 2,000+ interactive fiction games playable in the browser, with AI-generated cover art and illustrations. Built on Cloudflare Workers + D1 + R2.

**Technical highlights:**
- Dual-runtime architecture: single codebase runs on both Bun (local dev) and Cloudflare Workers (production) via dependency injection. Universal HTTP type: `Http = (request: Request) => Promise<Response>` -- no frameworks, just function composition
- Server-side JSX rendering with custom `@bodar/jsx2dom` + linkedom, slot-based template system (named slots + src slots with recursive internal fetch)
- Durable Workflow abstraction: portable workflow pattern (`DirectRunner` for Bun/tests, `CloudflareWorkflowRunner` for production) matching Cloudflare's `WorkflowStep.do()` API
- AI cover art pipeline: 3-step fallback chain (flux-2-klein-9b style transfer -> scene-enhanced transfer -> Leonardo Phoenix default)
- In-browser IF player: custom elements (`<interactive-fiction>`, `<buffer-window>`, `<user-input>`) running wasiglk interpreters in a Web Worker
- Custom LLM eval suite (`evals/`): evaluation framework for AI features with scorers for JSON validity, schema matching, tree structure, and vision-based judging. Runs against multiple Cloudflare AI models, computes per-model averages and latencies
- D1 database with materialized avg_rating, denormalized gamelinks, automated quarterly sync from IFDB via GitHub Actions
- Solved SQLITE_TOOBIG by storing workflow-generated images in R2 instead of Cloudflare Workflows SQLite persistence
- **Jul-Aug 2026:** LLM "enrich" endpoint with a few-shot prompt and its own eval harness, D1-based rate limiting and AI spend caps, fixed CloudflareWorkflowRunner hanging under concurrent bursts
- Spin-off: [dullahan](https://github.com/bodar/dullahan) (`@bodar/dullahan` on JSR) -- headless TypeScript port of text-engine returning structured engine state instead of DOM output

---

### WasiGlk -- Interactive Fiction Interpreters in WebAssembly
**Repo:** [bodar/wasiglk](https://github.com/bodar/wasiglk) | **Language:** Zig, TypeScript, C | **127 commits**

A platform for running interactive fiction interpreters in the browser via WebAssembly. Compiles 17 different C-based interpreters to wasm32-wasi using Zig's build system.

**Technical highlights:**
- **17 interpreters rebuilt from x86 to WASM:** AdvSys, Agility, Alan2, Alan3, Fizmo, Git, Glulxe, Hugo, JACL, Level9, Magnetic, Plus, Scare, Scott, TADS 2, TADS 3, Taylor
- **Dramatic binary size reductions vs Emscripten:** Glulxe 87% smaller (1.68MB -> 222KB), Hugo 83% smaller, TADS2 83% smaller (3.9MB -> 669KB)
- Full Zig implementation of the GLK I/O layer (RemGlk JSON protocol) for browser communication
- JSPI (JavaScript Promise Integration) for async I/O instead of Asyncify -- smaller binaries, better performance
- Forked [libfizmo](https://github.com/chrender/libfizmo) to add WASI platform guards for the Z-machine interpreter
- Unicode case conversion with full mapping tables (186 toLower + 204 toUpper ranges)
- TypeScript client library published to JSR as `@bodar/wasiglk`
- Converted 1500+ lines of Python regtest tooling to a single Bun TypeScript script
- **Interpreter toolchain migration: Rust -> Zig** -- switched from Emscripten (C->JS) to Zig (C->WASM) for cross-compilation
- **Jul 2026:** graphics (Blorb images; Hugo and Scare via data-URIs), window layout tree, `.glktra` transcript record/replay, agt2agx converter, single-file zip story delivery

---

## Hugging Face Model Ports

All published under [huggingface.co/danielbodart](https://huggingface.co/danielbodart). These demonstrate model conversion and optimization engineering -- taking research-grade models and making them deployable on specific hardware.

### nemotron-speech-600m-onnx
**[HF](https://huggingface.co/danielbodart/nemotron-speech-600m-onnx)** | **[GitHub](https://github.com/danielbodart/nemotron-speech-600m-onnx)** | 212 downloads

NVIDIA's Nemotron Speech 600M (FastConformer RNNT) exported to ONNX with 4 precision variants:

| Variant | Encoder Size | Target Hardware |
|---------|-------------|-----------------|
| FP32 | 2.3 GB | Any (reference) |
| FP16 | 1.2 GB | NVIDIA GPU |
| INT8 Dynamic | 799 MB | Intel CPU (VNNI/AMX) |
| INT8 Static | 799 MB | NVIDIA GPU + Intel CPU (QDQ) |

**Key insight:** INT8 static quantization required **warm-cache streaming calibration** -- naive approach with zero-initialized caches produced all-blank tokens because calibration gave wrong activation scales. Solution: sequential chunk processing carrying cache state forward, 6-chunk warmup per file.

### nemotron-speech-600m-coreml
**[HF](https://huggingface.co/danielbodart/nemotron-speech-600m-coreml)** | **[GitHub](https://github.com/danielbodart/nemotron-speech-600m-coreml)** | 34 downloads

Same model converted to CoreML for Apple Silicon:
- **93% of encoder ops run on the Apple Neural Engine** (1486/1597 ops)
- **~15.5x realtime inference speed** on Apple Silicon
- 3-level numerical validation against PyTorch reference
- Documented non-contiguous stride issue: CoreML output arrays may have ANE alignment padding requiring stride-aware copy

### ten-vad-ggml
**[HF](https://huggingface.co/danielbodart/ten-vad-ggml)** | **[GitHub](https://github.com/danielbodart/ten-vad-ggml)**

TEN-VAD (Voice Activity Detection) converted to GGML format -- described as "the only GGML implementation of TEN-VAD":
- 296 KB model (21 FP32 tensors)
- Complete Zig reference implementation: STFT + mel spectrogram + pitch extraction, separable convolution + LSTM + dense
- LSTM gate reordering (ONNX i/o/f/c -> PyTorch i/f/g/o)
- Verified identical outputs vs native `libten_vad.so` (max delta < 0.001)

---

## ML Experiments

### Masked Diffusion Text -- Discrete Text Diffusion from Scratch
**Repo:** [danielbodart/masked-diffusion-text](https://github.com/danielbodart/masked-diffusion-text) | **Language:** Python | March 2025

An experiment in discrete text diffusion -- fine-tuning ModernBERT-large to generate text through iterative unmasking, inspired by [MDLM (Masked Diffusion Language Models)](https://arxiv.org/abs/2406.07524).

**What it does:** Treats masked language modelling as a diffusion process. Training randomly masks tokens at any ratio; inference iteratively unmasks the most confident predictions first via a cosine schedule. Because generation is non-autoregressive (all positions simultaneously), the model can work in reverse -- given an answer, generate a valid question.

**Technical details:**
- Fine-tunes ModernBERT-large on synthetic arithmetic reasoning (90k examples of integer addition)
- Cosine, log-linear, and Fibonacci unmasking schedules
- Gumbel sampling for position selection
- Immutable state container with masking operations
- An early exploration that informed later work on LLM eval suites and model optimisation

---

## Open Source Libraries

### Dataflow -- Reactive Data Flow for HTML
**Package in [bodar/bodar.ts](https://github.com/bodar/bodar.ts)** | Published to JSR

A reactive dataflow library inspired by Observable Framework but grounded in HTML instead of Markdown. An extremely lightweight alternative to React.

**Technical highlights:**
- Add a single `data-reactive` attribute to `<script>` tags -- the engine topologically sorts all reactive scripts and builds a dependency graph
- Runtime-flexible: reactivity can be added at build time (static), as middleware on server/edge, or as a service worker in the browser
- **Contains the Javascript AST Walker** -- complete ESTree-compatible walker with enter/leave callbacks, skip/remove/replace operations. Mutable, small, fast.
- **Contains the JSX AST Compiler** -- transforms JSX AST into regular JS AST (createElement calls). Fast, bug-free, per-spec.
- **Contains the Javascript Scope AST Analyser** -- full lexical scope analysis tracking var/let/const/function/class/import/param/catch bindings, var hoisting to function scope, destructuring patterns, and all unresolved references. 100% strict spec compliance.
- DuckDB WASM integration with cross-filter flights demo (10M records) using Mosaic vgplot

### bodar.ts Monorepo -- TypeScript Library Ecosystem
**Repo:** [bodar/bodar.ts](https://github.com/bodar/bodar.ts) | **543 commits** | All published to JSR | [dataflow.bodar.com](https://dataflow.bodar.com/)

TypeScript ports and spiritual successors to long-running Java libraries. Development timeline:
- **Oct 2025 (69 commits):** Systematic JSR publishing (88%+ score), full transducer implementation in 7 phases, parser combinators
- **Nov 2025 (112 commits):** Created dataflow reactive framework from scratch, SharedAsyncIterable, HTMLTransformer, JSX support, advanced curry with full TypeScript type system
- **Dec 2025 (153 commits -- peak):** Bundling, import maps, memory leak hunting (rewrote combineLatest to avoid Promise.race leaks), SVG/Canvas/Web Audio tutorials, switched to `tsgo` (TypeScript Go compiler)
- **Jan 2026 (71 commits):** LazyRecords SQLite (Bun native), DuckDB (native + WASM), transactions, Mosaic vgplot with 10M-record cross-filter demo
- **Oct 2026:** dataflow PositionalJSX (blocks keep their DOM across re-runs), build moved to TypeScript 7 native `tsc`, lazyrecords typed bind-safe SQL AST, and a Claude Code plugin marketplace shipping a dataflow skill

Packages:

- **@bodar/totallylazy** -- Comprehensive FP library: composable predicates, transducers, parser combinators (complete JSON grammar with JSDoc custom type support), comparators, lazy evaluation
- **@bodar/lazyrecords** -- Type-safe SQL query builder bridging FP with SQL. Converts functional predicates/transducers into parameterized queries. PostgreSQL, SQLite, and DuckDB support
- **@bodar/yadic** -- Lightweight dependency injection container with lazy initialization via property getters
- **@bodar/jsx2dom** -- Thin adapter converting JSX/TSX to native DOM calls. Works in browser, edge, server, and tests (linkedom) without global namespace pollution. Added full SVG support (extending the library)
- **@bodar/dataflow** -- (Described above)
- **@bodar/lazyrecords-duckdb / lazyrecords-duckdb-wasm** -- DuckDB integrations

### Extending Third-Party Libraries
- **jsx2dom:** Added full SVG support to enable server-side rendering of SVG elements
- **Mosaic DuckDB Go Server:** Found and reported 3 security vulnerabilities -- function-blocklist flag parsed but never wired up, exec command bypasses schema validation, cross-tenant data leaks via shared preagg schema ([uwdata/mosaic#958](https://github.com/uwdata/mosaic/issues/958))

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

A coherent ecosystem of Java libraries created from ~2011 onwards, pioneering functional programming patterns in Java **years before Java 8 Streams existed**. All published to Maven (repo.bodar.com).

### TotallyLazy -- Functional Programming for Java
**Repo:** [bodar/totallylazy](https://github.com/bodar/totallylazy) | **318 stars, 32 forks** | Apache 2.0

A comprehensive FP library inspired by Clojure's collection library and ML-family naming conventions (Standard ML, OCaml, F#, Scala, Haskell):
- Lazy evaluation for Iterable, Iterator, Arrays, Char Sequences, Dates, Numbers
- Persistent data structures: PersistentSet, PersistentMap, PersistentSortedMap, PersistentList
- Functors and Applicative Functors
- Runtime multi-method dispatch and pattern matching
- Concurrent map (`mapConcurrently`) for background thread distribution
- Infinite sequences: `primes()`, `fibonacci()`, `powersOf(3)`, `iterate(increment, 1)`
- Tail-call optimisation (via JCompilo bytecode post-processing)
- Also ported to Objective-C (OCTotallyLazy)
- **Historical significance:** brought Haskell/ML-style FP to Java before `java.util.stream` was added in Java 8 (2014). The v1.x branch targets Java 7+.

### UtterlyIdle -- REST Framework for Java
**Repo:** [bodar/utterlyidle](https://github.com/bodar/utterlyidle) | **96 stars, 18 forks**

A REST library inspired by JSR-311 (JAX-RS) with distinctive design:
- **~1ms startup time** (vs seconds for Spring Boot)
- Configuration in code (no XML), no static state (fully testable)
- Multiple container support: Servlets, Jetty, SimpleWeb, Java 6 HttpServer, Undertow, **In-Memory** (for tests)
- Uniform client/server API using composition over inheritance
- DSL can bind HTTP methods directly to 3rd-party Java classes
- Request/Response objects can be `new`'d up, `toString`'d, and parsed

### JCompilo -- Advanced Java Compiler and Build Tool
**Repo:** [bodar/jcompilo](https://github.com/bodar/jcompilo) | **41 stars, 3 forks**

A build tool AND compiler extension:
- **20-80% faster** than all other Java build tools (the foundation for the 10-second build)
- **Bytecode post-processing** -- modifies compiled output
- **Tail-recursion call optimisation** via `@tailrec` annotation and bytecode transformation
- **Zero-copy Jar creation**
- Dependency resolution via ShavenMaven (Pack200 support, parallel downloads, no transitive dependencies)
- 100% convention (no build file needed) or customizable in Java code

### YatSpec -- Self-Documenting Tests
**Repo:** [bodar/yatspec](https://github.com/bodar/yatspec) | **59 stars, 24 forks**

A BDD test framework generating human-readable HTML reports from pure JUnit tests:
- No separate specification files (unlike Cucumber, Concordion, Fit)
- Full IDE refactoring support -- tests are just Java
- **Automatic sequence diagram generation** for microservice architectures
- Given/When/Then with automatic capturing
- Tabular data tests, cross-test links, "Interesting Givens" highlighting

### Yadic -- Tiny DI Container
**Repo:** [bodar/yadic.java](https://github.com/bodar/yadic.java) | **12 stars, 2 forks**

A radically minimal dependency injection container:
- **Only 12KB** in size
- **1 million resolves in 0.28 seconds**
- **Only 2 main functions** -- extreme simplicity
- Built-in decorator pattern support
- Also available in .NET and Ruby versions

### LazyRecords -- LINQ for Java
**Repo:** [bodar/lazyrecords](https://github.com/bodar/lazyrecords) | **8 stars, 3 forks**

A uniform functional API for data access across multiple backends:
- Supported: MemoryRecords, STMRecords (Software Transactional Memory), LuceneRecords, SqlRecords, SimpleDBRecords, XmlRecords
- Clojure-inspired Keywords (typed symbols) and Definitions (schemas)
- Can join across different data sources (SQL + Lucene, etc.)

### ShavenMaven -- Dependency Resolver
**Repo:** [bodar/shavenmaven](https://github.com/bodar/shavenmaven) | **31 stars**

Multiple repository types (mvn, s3, http, file, jar-within-jar), Pack200 compression (10x faster downloads), parallel downloads, zero transitive dependencies. Used by JCompilo.

---

## 10second.build -- Startup (2015-2016)

**GitHub org:** [10secondbuild](https://github.com/10secondbuild) | **Website:** [10second.build](https://10second.build)

A CI/build service startup aiming to bring super-fast Java builds to everyone, built around JCompilo and Docker:
- API written in Java using the bodar stack (UtterlyIdle, TotallyLazy)
- Docker-based pipeline: checkout -> build (JCompilo) -> release stages
- Integrated with SSL provisioning, DNS, bare metal provisioning (Packet), containerisation (Hyper)
- Related to the **2012 Agile Award Winner** -- "Most Valuable Agile Innovation (The 10-Second Build)"
- **QCon presentation:** ["Crazy Fast Build Times, or When 10 Seconds Starts to Make You Nervous"](http://www.infoq.com/presentations/Crazy-Fast-Build-Times-or-When-10-Seconds-Starts-to-Make-You-Nervous)
- The build time reduction (45 minutes -> 3 minutes for a core system at Sky) that inspired the startup

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
**Repo:** [danielbodart/dragoman](https://github.com/danielbodart/dragoman) | **Language:** TypeScript (Bun) | **171 commits, 34 releases** (Aug-Sep 2026)

A Claude Code plugin that runs OpenAI Codex as a background peer agent, so its approvals, progress and permissions feel built into Claude. The official Codex plugin hard-codes `approvalPolicy: never` and a fixed sandbox, so work outside the sandbox just fails and none of Claude's guardrails reach Codex. Dragoman's rule is *mirror Claude, don't reinvent it*.

**Technical highlights:**
- **Two protocols bridged:** talks to `codex app-server` over JSON-RPC (TypeScript bindings generated from Codex's protocol schema) and exposes itself to Claude as an MCP server (`codex_run`/`status`/`steer`/`cancel`/`continue`/`review`), plus a skill and a dedicated subagent
- **Native approval relay:** Codex approval requests become async MCP `elicitation/create` prompts in Claude's own UI, using Codex's own decision ids, so Codex escalates under your rules instead of failing
- **Config compiler:** compose (merge Claude's 4 settings layers by Claude's own rules) -> compile (pure `mirror` to a Codex policy) -> provision (isolated per-run `CODEX_HOME`, never touching `~/.codex`)
- Each Claude permission mode (plan/default/acceptEdits/auto/dontAsk/bypass) maps to a Codex approval policy, reviewer and sandbox scope. Allow/deny Bash rules compile to Codex execpolicy (fail-closed); sandbox filesystem and network domains to permission profiles and Codex's network proxy
- **Heartbeat filter:** collapses Codex's ~80-type notification stream into a few milestones -- a pure function, tested without a socket
- Steer a run mid-flight, continue on the same thread, or run Codex's dedicated code review over a diff
- Ships as a ~280KB single-bundle plugin with automated marketplace releases

### browser-agent-os -- WebAssembly Microkernel in the Browser
**Repo:** [danielbodart/browser-agent-os](https://github.com/danielbodart/browser-agent-os) | **Language:** Zig, TypeScript | **20 commits** (May-Jul 2026, exploratory)

A browser-resident microkernel for capability-scoped, multi-process AI agents: WASM is the sandbox, Web Workers are processes, OPFS is the disk. Environment-agnostic WASI Preview 1 kernel in TypeScript (Bun and browser hosts), Zig `sh.wasm` shell (pipes, redirects, REPL) via custom `proc_spawn`/`proc_join`/`fd_pipe` syscalls, 16 Zig coreutils compiled to wasm32-wasi, xterm.js terminal, JSPI/Atomics transports, headless-Chrome CI.

### Claude Code Plugins
**Repo:** [danielbodart/claude-code-plugins](https://github.com/danielbodart/claude-code-plugins) | 3 stars | 76 commits

A collection of Claude Code plugins for security, git workflows, code review and research:
- **url-allowlist:** Restricts WebFetch to URLs discovered via WebSearch with three-level match (exact/prefix/domain) to prevent data exfiltration via query parameters
- **commit-push-trunk:** Trunk-based development workflow with git worktree support and CI build monitoring
- **code-review-local:** Adapted official code-review for local/trunk-based workflows (5 parallel agents, confidence scoring)
- **graphical-sudo:** Graphical sudo prompts via `SUDO_ASKPASS`, preferring galley with zenity as fallback (superseded on NixOS by quoin's sudo gate)
- **statusline:** context usage, 5h/7d quota bars from the `rate_limits` payload, effort level, collapsed worktree paths
- **notification-sound**, **web-researcher**
- Includes systematic red-team testing documentation (19 exfiltration vectors tested)

### Claude Code Sandbox
**Repo:** [danielbodart/claude-code-sandbox](https://github.com/danielbodart/claude-code-sandbox)

Deny rules for remote-mutating operations across git, gcloud, wrangler, gh, and hf CLIs. Filesystem denyRead for ~/.ssh, ~/.gnupg, AWS/GCP credentials, .env files. The settings-level precursor to the [Agent Safety Stack](#agent-safety-stack-2026), which moves enforcement out of the agent's config and into the kernel and host daemons.

---

## Infrastructure / Ops: Nix

### nix-config -- Every Machine in the House as One Flake
**Repo:** danielbodart/nix-config (private) | **Language:** Nix, Shell | **548 commits in ~8 weeks** (Sep-Oct 2026) | 139 modules, ~11k lines, 31 pinned inputs

One flake declares 10 built machines (plus 2 that are only pinned). One command, `sudo nixos-rebuild switch`, works on every machine with no path or `#host`. System config and `$HOME` (home-manager) apply together in one rebuild.

**Hosts:**

| Host | Machine | Built as |
|------|---------|----------|
| dan-desktop | i9-12900K + RTX 5070 Ti | NixOS |
| dan-blade | Razer Blade 16 laptop (iGPU + NVIDIA offload) | NixOS |
| server | ASRock DeskMini (game servers, Plex, Samba, Home Assistant) | NixOS, upgrades itself hourly |
| dan-mac | Apple Silicon M4 | nix-darwin over Determinate Nix, sharing the same home-manager config |
| gateway + 4 access points | Google WiFi units | OpenWrt images built in Nix (`nix build .#openwrt-<ap>`) via nix-openwrt-imagebuilder |
| moonlight | Raspberry Pi 4 (game streaming to the TV) | Raspberry Pi OS image built in Nix via **nix-raspios-imagebuilder** (my own) |
| modem, phone | not built | pinned only: host key and login, WireGuard peer key |

**Architecture highlights:**
- **Dendritic flake-parts layout:** every `.nix` file is a flake-parts module found on disk by import-tree, so no host lists its imports. A feature (capsper, the SSH signer, the access point) carries its NixOS, home-manager and OpenWrt halves in one folder. Adding a machine needs no edit to `flake.nix`
- **A small module system for OpenWrt:** OpenWrt has none, so a thin one sits in front of the imagebuilder -- e.g. the Wi-Fi channel plan for every box is set in one file
- **One installer USB key per machine, fully offline:** the disko partitioning script and the host's whole prebuilt system as a zstd binary cache, inside LUKS. Install is one passphrase and the hostname typed back: no network, no evaluation, no build. Rehearsed in QEMU first
- **Every machine must fit on an 8GB key:** a hard rule that keeps closures honest (the laptop's 19.8GiB system compresses to 6.3GiB)

**Security posture:**
- **Secure Boot** with lanzaboote and per-machine signing keys (MSI firmware enrolls the key in db; the Razer can't, so it goes through shim with a MOK)
- **LUKS unlocked by the TPM, sealed to PCR 7 and PCR 15** -- PCR 15 defeats swapping in your own LUKS volume to get the key unsealed; passphrase is the fallback
- **SSH keys held in the TPM:** host and user keys have no Ed25519/RSA copy anywhere, so a disk or backup carries nothing that could stand in for the machine. Every machine trusts every other with pinned host keys, no trust on first use
- **sops-nix with age keys in the TPM:** two recipients per machine (edit and boot-time decrypt) plus a post-quantum recovery key kept offline. Secrets are only ever paths, never values in the world-readable store

### Nix + Claude: Making the Machine Safe for Agents
The premise: an OS that is reproducible, declarative and can be rolled back to any earlier generation from the boot menu is the right base for running coding agents with real autonomy. If an agent (or I) breaks something, the previous generation is one reboot away; every change is a reviewable diff.

- **Claude Code is a pinned, reproducible package.** Plugin marketplaces are store paths and `settings.json` is a read-only store symlink, so an agent can't quietly change its own config -- a plugin is added by editing the config and rebuilding, on purpose
- **MCP servers come from nixpkgs, not `npx @latest`:** Playwright, waydriver (headless Wayland GUI testing) and Openterface bundled into a home-manager-generated plugin
- **The agent safety stack is wired in fleet-wide:** chase tiers by checkout path, flong containers, frisket credential injection (each tier with its own scoped Cloudflare account), quoin's TPM signer and sudo gate on the workstations, remote-mode sudo on the server, the darwin sudo module on the Mac, galley as the desktop approval queue. The server, gateway and modem are declared as ops-tier SSH targets with per-command expectations
- **Codex inside Claude:** dragoman bridges OpenAI Codex in as a subagent, running inside the same chase tiers

### Home Server: Flatcar -> NixOS
**Repo:** danielbodart/server (private) | **130 commits** (2024-2026)

- **Earlier, Flatcar:** net-booted from the OpenWrt gateway with a custom iPXE menu. When Flatcar's CDN rotated its root certificate the diskless server couldn't boot, so boot images moved onto the gateway over plain HTTP -- integrity moved from the transport to the content (GPG + SHA512 on mirror, then pinned hashes)
- **Now, NixOS:** a host in nix-config. Docker containers became confined systemd units one at a time, with the install rehearsed in a VM against a decoy data disk. It upgrades itself hourly from trunk using a read-only deploy key held in its TPM; reboots wait until nobody is playing
- **Hands-off game updates:** a GitHub Actions cron moves only the game-fleet inputs in the lock every hour, the server picks that up within the hour, and Tidewaiter swaps each game in once its players leave

### Game Server Fleet -- Hands-Off Hosting for Me and My Kids

Self-hosted Minecraft Bedrock, Valheim and Hytale servers for my kids and me, built as a small open-source toolchain so the servers look after themselves. New game versions flow in automatically (hourly CI bumps the game-fleet inputs in the flake -> the NixOS server rebuilds itself -> Tidewaiter swaps each game in once its players leave), and nobody gets kicked mid-game. Upstream fixes along the way: LinuxGSM (Bedrock monitor restart loop) and miniupnp (IPv4 mask with IPv6 disabled on OpenWrt).


| Project | Purpose | Commits |
|---------|---------|---------|
| **[Portical](https://github.com/danielbodart/portical)** (26 stars) | UPnP port forwarding from a single Docker label or systemd key; static binaries for amd64, arm64 and armv7 | 93 |
| **[Tidewaiter](https://github.com/danielbodart/tidewaiter)** | Auto-updater that only updates idle services; activity detected with conntrack (incl. UDP); health-gated swap with rollback by renaming rather than deleting; covers Docker and NixOS systemd units | 42 |
| **[Chandlery](https://github.com/danielbodart/chandlery)** (Shell) | Clean-room, Docker-native game-server images (Bedrock, Valheim, Hytale): game checksum-verified and baked in, so the image tag *is* the game version; graceful stop, real health checks, console via `docker attach` | 145 |
| **[Treesinger](https://github.com/danielbodart/treesinger)** | Hytale OAuth broker: holds the only refresh token and issues short-lived sessions per server on request; adding a server needs no config | 41 |
| **[gokit](https://github.com/danielbodart/gokit)** | Shared helpers pulled out of the three daemons: http.Handler/RoundTripper fakes, a stoppable clock for tests that never sleep, logging, environment | 4 |

### Nix Packaging (public)
- **[nix-raspios-imagebuilder](https://github.com/danielbodart/nix-raspios-imagebuilder):** Raspberry Pi OS images built by Nix. Every `.deb` locked by URL and SHA256 and resolved against signed indexes. Read-only root with a RAM overlay; A/B upgrades over SSH via the firmware's `tryboot`, falling back automatically if the new slot isn't healthy
- **[claude-desktop](https://github.com/danielbodart/claude-desktop) / chatgpt-desktop:** flakes for the **official** Linux apps. Versions pinned to the vendors' PGP-signed apt indexes rather than a hand-copied hash, self-updating hourly, built and launched on x86_64 and aarch64. Makes Claude Desktop's Cowork (QEMU VM) work on NixOS, with the closure at 900MiB vs 2.8GiB
- **[glim](https://github.com/danielbodart/glim)** (Zig): ~100KB static binary exposing the Razer Blade keyboard backlight to Linux as an LED through uleds -- GNOME's slider, screen-lock blanking and Fn keys all work with no kernel module, so it survives Secure Boot

### Blog Migrations: WordPress -> Hugo
2 blogs rebuilt/migrated from WordPress to Hugo static sites.

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
