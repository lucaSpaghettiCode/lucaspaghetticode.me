## The Itch That Started It All

Every few months I go through the same ritual. I open my Nextcloud web interface, stare at the sync client — the official desktop app — and wait. I watch it re-index directories it already indexed. I watch it eat 400 MB of RAM to display a spinner. I open Activity Monitor, see it spawning child processes I can't account for, and close the laptop lid hoping the problem goes away.

It doesn't.

The official Nextcloud desktop client is a Qt application that has been evolving for well over a decade. It is the product of many contributors, many Nextcloud versions, many corporate integrations, and many compromises. It works. But it carries the weight of all those decisions — and it shows.

I wanted to know: what would a sync client look like if you started from scratch today, in Rust, with a single guiding constraint — lean by default? No bundled Chromium. No background processes you can't see. No state that survives a restart in ways you can't explain. Just a daemon, a UI shell, a CLI, and a clearly defined contract between them.

That question became Adagio.

## What Adagio Is

Adagio is a cross-platform Nextcloud desktop sync client written in Rust. It runs a background daemon (`adagio-daemon`), exposes a native GUI (`adagio-desktop`, built with Tauri v2 and React/TypeScript), a CLI (`adagio-cli`), and a virtual filesystem (`adagio-vfs`) powered by FUSE3 on Linux. All communication between the GUI, CLI, and daemon happens over a Unix socket using NDJSON-framed RPC.

In plain terms: there is one process doing the actual work. Everything else — the window you open, the `adagio status` command you run from a terminal — is a client talking to that process over a socket.

The architecture is nearly embarrassingly simple:

```
┌──────────────────────────────────────────────────────┐
│  adagio-desktop  (Tauri + React/TypeScript)          │
│  adagio-cli      (clap v4)                           │
│              │ NDJSON IPC (Unix socket)              │
│  adagio-daemon   (tokio, background process)         │
│    ├─ SyncEngine      (adagio-core)                  │
│    ├─ VfsPairRunner   (adagio-vfs / FUSE3)           │
│    └─ E2eeRunner      (adagio-e2ee)                  │
│              │ WebDAV / OCS API                      │
│  Nextcloud server                                    │
└──────────────────────────────────────────────────────┘
```

Simple. Auditable. Every component has a clear job and a clear boundary.

## The Process Experiment: SpecKit and Spec-Driven Development

Building Adagio was also an experiment. I had a theory: that AI-assisted software development produces its worst results when it operates without constraints, and its best results when it operates inside a disciplined process. I wanted to test that theory rigorously — on a real project, over a sustained period, at a level of complexity that would stress the process rather than flatter it.

I used SpecKit, a Claude Code harness that enforces a Spec-Driven Development (SDD) workflow. Every feature in Adagio — all 18 of them — went through the same pipeline:

1. `/speckit-specify` — a structured specification with user stories, acceptance scenarios, and independently testable P1/P2/P3 priorities
2. `/speckit-plan` — an architectural plan with ADRs, data models, and IPC contracts
3. `/speckit-tasks` — a dependency-ordered task list, with each task mapped to a specific artifact
4. `/speckit-implement` — execution, one task at a time, with CI validation between each

The constraint was simple: nothing gets built without a spec. No "quick prototype" that lives forever. No architecture decisions without a written rationale.

This sounds bureaucratic. In practice, it was the opposite. Because the spec existed, I could hand any task to Claude Code and trust the output was coherent with what came before. Because the ADR existed, every future maintainer (including me, three weeks later) could reconstruct not just what was decided but why.

## Testing the Limits of the Process

I didn't start the project believing SDD was the right answer. I started believing it was probably overkill. The point was to find out where it broke.

### The specs are wrong, and that is fine

The first spec I wrote (feature 001, the core sync engine) contained factual errors about Nextcloud's WebDAV schema. The first E2EE spec specified AES-256-GCM for file content encryption and Ed25519 for metadata signatures. Both are wrong. The Nextcloud E2EE RFC mandates AES-128-GCM and RSA-4096/CMS.

In a traditional workflow, these errors would live in a Slack thread or someone's memory until a bug report surfaces in production. In SDD, they live in a spec document that gets corrected when the implementation diverges — and the correction creates an ADR that explains why the initial assumption was wrong. ADR-017 documents this explicitly:

> The feature proposal stated "AES-256-GCM for content encryption" and "ed25519-dalek for metadata signatures". Both are incorrect against the published RFC. Using 256-bit would be incompatible with all other Nextcloud clients.

That is useful institutional knowledge. It did not exist before the spec forced it to be written.

### The process scales — but unevenly

