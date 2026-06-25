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
- **Drive import (primary path)** — `importFromDrive()` uses the File System Access API (`showDirectoryPicker()`) to read recordings directly off the device when it is mounted as a USB drive (e.g. `/Volumes/AIRec`, ExFAT). Walks the folder, de-dupes `.wav`/`.opus` twins (keeps `.wav`), skips already-transcribed files, splits oversized WAVs, transcribes, runs one diarization pass, saves to IndexedDB. This is the working path.
- **`BLE` (experimental/deprecated)** — Web Bluetooth connection to a device named `"CB08"`. The JieLi BLE command protocol is undocumented and every probe returns `[0,0]`; kept for reference only. Use drive import instead.
- **`API`** — Two-call transcription flow: (1) POST audio to `/audio/transcriptions` with `verbose_json`; (2) if no speaker labels detected, POST transcript to `/chat/completions` for diarization. Split into `transcribeRaw()` + `diarize()` so chunked uploads get one consistent diarization pass. Separate `summarize()` method uses a fixed 9-section business analyst prompt
- **UI layer** — vanilla DOM manipulation; no framework. `refreshGrid()` rebuilds the recordings grid from IndexedDB on every change

See **`context.md`** for full device protocol reverse-engineering notes (USB drive layout, the SoniCloud app's WiFi/HTTP path at `192.168.1.1:27689`, and the abandoned BLE investigation).

## Key Behaviors

**Connection methods**: The CB08/AIRec exposes recordings three ways — (1) **USB Mass Storage** (mounts as ExFAT drive, `/record/note<YYYYMMDD>-<HHMMSS>.wav`), used by drive import; (2) **WiFi/HTTP** at `192.168.1.1:27689` (how the vendor SoniCloud app works — not yet implemented, blocked by browser mixed-content + CORS questions); (3) **BLE** (dead end). See `context.md`.

**25MB transcription limit**: WAVs over OpenAI's 25MB cap are split in-browser by `splitWavPCM()` into <24MB PCM chunks (parses the RIFF header, slices on sample boundaries, rebuilds valid WAV headers via `buildWavBlob()`). Chunk transcripts are concatenated then diarized once. The `.opus` files on the device use a proprietary "KA" container (not Ogg-Opus) and are NOT API-compatible — always prefer the `.wav`.

**Skip-already-transcribed logic**: `processAudio()` checks `DB.getByFilename()` before transcribing. A file is only skipped if `existing.transcript` is truthy.

**BLE protocol**: The CB08 uses a JieLi AC69x BLE SoC. Confirmed GATT services (discovered via `chrome://bluetooth-internals`):
- `0xAE20` (`CB08_JIELI_SVC`) — JieLi proprietary command/control channel
- `0x3802` (`CB08_FILE_SVC`) — custom file-transfer service
- `0x1805` — Current Time (used for recording timestamps)
- `0x180D` — Heart Rate (repurposed for metadata)
- `0x1814` — Running Speed & Cadence (repurposed control)
- `0x180F` — Battery

The app tries JieLi RCSP binary protocol (framed with `0xAA 0x55` header) on `0xAE20`/`0x3802`, then falls back to text AT commands, then single-byte probes. Once the exact characteristic UUIDs and command bytes are confirmed from BLE logs, update `setupCB08()` and `cb08ListFiles()` in the `BLE` module.

**Diarization**: Happens in `API.transcribe()`. If `/audio/transcriptions` response text matches `/\b(speaker\s*\d|person\s*\d|spk_\d)/i`, it's treated as already diarized. Otherwise a second chat completion call adds speaker labels.

**Audio storage**: `audioData` (ArrayBuffer) is stored in IndexedDB to enable re-transcription without re-downloading from the device. Files over 50MB are not stored (transcript + summary still are).

## Deployment

Push `index.html` to a GitHub repo, enable GitHub Pages on `main` branch. No build step needed. Web Bluetooth only works in Chrome/Edge (desktop or Android).
