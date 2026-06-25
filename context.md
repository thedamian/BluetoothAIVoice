# CB08 / AIRec Voice Recorder — Protocol Reverse-Engineering Context

This file captures everything learned about how to pull recordings off the **CB08 / "AIRec"**
voice recorder, so the webapp (`index.html`) can talk to it without the vendor app.

---

## TL;DR — Three connection paths, ranked

1. **USB Mass Storage (works today, best).** The device mounts as an ExFAT drive at
   `/Volumes/AIRec`. Recordings live in `/record`. The webapp can import them via the
   **File System Access API** (`showDirectoryPicker()`) — already implemented as
   "Import from Drive". No BLE, no app, no protocol guessing.

2. **WiFi / HTTP (the vendor app's real method — promising for the webapp).** The official
   **SoniCloud** app connects to the device over WiFi at **`192.168.1.1:27689`** and speaks
   **HTTP**. This is the path that could make the webapp a true wireless solution. See
   "WiFi/HTTP protocol" below — NOT yet fully mapped.

3. **BLE (dead end).** The device's JieLi AC69x BLE stack uses an undocumented proprietary
   protocol. Every probe returns `[0,0]`. Abandoned — see "BLE findings" below.

---

## USB drive layout (confirmed)

```
/Volumes/AIRec/                      (ExFAT, USB Mass Storage)
├── MRECSET.txt                      time-set config: "TIME:8:00 2025/6/10 (Hour:Minute Year/Month/Day)"
└── record/
    ├── note20260506-022421.wav      16kHz mono 16-bit PCM
    ├── note20260506-022421.opus     proprietary "KA" container (NOT Ogg-Opus)
    ├── note20260506-022429.wav      (65 MB)
    ├── note20260506-022429.opus     (4 MB)
    └── ... 7 recordings total, each as .wav + .opus
```

- **Filename format:** `note<YYYYMMDD>-<HHMMSS>.wav` → parseable date.
- **WAV:** standard PCM, 16000 Hz, mono, 16-bit. Whisper-compatible.
- **OPUS:** header starts `4B 41 1E 0B...` ("KA") — a **proprietary JieLi container**, not
  standard Ogg-Opus. Whisper rejects it. Use the WAV.
- **25 MB limit gotcha:** 3 of the 7 WAVs exceed OpenAI Whisper's 25 MB cap (65/109/80 MB).
  The webapp now splits oversized PCM WAVs into <24 MB chunks in-browser, transcribes each,
  and stitches one diarization pass. (Verified: chunking preserves 100% of PCM bytes.)

---

## WiFi / HTTP protocol (from SoniCloud.app analysis) — THE KEY PATH

The vendor app is a Flutter iOS app shipped as a Mac Catalyst / wrapped binary:

```
/Applications/SoniCloud.app/Wrapper/Runner.app/
├── Runner                                  (native Swift/ObjC host binary)
└── Frameworks/App.framework/App            (compiled Dart / Flutter code)
```

Bundle id hints: `com.wanshi.voice_cloud`, Dart package `voice_cloud`.

### Confirmed facts
- **Device endpoint:** `192.168.1.1:27689` (found as a literal string in the `Runner` binary).
- **Protocol is HTTP:** Dart code contains HTTP verb tables (`GET POST HEAD PUT DELETE ...`),
  `content-length`, `content-type`, `user-agent` handling.
- **Device connects via WiFi hotspot:** the app declares the entitlement
  `com.apple.developer.networking.HotspotConfiguration` — i.e. it programmatically joins the
  device's WiFi network. (This is why the user asked: can we just have the user switch WiFi
  manually instead? See "Open questions".)
- **Relevant Dart source files (from string table):**
  - `package:voice_cloud/pages/file/mspage_device_file.dart`      ← device file browser
  - `package:voice_cloud/pages/file/mspage_recordpen_file.dart`   ← "record pen" file manager
  - `package:voice_cloud/pages/file/mspage_async_file_drawer.dart`
  - `package:voice_cloud/pages/file/mspage_cloud_file.dart`
- **Class/handler names:** `RecordPenProvider`, `RecordPenEventHandler`,
  `MsPageRecordPenFileManage`, `BaseDeviceInfo`, channel `com.wanshi.voice_cloud/recordPen`.
- **Cloud API (separate from device):** paths like `/api/passport/deviceSn`,
  `apiFolderDeleteFile` — these are the *cloud* backend, not the device. Ignore for local use.