The lighter features — bandwidth throttling (feature 009), CLI binary (feature 008), sync resource efficiency (feature 006) — went through the SpecKit pipeline in hours. The spec was thin because the problem was thin. The tasks were mechanical. Implementation was direct.

The heavier features — E2EE (feature 013), VFS (feature 012), the bulk upload driver (feature 011) — needed more spec cycles. The first pass of the VFS spec didn't account for the LRU eviction requirement. The first pass of the E2EE spec hadn't resolved which OCS API version to target. These are the moments where the process earns its overhead: instead of discovering the gap during implementation (expensive), you discover it during specification (cheap).

### AI-assisted implementation without a spec is a random walk

I spent a week early in the project experimenting with "just building" — giving Claude Code a feature description and asking it to implement directly. The results were technically impressive but architecturally inconsistent. Types were named differently than adjacent modules. Error handling used different patterns than already-established crates. The IPC protocol grew organically rather than matching the existing contract.

The spec doesn't prevent these problems entirely. But it gives the model a target. When the spec says "all IPC messages follow the `DaemonRequest` / `DaemonResponse` enum pattern defined in `adagio-ipc`", the model cannot invent a new pattern without explicitly violating that constraint.

## Technical Decisions Worth Explaining

Adagio's architecture is the product of 18 ADRs. Here are the ones I think about most.

### Tauri, not Electron

The memory budget for Adagio is 100 MB RSS at idle. That constraint rules out Electron immediately. Tauri v2 uses the platform's native WebView — WebKitGTK on Linux, WKWebView on macOS, WebView2 on Windows — which means the renderer is a shared system resource rather than a 150 MB private allocation.

The tradeoff is WebView inconsistency: CSS that looks perfect on macOS needs tweaks on Linux. Tauri's WebView on Linux is behind Safari and Chrome in some areas. This is real, and it's felt during UI development. But for a background sync client where the window is mostly closed, it's the right tradeoff.

### NDJSON over Unix socket, not gRPC

The original proposal specified gRPC/tonic for IPC. I deferred it. The reason is pragmatic: NDJSON over a Unix socket requires zero new dependencies (Tokio already provides `UnixListener`), it's human-readable and debuggable with `nc`, and it's the minimal thing that works. When the time comes to add schema validation, versioning, and code generation, the migration to gRPC is mechanical — the protocol is designed as a drop-in replacement.

This is an example of a decision that felt wrong in the abstract ("everyone says use gRPC") but was right in context. At feature 007, I had one IPC client (the desktop app). Adding a protobuf toolchain for one client is not the right tradeoff. At feature 012, I had two clients and an emerging protocol surface. The tipping point hasn't arrived yet.

### SQLite WAL mode everywhere

Every stateful piece of Adagio — the sync journal, the VFS cache metadata, the pinned paths — lives in a single SQLite database in WAL mode. This is not glamorous. It is, however, exactly right.

WAL mode allows concurrent readers alongside a single writer without blocking. The schema is a single `journal_entries` table with indexed columns for the most common queries. For 500,000 entries, the database stays under 50 MB. For 1,000,000 entries, it stays under 100 MB. SQLite has a 40-year track record of not corrupting data. These are not small things.

The single-writer constraint matters: the journal serializes all writes through one connection. This is fine for sync workloads, which are mostly reads interspersed with bursts of writes. It becomes a bottleneck only under pathological conditions — and the bulk upload driver (feature 011) was specifically designed to avoid hammering the journal with per-chunk writes.

### Per-chunk throttling at 64 KB boundaries

The bandwidth throttling implementation (ADR-013) contains one of my favorite decisions in the codebase. The naive approach — `acquire(file_size)` before uploading a file — produces a system that sleeps upfront for the correct amount of time, then transfers at full network speed. This is trivially correct on average but catastrophically wrong under the spec's "any 10-second window must stay within 10% of the ceiling" requirement.

For a 50 MB file at 500 KB/s: the naive approach sleeps 100 seconds, then uploads at ~5 Gbps in under 100 milliseconds. Any 10-second measurement window that captures those 100 ms sees a throughput 1000× over the limit.

Per-chunk throttling at 64 KB boundaries distributes the sleeps evenly across the transfer. An `acquire(65536)` at 500 KB/s sleeps ~130 ms. Any 10-second window sees approximately 77 chunks × 64 KB = 4.9 MB, within rounding of the 5 MB target. The spec compliance isn't accidental — it's derived from the chunk size and the token bucket semantics.

### E2EE: protocol fidelity over crypto novelty

Feature 013 (E2EE) is the most technically dense feature in the project. The temptation with end-to-end encryption is to design something from first principles — elliptic curve everything, latest KEM, modern AEAD. That temptation must be resisted.

