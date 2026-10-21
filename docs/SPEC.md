# Build Prompt: "Reelhouse" — Self-Hostable Video Review & Production Hub

> Codename **Reelhouse** (placeholder — rename freely). Paste this whole file into Claude Code at the root of an empty repo, or save it as `CLAUDE.md` / `docs/SPEC.md` and tell Claude Code: *"Read SPEC.md and build it, milestone by milestone."*

---

## 0. Instructions to Claude Code (read first)

You are building a production-grade application from scratch. You have **full autonomy** to:

- Install any languages, runtimes, package managers, system tools and libraries you need (Node.js, pnpm, Rust/Cargo for Tauri, Python, FFmpeg, Docker, etc.).
- Choose specific library versions, scaffold folders, write migrations, scripts, tests and docs.
- Make reasonable technical decisions without asking. Record each significant decision in `docs/DECISIONS.md` (one short entry: what, why, alternatives).

Working rules:

1. **Build milestone by milestone** (Section 9). Do not start a milestone until the previous one runs end to end and its acceptance checks pass.
2. After each milestone: run the app, run tests, fix what's broken, commit with a clear message, and update `docs/PROGRESS.md` (done / next / known issues).
3. Keep everything runnable with **one command** for dev (`pnpm dev`) and **one command** for self-hosting (`docker compose up`).
4. Prefer boring, well-maintained libraries. No abandoned packages.
5. Never hard-code secrets. Use `.env` with a committed `.env.example`.
6. Write a `README.md` that a non-developer team lead could follow to install it.
7. If something in this spec is ambiguous, pick the simplest option that keeps the door open, note it in `DECISIONS.md`, and continue.

---

## 1. Product vision

A **video review + production management platform** for video editors, agencies and their clients — like Frame.io and Timeliner.io combined, but:

- **Self-hostable and privacy-first.** Runs fully on your own machine / LAN with zero cloud dependency, *or* on a cheap VPS for remote teams. Same app, same features.
- **More customizable.** Custom workflow stages, custom fields, custom roles, branding, keyboard shortcuts, plugins/webhooks.
- **Richer feedback.** Frame-accurate comments, voice-note comments, paint-over annotations, version comparison.
- **Light editing built in.** Quick trims/cuts without opening a full NLE.
- **Available as a desktop app *and* a web app**, sharing one codebase.

Primary user: a small-to-mid video team (editors, a lead/producer) + their external clients who review and approve.

---

## 2. Deployment model (critical architecture decision)

There is **one server application** and **two clients** (web + desktop). "Local" vs "hosted" is only a question of *where the server runs*:

| Mode | Where the server runs | Who can access | Storage |
|---|---|---|---|
| **Solo / Local** | Embedded inside the desktop app (sidecar process) on `localhost` | Just you | Local disk + SQLite |
| **LAN / Studio** | One machine in the office (Docker) | Team on the same network / VPN (Tailscale recommended) | Local disk/NAS + Postgres |
| **Hosted** | A VPS (e.g. Hetzner) via Docker | Team + clients over the internet | S3-compatible (Cloudflare R2 recommended) + Postgres |

Requirements:

