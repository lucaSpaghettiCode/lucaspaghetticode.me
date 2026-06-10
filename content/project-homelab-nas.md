## Context

Every homelab eventually converges on the same realisation: the interesting part isn't the hardware, it's owning the failure modes. After years of data and subscriptions scattered across services whose incentives kept drifting away from mine, I built one small machine whose job is to hold my files, serve my media, and be understandable end to end.

The design brief was short: **one box, one compose file, no cloud accounts in the critical path**. Everything reachable from anywhere, nothing exposed to the internet.

For the longer story — the philosophy, the 2am debugging sessions, the mistakes — see the companion essay, [The deliberate machine](#post/deliberate-machine). This page is the technical summary.

---

## Hardware: Constraints as Features

The build centers on a **Raspberry Pi 4**: a single-board ARM computer drawing roughly five watts at idle, living on a shelf. This was a deliberate choice, not a budget compromise. Constrained hardware forces discipline — every service must publish an ARM64 image and justify its memory footprint, which filters out bloated or poorly-maintained projects before they ever join the stack.

Storage is a **2 TB spinning HDD**, mounted at `/mnt/nas`. Spinning rust in the SSD era is the right call here: media libraries are large, cold-ish storage where cost-per-gigabyte still favors HDDs by a factor of four or five, and seek latency is irrelevant when the workload is streaming single audio files.

---

## Networking: Tailscale Instead of Port Forwarding

There is no port forwarding, no dynamic DNS, no reverse proxy on the public internet. The Pi joins a **Tailscale** mesh and simply appears at its `.ts.net` hostname from any device on the tailnet — laptop, phone, anywhere. No firewall rules to maintain, no exposed attack surface, and the **split-brain DNS problem** (different addresses inside vs. outside the LAN) disappears entirely: there is one address, and it always works.

One sharp edge worth recording: `.ts.net` hostnames are on browsers' **HSTS preload lists**, baked into the binary — plain-HTTP services on a `.ts.net` name get silently upgraded to HTTPS even in private tabs. That, plus a SPA that assumes it lives at the root path, is why one service is still accessed by raw Tailscale IP and port. It's a workaround, not a solution, and it's on the roadmap.

---

## The Stack: Nineteen Services, One Compose File

The entire fleet lives in `~/stack/` as a single **Docker Compose** project, fronted by **Caddy** for TLS termination and routing. Application data goes under `/mnt/nas/appdata/` on the external drive, deliberately separated from the stack config: code and configuration live in the home directory (versionable), persistent state lives on the drive. A full stack rebuild doesn't touch data; a data migration doesn't touch config. This separation has saved me twice.

The load-bearing services:

- **Jellyfin** — open-source media server, direct-playing Opus audio and H264 video. The memorable bug: the music library is tagged with non-standard Vorbis comment fields (`ALBUM_ARTIST` vs `ALBUMARTIST`), which Jellyfin silently ignores by default — tracks and albums indexed, artists missing, zero errors logged. The fix is one toggle; finding it required diffing the Vorbis spec against what was actually in the files.
- **Nextcloud** — file sync, photos, calendar, contacts. The largest complexity surface in the stack, and the service that fights self-hosting the hardest: PHP app, database, background jobs, and a configuration surface where the proxy headers, `overwriteprotocol`, and trusted-proxy list must all agree or warnings accumulate.
- **Yubal** — a YouTube Music → local library pipeline on a nightly cron: downloads curated playlists as Opus and drops them straight into Jellyfin's music folder, memory-limited and running as the stack's user. No subscription, library stays current.
- **Beets + a `mutagen` fallback script** — automated tagging via AcoustID/MusicBrainz works brilliantly for mainstream releases and matched roughly fifteen percent of this live-heavy, regional-indie library. The fallback, `tag_from_path.py`, reads metadata from the `Artist/Album/Track` folder convention — idempotent, dry-run by default. The lesson: an automated solution is bounded by the quality of the data it depends on.

---

## Lessons Paid For

- **Validate bind mounts before the first container run.** A misconfigured volume meant several hundred downloaded tracks were written to the ephemeral container layer and ceased to exist on recreation. No error, no warning. Thirty seconds of `docker inspect` now precedes every first start.
- **Idempotent, dry-run-safe automation** is the only kind worth writing for a system you operate alone.
- **Operational intimacy is the actual product.** I know why each service breaks because I've watched each one break. That mental model doesn't deprecate, and it transfers to every distributed system I touch professionally.

---

## The Honest Backlog

No project page is complete without it: a proper **3-2-1 backup strategy** (local redundancy exists, offsite doesn't yet — unacceptable for a curated music library built over years), **monitoring** (Uptime Kuma first, lightweight metrics later; today I find out a service is down by trying to use it), and fixing the subpath routing properly instead of living with the direct-IP workaround.

The machine runs. Five watts. Twenty-four hours a day. Entirely mine.