Adagio's E2EE must interoperate with the official Nextcloud clients on Android, iOS, and desktop. That means implementing the Nextcloud E2EE RFC exactly — not approximately. The RFC mandates RSA-4096 key pairs, AES-128-GCM for file content (128-bit, not 256-bit), PBKDF2-HMAC-SHA256 at 600,000 iterations, BIP-39 12-word mnemonics, and CMS SignedData for metadata integrity.

None of these are the choices I would make building a system from scratch in 2026. But they are correct choices for this system, because "correct" here means "interoperable with a 200-million-user ecosystem." The E2EE spec's initial errors (AES-256-GCM, Ed25519) would have produced an implementation that passed unit tests and silently failed to mount any folder encrypted by the official Nextcloud Android client.

## What Got Better Along the Way

Looking across the 18 features chronologically, there are clear inflection points where the project quality improves.

### Feature 004 — UI Design System

Before this feature, the UI was assembled component by component from whatever Tailwind classes seemed right. After it, there is a design system: tokens, a dark theme, spacing rules, a typography scale, a color palette with names that mean something. The handoff directory contains the design artifacts. New components can now be built consistently rather than hopefully.

### Feature 007 — Background Sync Daemon

The move from a tightly coupled desktop app to a daemon-plus-clients architecture was the most structurally significant decision in the project. Before feature 007, the sync engine lived inside the Tauri process. After it, the sync engine runs independently and the desktop app is just one client. This means the CLI can observe and control the same sync state as the GUI. It means the daemon can run headlessly on a Raspberry Pi. It means crash recovery doesn't lose sync state.

### Feature 011 — Bulk Upload Driver

The standard sync cycle was designed for steady-state maintenance. Watching it upload a 10,000-file archive sequentially — 3 concurrent transfers, file after file — was demoralizing. The bulk upload driver activates automatically when the remote is effectively empty and there are more than 50 pending uploads. It runs 8 workers in parallel, handles file-level resume after interruptions, and routes large files through the chunked upload protocol. A user syncing a 100 GB photography archive no longer has a multi-day initial sync.

### Feature 012 — VFS On-Demand Files

This one made me most proud of the architecture decision to isolate platform specifics behind a trait. The `VfsProvider` trait has five operations. The Linux FUSE3 implementation uses tokio tasks inside the daemon and calls the existing `Arc<dyn RemoteClient>` directly — no extra socket, no extra process. The macOS FileProvider and Windows Cloud Files API implementations are behind the same trait. When a new platform driver is needed, the sync engine does not change.

## What the Specs Taught Me About Complexity

One thing the SpecKit process makes viscerally clear: the features that seem simple in a pitch are often complex in a spec, and vice versa.

Conflict resolution sounds complex. In the spec, it resolves into three clean policies (Ask, Keep Local, Keep Remote), a wizard UI that surfaces exactly one decision to the user, and an execution model where the `resolve_conflict` Tauri command runs the file I/O directly rather than queuing it for a background propagator. The "complex" problem is actually two simple problems that compose.

Network awareness sounds simple — "don't sync on metered connections." In the spec, it expands into: detecting metered vs. unmetered (which requires OS-specific APIs on Linux, macOS, and Windows), detecting battery state, detecting SSID blocklists, and reconciling all of these into a priority-ordered set of sync policies. The implementation touches platform abstractions that don't exist in the Rust standard library. It needs its own abstraction layer (`NetworkMonitor`) with platform backends.

The spec doesn't resolve this complexity — but it exposes it before you have written a single line of code. That is worth a great deal.

## The Roadmap

Adagio at feature 018 is a functional, well-architected sync client with end-to-end encryption and virtual filesystem support. What comes next?

### Feature 014 — LAN-Peer Protocol

The most requested capability that doesn't require a Nextcloud server: direct device-to-device sync over a local area network. Two devices running Adagio, same network, no cloud intermediary. The protocol will use mDNS for peer discovery and the existing `adagio-ipc` types adapted for peer-to-peer framing. No central coordinator. No cloud hop for files staying inside your home.

This feature tests the architectural decision to isolate the WebDAV client behind a `RemoteClient` trait. If the trait is well-designed, LAN sync should slot in as a new `LanPeerClient` implementation with minimal changes to the sync engine. If the trait is wrong, we find out.

### Feature 015 — Linux System Tray Support

The tray popover is designed but not yet fully wired to live daemon state on all Linux desktop environments. GNOME, KDE Plasma, and XFCE have different AppIndicator support levels and different quirks. Getting the tray icon to be reliably visible — the right icon, in the right state, across all major Linux DEs — is a solved problem in principle but a fiddly one in practice. Feature 015 closes that gap.

