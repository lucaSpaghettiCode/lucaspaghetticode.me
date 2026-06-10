## Context

Adagio is a high-performance, cross-platform **Nextcloud desktop sync client written in Rust** — a from-scratch alternative to the official client, built around a background daemon rather than a monolithic GUI app. It syncs files between your devices and a Nextcloud server, delivers files on demand through a virtual filesystem, and (in progress) speaks Nextcloud's end-to-end encryption protocol wire-compatible with the official clients.

The name is the tempo: sync that is deliberate, steady, and doesn't trample the rest of your machine. The first release, **v0.1.0 "Allegretto"**, keeps the theme.

The project is also an experiment in *how* software gets built: every one of its 18 features started as a written specification before any code, with architecture decisions captured in ADRs. That story has its own essay — [Building Adagio: a spec-driven experiment](#post/adagio-sdd-experiment).

---

## Architecture: Daemon First

Everything runs through `adagio-daemon`, a tokio-based background process. The native GUI (`adagio-desktop`, Tauri v2 with a React/TypeScript frontend), the CLI (`adagio-cli`, clap v4), and the virtual filesystem are all thin clients of the same daemon.

```
adagio-desktop (Tauri + React)      adagio-cli (clap)
        │        NDJSON IPC (Unix socket / named pipe)
adagio-daemon (tokio)
  ├─ SyncEngine      (adagio-core)
  ├─ VfsPairRunner   (adagio-vfs / FUSE3)
  └─ E2eeRunner      (adagio-e2ee)
        │        WebDAV / OCS API
Nextcloud server
```

IPC is deliberately boring: **newline-delimited JSON-RPC 2.0 over a Unix socket** (named pipe on Windows), with a second connection type on the same endpoint for push-event subscriptions. OS-enforced user isolation for free, zero extra dependencies, and you can debug the entire protocol with `socat` and your eyes.

The workspace is eight crates with clean seams: `adagio-core` (sync engine, journal, reconciler, propagator, bandwidth), `adagio-nextcloud` (WebDAV + OCS client), `adagio-e2ee` (crypto), `adagio-vfs` (FUSE3 driver), `adagio-ipc` (shared request/response types), plus the three binaries.

---

## The Sync Engine

The source of truth is a **sync journal in SQLite (WAL mode, via sqlx)**: last-known-good checksum, etag, mtime, and status for every file — concurrent reads from the UI alongside write-heavy sync cycles, under 50 MB for half a million entries. Each cycle scans both sides, the reconciler diffs against the journal, and the propagator executes the plan.

The detail work is where it gets interesting:

- **Conflicts** get three resolution policies — Ask, Keep-Local, Keep-Remote — with a wizard UI for the "Ask" path, fed through a bounded channel so a thousand-conflict initial sync can't balloon memory.
- **Initial sync has a fast path.** The propagator's steady-state concurrency (3 uploads) would make a 100 GB first sync take days, so a **BulkUploadDriver** kicks in when the remote is effectively empty and pending uploads exceed a threshold — more workers, file-level resume across daemon restarts, no per-cycle rescan overhead.
- **Bandwidth limits are enforced per 64 KB chunk**, not per file: a token bucket consulted inside the transfer loop. The naive per-file alternative fails visibly — `acquire(50 MB)` at a 500 KB/s limit sleeps 100 seconds and then uploads at full speed anyway.

---

## Knowing When *Not* to Sync

A sync client is a guest on someone's machine. The daemon watches three runtime conditions and reacts within seconds: **metered connections** (pause or restrict heavy transfers), **battery state** (defer work until on AC), and **SSID blocklists** (never sync on networks you've named). Each detection path has a per-platform implementation with graceful degradation where the OS won't say.

---

## VFS and E2EE

The **virtual filesystem** (FUSE3 on Linux, with the driver layer designed for Windows CfAPI and macOS FileProvider later) shows the full remote tree while materialising file content on demand, with explicit **pin/evict** for offline access — the placeholder model, without dedicating a disk to a mirror.

**End-to-end encryption** implements the Nextcloud E2EE RFC for interoperability with the official desktop and mobile clients: RSA-4096 device keys with OCS certificate signing, AES-128-GCM file content encryption, CMS-signed metadata, and **BIP-39 mnemonic** device pairing. Interop is the hard constraint — the algorithm choices are the RFC's, not mine, and the work is making a second implementation agree with the first one byte for byte.

---

## Process

Development follows a strict spec-driven workflow: each feature lives in `specs/NNN-name/` with a specification, implementation plan, data model, and task list — 18 of them shipped or in flight, from `001-nextcloud-file-sync` to `018-mkdocs-docs-site`. Around twenty **Architecture Decision Records** document the choices that would otherwise live in commit messages: why SQLite over anything with a server, why NDJSON over gRPC for local IPC, why chunk-level throttling, why the conflict channel is bounded. Contract and integration test suites plus criterion benchmarks (checksum, reconciler) keep the engine honest.

Current state: v0.1.0 "Allegretto" released — sync, daemon, GUI, CLI, OAuth2 login, conflicts, throttling, network awareness, bulk upload, and FUSE3 VFS all done; E2EE in progress; a LAN-peer sync protocol queued next.
