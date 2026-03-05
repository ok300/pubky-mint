# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Pubky Mint is an RSS-like web interface for browsing the activity of Pubky network users you follow. It is a **frontend-only, single-file application** — the entire app lives in `index.html` with no build step, no npm, and no framework.

## Running Locally

Just serve the project root with any static file server:

```bash
python3 -m http.server 8080
# or
npx serve .
```

The `pkg/` directory contains pre-built WebAssembly bindings (from Pubky SDK v0.6.0) that are checked in and ready to use. There are no tests and no linter configured.

## Architecture

### Single-file structure

All application logic, styles, and markup live in `index.html`. The `<script type="module">` block at the bottom is the entire JavaScript codebase (~420 lines). All CSS is inline in `<style>`.

### External dependencies

- **`./pkg/pubky_wasm.js`** — WASM bindings for the Pubky SDK (Rust → WebAssembly, pre-built). Provides `Pubky`, `AuthFlowKind`.
- **`https://esm.sh/qrcode@1.5.4`** — QR code generation (ESM CDN, no local install).

### Key APIs consumed

| Endpoint | Purpose |
|---|---|
| `https://homeserver.pubky.app/events-stream?user=<z32>` | Fetch user events (SSE/text format, paginated by cursor) |
| `https://nexus.pubky.app/v0/user/<z32>/following` | Get list of followed user IDs |
| `https://nexus.pubky.app/v0/stream/users/by_ids` | Batch fetch user details (name, etc.) |
| `https://nexus.pubky.app/v0/user/<z32>/details` | Fetch single user details |
| `https://nexus.pubky.app/static/avatar/<z32>` | User avatar image |
| `pubky.publicStorage.getJson(uri)` | Fetch JSON content via WASM SDK |
| `pubky.publicStorage.getText(uri)` | Fetch text content via WASM SDK |

### UI layout

Three-column layout (all rendered directly into the DOM without a virtual DOM):

- **Left (`#col-left`)**: List of followed users with unread indicators
- **Middle (`#col-mid-wrap`)**: Event stream for the selected user, filterable by type (post/tag/follow/bookmark)
- **Right (`#col-right`)**: Parsed + raw content view for the selected event

### Checkpoint system

Checkpoints track "read" progress per user. `checkpoints` is an object mapping `z32` (user ID) → cursor string. `latestCursors` maps `z32` → most recently seen cursor. A user is marked unread when `latestCursors[z32] > checkpoints[z32]`. Checkpoints can be exported/imported as JSON files via Save/Load buttons.

### Auth flow

On load, the WASM `Pubky` client is initialized. If no session exists in `localStorage`, a QR code auth flow is started via `pubky.startAuthFlow()` and awaits approval. The session is then persisted to `localStorage` for subsequent visits.

### Event kinds

Events from the homeserver identify their type by URI path segments:
- `/posts/` — user posts
- `/tags/` — user tags
- `/follows/` — follow relationships
- `/bookmarks/` — bookmarked content
- anything else — metadata (settings, last_read, etc.)

### WASM build (CI only)

The `pkg/` directory is not rebuilt locally — it is built in CI (`.github/workflows/deploy.yml`) from `pubky/pubky-core` at tag `v0.6.0` using `wasm-pack build --target web`. To update the SDK version, change `WASM_LIB_REF` in the workflow and delete the cached `pkg/` directory.

## User IDs

Pubky user IDs appear in two forms: a raw public key string (`userId`) and a `z32`-encoded form (`userZ32`). The WASM SDK provides `.z32()` on the public key object. URI parsing uses `uriZ32(uri)` which strips `pubky://` and takes the first path segment.