### Feature 016 — Onboarding Wizard (Wire-Up)

The five-step onboarding wizard (Welcome → Server → Authorize → Where to sync → Begin) exists as a UI component with hardcoded data throughout. Feature 016 wires it to real Tauri IPC: genuine server URL validation with Nextcloud capability detection, the OAuth2 device flow with a real auth code and QR code, a real folder picker, persisted toggle preferences, and a live initial sync progress readout. This is the feature that makes Adagio accessible to someone who has never opened a terminal.

### Feature 017 — Release Packaging

Installable packages for Linux (`.deb`, `.rpm`, `.tar.gz`), macOS (`.dmg`), and a GitHub Releases page with clear version information, a human-readable changelog, and labeled download links. The CI pipeline already exists. The packaging artifacts need final refinement: correct icon embedding, desktop entry for Linux, notarization for macOS. This is the feature that makes Adagio real to someone who finds it on GitHub.

### Feature 018 — Documentation Site

A MkDocs documentation site hosted on GitHub Pages with the midnight theme, Geist typeface, and Adagio's blue accent palette. Installation guides, configuration reference, CLI reference, IPC protocol documentation, and the ADR catalogue. The docs site is already structured; feature 018 populates it with content and deploys it.

### Beyond 018: The Longer Arc

The longer roadmap is less certain but more interesting. Three directions feel most compelling:

- **Windows and macOS clients.** The architecture is cross-platform by design. The FUSE3 VFS runs on Linux; the `VfsProvider` trait has Windows (Cloud Files API) and macOS (FileProvider) implementations scaffolded but not shipped. Getting all three platforms to production quality requires per-platform CI runners, platform-specific packaging, and platform-specific tray support.
- **gRPC migration.** The NDJSON IPC protocol was always intended as a stepping stone. When the protocol surface stabilizes and the CLI use cases multiply, the migration to gRPC/tonic will pay off: schema validation at compile time, code generation for future language bindings, streaming RPC for progress events.
- **Headless / server deployments.** Adagio on a Raspberry Pi, running without a GUI, syncing files to a Nextcloud server and making them available over a local network share. The daemon already supports this — it's a different packaging and documentation story more than a different code story.

## What I Would Tell Someone Starting a Project Like This

Three things.

1. **Write the spec before the code**, even when it feels like ceremony. The spec is not for the model — it's for you. Writing the acceptance scenario forces you to decide what "done" means before you are emotionally invested in the implementation. The model will implement whatever you describe. You need to describe the right thing.

2. **Trust the ADR process.** When you make a decision that could have gone another way — and those decisions are the only interesting ones — write a short document explaining why you chose what you chose and what you rejected. You will read it in three months when you wonder why the codebase is shaped the way it is. You will be glad you wrote it.

3. **The architecture is the artifact that lasts.** The code changes. Features are added and removed. The architecture — the module boundaries, the trait abstractions, the IPC contract — is the thing that determines how hard every subsequent change is. Spend time on it. Give it a spec. Write the ADR. Change it when you learn something. The incremental cost of getting it right early is trivial compared to the cumulative cost of getting it wrong.

## A Note on the Process Itself

I want to be honest about something. SpecKit and SDD are not magic. The process does not write the code or make the architectural decisions. It provides structure — a place for each type of knowledge to live, a sequence for generating each artifact, a set of questions you must answer before moving on.

What the process does, concretely, is slow you down at the beginning and speed you up everywhere after. The spec for a feature takes an hour or two. The implementation, guided by that spec and its task list, takes a fraction of the time it would take without one. The maintenance — explaining to a new contributor why the NDJSON protocol was chosen, finding the right place to put a new IPC message type — takes almost no time because the answers are already written down.

This is a straightforward time-value tradeoff. But it only works if you actually trust the process enough to complete the spec before opening a code editor. Skipping the spec and writing "just a quick prototype" is the original sin. There are no quick prototypes in sync clients. There are only production bugs waiting to be discovered.

---

Adagio started as a question — what would a lean, privacy-first Nextcloud sync client look like if designed from scratch in 2026? — and became a working answer. It is not finished. It will probably never be finished in the sense of having no more features to add or edges to smooth. But it is honest: every decision has a written rationale, every feature has a spec, and every component has a clear job.

That is, I think, the most useful thing you can say about a piece of software.


<img src="https://raw.githubusercontent.com/TheDarkPyotr/adagio/main/docs/adr/adagio_banner.png" alt="markdown language" width="1000" height="300" class="project-img-wide">
---



*Adagio is open source. The repository, specifications, and ADR catalogue are at [https://github.com/TheDarkPyotr/adagio](https://github.com/TheDarkPyotr/adagio).*