- The desktop app can either **run its own embedded server** or **connect to any remote server URL** (like a Slack workspace picker). Multiple servers can be saved.
- The web client is served by the server itself (no separate hosting needed).
- Database layer must support **SQLite (local) and Postgres (team/hosted)** with the same schema and migrations (use Drizzle ORM or Prisma — your call, record it).
- **Export/Import**: a project can be exported as a self-contained bundle (DB rows + media + comments as JSON) and imported into any other instance. This is how data moves between local and hosted — *no live two-way sync in v1* (that's a stretch goal).

---

## 3. Recommended tech stack (adjust if you have a strong reason; log it)

- **Monorepo**: pnpm workspaces + Turborepo.
- **Language**: TypeScript everywhere possible.
- **Frontend**: React + Vite, TanStack Query, Zustand, Tailwind CSS, Radix/shadcn-style components.
- **Desktop**: **Tauri 2** wrapping the same frontend; the server ships as a sidecar binary (bundle Node with the server, or compile it with `bun build --compile` / `pkg` — choose and document).
- **Server**: Node.js + Fastify, WebSockets for realtime (comments, presence, status changes), Zod for validation.
- **DB**: Drizzle ORM (or Prisma) → SQLite / Postgres.
- **Jobs/queue**: lightweight DB-backed job queue (e.g. pg-boss for Postgres; simple polling queue for SQLite) — no Redis requirement in v1.
- **Media**: **FFmpeg / ffprobe** for transcoding, proxy generation, thumbnails, waveforms, trims. HLS for adaptive playback (hls.js on the client).
- **Storage abstraction** (`StorageDriver` interface): `local-disk`, `s3` (works with Cloudflare R2, Backblaze B2, MinIO, AWS), and `google-drive` (archive tier only — see §6).
- **Auth**: email + password (argon2), magic-link for clients, optional OIDC/Google sign-in. Session cookies, RBAC.
- **Transcription (optional, local)**: whisper.cpp or faster-whisper (Python) for transcribing voice comments and video audio. Must be optional and run offline.
- **Testing**: Vitest (unit), Playwright (e2e web).
- **Deploy**: Dockerfile + `docker-compose.yml` (server + Postgres + optional MinIO), Caddy for automatic HTTPS.

---

## 4. Core features

### 4.1 Workspaces, projects, assets
- Workspace → Projects → Folders → Assets (video, image, audio, PDF).
- Assets have **versions** (v1, v2, v3…) stacked under one asset; uploading a new cut adds a version.
- Chunked, resumable uploads (tus protocol or S3 multipart). Drag-and-drop folders.
- **Watch folders** (desktop/local mode): point at a folder; new renders auto-upload as new versions.

### 4.2 Video player & review
- Frame-accurate scrubbing, J/K/L shuttle, frame stepping, in/out points, playback speed, loop range.
- Timecode display (SMPTE, frames, seconds), configurable FPS handling.
- Proxy playback (HLS 1080p/720p/360p) with original file available for download.
- Audio waveform under the timeline. Safe-area / aspect-ratio guide overlays (16:9, 9:16, 1:1, 4:5).
- **Side-by-side and overlay (wipe) version comparison**, synced playback.

### 4.3 Comments (the heart of the app)
- Frame-accurate or **range** comments (in→out) shown as markers on the timeline.
- Threaded replies, @mentions, emoji reactions, resolve/unresolve, attachments.
- **Voice-note comments**: record in browser (MediaRecorder → Opus), show waveform, playback speed; optional auto-transcription so voice notes are searchable and readable.
- **Paint-over annotations**: freehand pen, arrow, rectangle, circle, text, color picker, stroke width. Stored as **vector JSON tied to a frame/timecode** (not baked images), rendered on a canvas overlay, scale-correct at any player size.
- Comments carry over between versions with a "may be outdated" flag.
- Filter comments by status, author, version, tag. Export comments as **CSV, PDF, and NLE markers** (Premiere XML, DaVinci Resolve EDL/marker CSV, Final Cut FCPXML).
- Internal vs client-visible comments.

### 4.4 Quick editor ("Fast Cut")
- Trim head/tail, split, delete segments, reorder segments on a simple single-track timeline.
- Mute / basic volume, optional fade in/out.
- Output is a **new version** of the asset (non-destructive). Use FFmpeg stream-copy when cuts land on keyframes, re-encode otherwise; show which mode will be used.
- Export presets (H.264 MP4 1080p, 9:16 social, ProRes proxy if available).

### 4.5 Production management
- Every deliverable is a **task** with owner(s), due date, status, priority, linked assets.
- **Kanban board**, list view, calendar view, and a **workload view** per editor.
- **Fully custom workflow stages** per workspace/project (default: To do → In progress → Internal review → Client review → Changes requested → Approved → Delivered).
- **Custom fields** (text, number, select, date, user, URL) on projects and tasks.
- **Review gating**: a version must pass internal approval before clients can see it.
- Revision counter per deliverable (with optional revision-limit warning).
- Templates for recurring project types.

### 4.6 Client portal
- Branded portal per workspace (logo, colors, custom domain in hosted mode).
- Clients join via magic link — no password, unlimited free guest reviewers.
- Clients see only what's shared with them, can comment, voice-note, draw, and **Approve / Request changes**.
- Password-protected and expiring share links; download permission toggle; optional watermark (viewer email + timestamp burned into proxy).

### 4.7 Business & analytics
- Per-project budget, payment status, invoices tracking (record-keeping, not payment processing).
- Editor cost/rate tracking, per-project revenue vs. cost.
- KPIs: average revision rounds, turnaround time, on-time delivery %, per-editor throughput.

### 4.8 Notifications & integrations
- In-app notification center, email digests, desktop notifications.
- **Webhooks** for all major events (comment.created, version.uploaded, task.status_changed, asset.approved).
- Public REST API with personal access tokens + OpenAPI docs.
- Optional Slack/Discord notification webhook.

---

## 5. "Better than Timeliner" upgrades (build after core, see milestones)

1. **Fully offline mode** — the entire product works with no internet.
2. **Searchable everything** — global search across comments, transcripts of voice notes and video dialogue (local Whisper), task names, file names.
3. **Keyboard-first review** — every action has a shortcut; shortcut editor; command palette (Ctrl/Cmd+K).
4. **Plugin system** — server-side hooks + UI extension slots, loaded from a `plugins/` folder.
5. **Theming** — light/dark + custom accent colors, saved per user.
6. **Approval audit trail** — immutable log of who approved what version when (useful for client disputes).
7. **AI assist (optional, pluggable, off by default)** — summarize all comments on a version into a change list for the editor.
8. **Data portability** — one-click full workspace export; no lock-in.

---

## 6. Storage strategy & cost guidance

Implement `StorageDriver` with these drivers, configurable per workspace:

- **local-disk** — default for Solo/LAN modes.
- **s3** — default for hosted mode. Recommend **Cloudflare R2** (≈ $0.015/GB-month, **no egress fees**, S3-compatible). Also works with Backblaze B2 / MinIO.
- **google-drive** — **archive/backup tier only**, not for active playback. Uses the Google Drive API with the user's own account (OAuth). Use cases: auto-archive delivered projects after N days, nightly backups of originals. Must respect Drive API rate limits (exponential backoff, queued jobs). Drive is *not* suitable as the playback source because it isn't a CDN and isn't built for frame-accurate seeking.

Storage tiers per asset: **hot** (proxies + active originals on local/R2) and **archive** (Drive). Add a "Restore from archive" action.

Target hosting cost for a small team (document this in README):
- VPS (Hetzner small instance): ~ $5–20/month (prices have shifted in 2026 — verify current pricing).
- R2 video storage: ~ $7.50/month per 500 GB, playback bandwidth free.
- Google Drive: already paid by the user → free archive.

---

## 7. Data model (starting point — refine as needed)

`users, workspaces, workspace_members(role), projects, folders, assets, asset_versions, media_files(proxy/original/thumb/waveform/hls), comments(parent_id, version_id, timecode_in, timecode_out, frame_in, frame_out, body, visibility, status), comment_annotations(vector JSON), comment_audio(file, duration, transcript), reactions, tasks, task_assignees, workflow_stages, custom_field_defs, custom_field_values, share_links, approvals(audit), notifications, webhooks, api_tokens, jobs, activity_log, invoices, rates.`

All IDs are UUIDv7 or ULIDs (export/import-safe across instances).

---

## 8. Security & privacy

- RBAC roles: Owner, Admin, Producer, Editor, Reviewer (internal), Client (external guest).
- Signed, expiring URLs for every media request. No public buckets.
- Rate-limiting, CSRF protection, secure cookies, CSP headers.
- Optional at-rest encryption for local storage.
- No telemetry. Nothing leaves the machine in Local mode unless the user configures a cloud driver.
- Backups: `reelhouse backup` CLI command (DB dump + media manifest).

---

## 9. Milestones (build in this order)

**M0 — Foundation**
Monorepo, tooling, lint/format, env config, Docker compose, CI script, auth (email/password), workspaces, RBAC skeleton. ✅ `pnpm dev` runs web + server; can register and log in.

**M1 — Upload & playback**
Chunked uploads, local-disk driver, FFmpeg pipeline (probe, proxy HLS, thumbnails, waveform), job queue, player with frame-accurate controls and timecode. ✅ Upload a 2 GB video, it transcodes, plays smoothly, scrubbing is frame-accurate.

**M2 — Comments core**
Timestamped + range comments, threads, resolve, markers on timeline, realtime updates via WebSocket. ✅ Two browsers see each other's comments live.

**M3 — Rich feedback**
Paint-over annotations (vector), voice-note comments, versions + side-by-side/wipe compare, comment carry-over. ✅ Draw on a frame, record a voice note, compare v1 vs v2.

**M4 — Production management**
Tasks, custom workflow stages, kanban/list/calendar, custom fields, internal→client review gating. ✅ Move a task through the full pipeline.

**M5 — Client portal & sharing**
Magic-link guests, branded portal, share links (password/expiry/download/watermark), approve/request changes, approval audit trail. ✅ A client approves a version from an incognito window.

**M6 — Desktop app**
Tauri shell, embedded server sidecar with SQLite, "connect to remote server" picker, watch folders, native notifications. ✅ Installer builds for Windows and macOS; works fully offline.

**M7 — Fast Cut editor**
Trim/split/delete/reorder, fades, export presets → new version. ✅ Trim 5 seconds off a clip and the result appears as v(n+1).

**M8 — Cloud & archive storage**
S3/R2 driver, Google Drive archive driver, tiering, restore. Project export/import bundles. ✅ Export a project from local, import into hosted instance, all comments intact.

**M9 — Business, analytics, integrations**
Budgets/payments tracking, editor KPIs, webhooks, REST API + OpenAPI, Slack/Discord notifications, NLE marker export. ✅ Webhook fires on approval; markers import into Premiere/Resolve.

**M10 — Upgrades & polish**
Local Whisper transcription + global search, command palette, shortcut editor, theming, plugin system, accessibility pass, performance pass, full docs. ✅ Search finds a word spoken inside a voice note.

**Stretch:** live two-way sync between local and hosted instances (CRDT/event-log based), mobile-friendly review UI, AI comment summaries.

---

## 10. Definition of done (whole project)

- `docker compose up` on a fresh VPS gives a working HTTPS instance (Caddy) in under 10 minutes following README.
- Desktop installer runs offline with no setup.
- Test suite green; e2e covers upload → comment → approve.
- `docs/` contains: README, SELF_HOSTING.md (VPS + R2 + Google Drive setup, cost table), DECISIONS.md, PROGRESS.md, API docs.

Start with **M0** now.
