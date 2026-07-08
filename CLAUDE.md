# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Sigil is a Tauri 2 (Rust shell) + Vite/React 18/TypeScript desktop app for voice, captions, and stream tooling (STT, TTS, translation, AI transform, Twitch/Kick/Discord/OBS, VRChat OSC, voice changer). Package manager is **pnpm**. Windows is the primary target. It began as a fork of [curses](https://github.com/mmpneo/curses), so `CURSES_*` env vars and legacy persisted values still appear for compatibility.

## Commands

```bash
pnpm install
pnpm tauri dev            # full desktop app (Rust + Vite) — the real runtime
pnpm tauri build          # single Windows exe/MSI under src-tauri/target/
pnpm dev                  # Vite only (web, no Tauri) — runs scripts/sync-manual.mjs first
pnpm build && pnpm preview  # static web build + serve; no native features

pnpm typecheck           # tsc --noEmit — run before any PR
pnpm test:e2e            # Playwright: builds then smokes / and /client (both mount #root)
pnpm exec playwright test tests/smoke.spec.ts   # single test file
PW_PREVIEW_ONLY=1 pnpm test:e2e                 # skip rebuild when dist/ already exists

# Rust (from src-tauri/)
cargo clippy --all-targets   # CI gate on the Rust side

# i18n (CI runs i18n:check and will fail on key drift)
pnpm i18n:check          # locale JSON key parity vs public/i18n/en + codes in src/i18n.ts
pnpm i18n:sync           # copy missing English keys into other locales
pnpm i18n:report         # list keys whose value still equals English
node scripts/merge-http-tts-i18n.mjs   # merge new HTTP-TTS UI keys into non-EN locales
node scripts/merge-neural-i18n.mjs     # same, for neural TTS keys
```

`@/` is aliased to `src/` (see `tsconfig.json` + `vite.config.ts`). `pnpm dev`/`pnpm build` run `scripts/sync-manual.mjs`, which copies `docs/manual/` into `public/manual/` and `public/docs/` for the in-app Help reader — edit the source under `docs/`, not the generated copies.

## Architecture

**Two runtime modes**, decided by `AppConfiguration` in `src/config.ts` from the URL path:
- **Host** (default, desktop): root `SigilRoot` — full editor, integrations, inspectors. Has Tauri.
- **Client** (`/client?host=&port=&id=`): root `ClientView` — canvas/elements only, no Tauri; syncs to the host over PeerJS.

**Global singletons** live on `window` (declared in `src/index.tsx`): `Config`, `ApiShared`, `ApiServer` (host services), `ApiClient` (document). Code reaches services via `window.ApiServer.*`.

**Three source layers** (`src/services-registry.ts` holds the `Services` enum and must not import from core/client, to break circular types):
- `src/client/` — document, scenes, elements, files, particles, sound. **Remote-safe: never call Tauri `invoke` or import host-only modules.** This is a hard rule; clients run this code without Tauri. Legacy `audioViz` elements are stripped on load (`src/client/schema/`).
- `src/core/` — host services (`src/core/services/<id>/`), persisted settings, Sigil chrome, `ApiServer` (`src/core/index.ts`), Zod schema (`src/core/schema.ts`).
- `src/shared/` — PubSub bridge and Peer bridge used by both.

**State management** is deliberately split: **Valtio** for backend/service state and doc sync, **Zustand** (`src/core/ui/store.ts`, `useAppUIStore`) for shell UI (sidebar tabs, stats panel), **Yjs + PeerJS** for the collaborative canvas document (undo/redo is document-layer via `documentUndoState`, *not* the stubbed `Service_History`).

**Native side** (`src-tauri/`): frontend calls `invoke("plugin:<name>|<command>")`. Custom Rust plugins are in `src-tauri/src/services/` and registered in `src-tauri/src/main.rs` (osc, web, audio, whisper, vosk-stt, moonshine-stt, windows-tts, uberduck-tts, kokoro/melo/chatterbox/fish/neural-onnx/neural-sidecar TTS, translate, voice-changer, keyboard, uwu). IPC and FS access are gated by capability JSON in `src-tauri/capabilities/` (Tauri 2 model) — prefer tightening `fs:scope`/plugin permissions over broad `*:default`.

**Data flow (host):** `Service_PubSub.publish` → PubSub-JS topics → `invoke("plugin:web|pubsub_broadcast")` → Rust → `emit("pubsub", …)` → JS listeners, plus PeerJS broadcast to clients. The Rust `web` plugin serves `127.0.0.1:<port>` (PeerJS signaling, pubsub WebSocket, static assets) — **localhost only**; treat any exposure beyond localhost as a high-risk change.

## Adding or changing a host service

A service exists across three registration points that must stay in sync — miss one and TypeScript exhaustiveness checks (or runtime routing) will catch it:
1. `Services.<id>` in `src/services-registry.ts` (if it needs a sidebar/inspector tab).
2. `Service_*_Schema` under `BackendSchema.services` in `src/core/schema.ts`, plus `PERSISTED_SERVICE_KEYS` — **only** for persisted services. Use `window.ApiServer.patchService(key, fn)` exclusively for these keys.
3. `ApiServer` field + `init` task in `src/core/index.ts` (`buildHostServiceInitTasks`), and a panel entry in `src/core/ui/servicePanels.tsx` (lazy inspector, nav, sidebar, title).

Non-persisted tabs (e.g. `voice_changer`) skip step 2's schema slice — do **not** call `patchService` for them. The full checklists and the STT/TTS/translation backend wiring tables are in `docs/DEVELOPER_GUIDE.md`.

## Key docs

- `docs/DEVELOPER_GUIDE.md` — architecture, host-service registration, connectivity audit, `src/client` rules (the deep reference).
- `docs/OPERATOR_REFERENCE.md` — STT/TTS backends, STT reliability, voice changer, local speech package.
- `docs/RELEASE_AND_ENGINEERING.md` — release smoke, threat model, dependency/perf audits.

## Gotchas

- **Secrets are build-time.** `pnpm build`/`pnpm tauri build` can embed `SIGIL_*`, legacy `CURSES_*`, and `VITE_*` from the environment into the frontend bundle. Never publish a `dist/` built with real secrets. Copy `.env.example` → `.env`; never commit `.env`.
- **RTL:** `ar`/`ur` are supported; on shared chrome prefer logical Tailwind utilities (`ms-`/`me-`, `ps-`/`pe-`, `start-`/`end-`) over physical `ml-`/`mr-` where mirroring matters. `src/i18n.ts` sets `<html dir lang>`.
- **Background input** (Windows low-level keyboard hook) is behind the `background_input` Cargo feature; it is a no-op init when off.
- Voice changer chunk sizes, ONNX execution provider (CPU/CUDA/DirectML), and neural TTS all select their backend at runtime — a single build ships all providers via `ort`. w-okada Socket.IO/REST servers are **not** drop-in compatible with Sigil's WebSocket sidecar.