- **File ops hinted:** `_getFile`, `_getFiles`, `getFileList`, `MagicNumber` (the last is from
  the Dart `mime` package's magic-number detection, probably a red herring).
- **TLS:** `kCFStreamSocketSecurityLevelTLSv1_2` appears near the port string, but that may be
  for cloud traffic. Unclear if device HTTP is plain or TLS. Device on `192.168.1.1` is most
  likely **plain HTTP** (self-signed TLS on an embedded recorder is unlikely).

### NOT yet known (needs more digging or a packet capture)
- Exact HTTP **paths** on the device (e.g. `GET /record/list`? `GET /file?name=...`?).
- Request/response **shape** (JSON? XML? raw?).
- Whether it needs any **auth/handshake** before serving files.
- The device's **WiFi SSID pattern** (so we can tell the user which network to join).

### How to finish mapping it (next steps)
1. **Packet capture (most reliable):** join the device WiFi on the Mac, run
   `sudo tcpdump -i en0 -A host 192.168.1.1 and port 27689 -w /tmp/cb08.pcap`, then use the
   app once to list+download a file. Open in Wireshark → "Follow HTTP Stream" reveals exact
   requests. (User doesn't want the app installed — but it IS installed as SoniCloud, so a
   one-time capture is possible without re-installing anything.)
2. **Deeper static extraction:** decompile the Dart `App` snapshot
   (e.g. `blutter` or `flutter-spy`) to recover the exact endpoint strings and request builders
   from `mspage_device_file.dart` / `RecordPenProvider`.
3. **Probe directly:** join the WiFi and curl common paths:
   `curl -v http://192.168.1.1:27689/` , `/list`, `/record`, `/sd`, `/api/...`.

---

## ⚠️ Webapp constraint: browser → `192.168.1.1:27689` (mixed content / CORS)

If we go the WiFi route from a **GitHub Pages (HTTPS)** webapp, two browser limits apply:
- **Mixed content:** an HTTPS page **cannot** `fetch('http://192.168.1.1:...')`. The browser
  blocks it. Options: serve the webapp over plain HTTP locally, OR the device must support
  HTTPS (unlikely), OR run the app from `http://localhost`.
- **CORS:** the device's HTTP server almost certainly won't send `Access-Control-Allow-Origin`,
  so `fetch()` from a browser origin will be blocked even over HTTP. Embedded servers usually
  don't do CORS. This may make a **pure browser webapp impossible** for the WiFi path without
  a tiny local proxy — needs verification against the device's actual response headers.

This is the crux of the user's "keep it just a webapp" question: **technically the WiFi path
may not work from a sandboxed browser** due to mixed-content + CORS, even though it's how the
native app does it. A packet capture will confirm whether the device sends permissive CORS
headers (some JieLi firmwares do send `Access-Control-Allow-Origin: *`).

---

## BLE findings (abandoned — documented so we don't repeat it)

Hardware: **JieLi AC69x** BLE SoC. Device name "CB08".

### Confirmed GATT map
```
0x3802 / 0x4a02  [read, write, notify]  ← bidirectional; returns [0,0] to everything
0xAE20 / 0xAE21  [write, writeWithoutResponse]
0xAE20 / 0xAE22  [read, notify]         ← 22-byte zero heartbeat ~1/sec
0xAE20 / 0xAE23  [notify]
0x180D / 0x2A37  [read, write, notify]  ← repurposed HR char; value changes each read (NOISE)
0x180D / 0x2A38  [read, write, notify]  ← repurposed HR char; NOT a file count (was mislabeled)
0x1805 / 0x2A08  [read, write, notify]  ← Date/Time (last recording)
0x1814 / 0x2ACF  [read, notify]         ← repurposed
0x180F / 0x2A19  [read, notify]         ← battery
```

### What was tried (all returned `[0,0]` or silence)
- JieLi RCSP frames (`AA 55 len_lo len_hi op ...payload checksum`), many opcodes
- Both `writeValue` and `writeValueWithoutResponse`
- Bare 1/2/4-byte commands, 20-byte MTU-padded writes
- AT/text commands (`AT+LIST`, JSON, etc.)
- Writes to 2A37/2A38/2A08; challenge-echo; two-step auth attempts
- Passive listening for spontaneous notifications

**Conclusion:** the BLE command protocol is proprietary and requires the exact byte sequence
the vendor app sends, which can't be brute-forced. The device's `0x2A37`/`0x2A38` "file count"
was a **misread** — those are repurposed Heart-Rate sensor bytes that change every read.

---

## Current webapp state (`index.html`)

- **"Import from Drive"** button (primary): `showDirectoryPicker()` → walks folder → de-dupes
  .wav/.opus twins (keeps .wav) → skips already-transcribed → splits oversized WAVs → transcribes
  → one diarization pass → saves to IndexedDB. **This is the working path today.**
- **"BLE (experimental)"** button: kept but deprioritized (returns nothing useful).
- **"Upload File"**: manual drag/drop or picker fallback.
- Transcription via OpenAI-compatible `/audio/transcriptions` + `/chat/completions` for
  diarization & 9-section summary.

---

## Open questions for the user

1. **Is the device WiFi SSID always the same?** If so, we can tell the user "join network X"
   and skip the HotspotConfiguration entitlement (which a webapp can't use anyway).
2. **Is `192.168.1.1:27689` plain HTTP or TLS?** Determines mixed-content handling.
3. **Does the device send CORS headers?** Determines whether a pure browser webapp can fetch
   from it at all. ← This is the make-or-break unknown for the "keep it a webapp" goal.
