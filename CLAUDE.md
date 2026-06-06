# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Single-file static webapp (`index.html`) designed for GitHub Pages. No build step, no dependencies, no backend. Everything runs in the browser.

## Development

Serve locally with any static file server:
```bash
npx serve .
# or
python3 -m http.server 8080
```

Web Bluetooth API requires HTTPS in production — GitHub Pages satisfies this. Locally, `localhost` is also allowed.

## Architecture

All application code lives in `index.html` as a single self-contained file, structured into logical modules inside a `<script>` block:

- **`Config`** — reads/writes API settings (baseURL, apiKey, model, chatModel) to `localStorage`
- **`DB`** — IndexedDB wrapper (`CB08Transcriber` database, `recordings` store). Each record has: `id`, `filename`, `date`, `size`, `transcript`, `rawTranscript`, `summary`, `meta`, `audioData` (ArrayBuffer, capped at 50MB), `downloadedAt`, `transcribedAt`, `summarizedAt`, `error`
- **`BLE`** — Web Bluetooth connection to a device named `"CB08"`. Tries Nordic UART Service (NUS) UUIDs first; enumerates and logs all discovered GATT services/characteristics for debugging unknown protocols
- **`API`** — Two-call transcription flow: (1) POST audio to `/audio/transcriptions` with `verbose_json`; (2) if no speaker labels detected, POST transcript to `/chat/completions` for diarization. Separate `summarize()` method uses a fixed 9-section business analyst prompt
- **UI layer** — vanilla DOM manipulation; no framework. `refreshGrid()` rebuilds the recordings grid from IndexedDB on every change

## Key Behaviors

**Skip-already-transcribed logic**: `processAudio()` checks `DB.getByFilename()` before transcribing. A file is only skipped if `existing.transcript` is truthy.

**BLE protocol**: The CB08's exact GATT protocol is unknown. The app tries NUS commands (`AT+LIST`, `LIST`, `FILES`, etc.) and logs all service/characteristic UUIDs to the BLE Log panel + browser console. When the real UUIDs are discovered, update the `CANDIDATE_SVCS` array and the NUS command strings in the `BLE` module.

**Diarization**: Happens in `API.transcribe()`. If `/audio/transcriptions` response text matches `/\b(speaker\s*\d|person\s*\d|spk_\d)/i`, it's treated as already diarized. Otherwise a second chat completion call adds speaker labels.

**Audio storage**: `audioData` (ArrayBuffer) is stored in IndexedDB to enable re-transcription without re-downloading from the device. Files over 50MB are not stored (transcript + summary still are).

## Deployment

Push `index.html` to a GitHub repo, enable GitHub Pages on `main` branch. No build step needed. Web Bluetooth only works in Chrome/Edge (desktop or Android).
