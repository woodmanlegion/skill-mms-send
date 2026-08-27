---
name: mms-send
description: "Send an MMS attachment (audio, image, video) to a phone number from the command line on rooted Android/Termux. Primary path is headless — no screen interaction required."
metadata:
  status: "stable"
  platform: "android"
  requires:
    root: true
    bins: ["mms-send-smart", "mms-http-send", "ffmpeg", "python3"]
    apps: ["com.google.android.apps.messaging"]
    packages: ["termux-api"]
  openclaw:
    emoji: "📨"
    platform: "android"
    notes: "Rooted LineageOS only. mms-send-smart is the primary path; mms-http-send bypasses Google Messages via direct MMSC POST."
---

# mms-send

Send an MMS attachment — audio, image, or video — directly to a phone number from the terminal.

## Quick Start

```bash
mms-send-smart +17066228333 /path/to/audio.mp3
```

No screen interaction required. The file lands in the recipient's Google Messages as an attachment. WAV files auto-convert to MP3 to stay within carrier size limits.

---

## When To Use This Skill

| Situation | Command |
|-----------|---------|
| Send audio/image/video to a phone number | `mms-send-smart <phone> <file>` |
| Google Messages unavailable or Send button changed | `mms-http-send <phone> <file>` |
| Need to generate speech first, then send | Use `mms-audio` skill instead |
| Receive / read an MMS | Use `mms-fetch` |
| Plain text SMS | Use `sms-message` |

---

## Usage

### Primary: `mms-send-smart` (headless)

```bash
mms-send-smart <phone> <file>
```

**Arguments:**
- `phone` — E.164 format (`+17066228333`)
- `file` — local path to audio, image, or video file

**Examples:**
```bash
mms-send-smart +17066228333 /sdcard/voice.m4a
mms-send-smart +12025551234 /sdcard/DCIM/photo.jpg
```

**Expected output:**
```
[mms] Converting WAV to MP3...          # only for .wav input
[mms] Copying to shared storage (42K)...
[mms] Registering in MediaStore...
[mms] Content URI: content://media/external/audio/media/1234
[mms] Opening Messages with attachment...
[mms] Waiting for UI...
[mms] Tapping Send at (540, 2100)...
[mms] Done. MMS sent to +17066228333
```

Exits 0 on success, non-zero with a descriptive error on failure.

---

### Advanced: `mms-http-send` (direct carrier)

```bash
mms-http-send <phone> <file>
```

Posts the MMS PDU directly to the AT&T MMSC — no Google Messages involved. Uses root to bind a socket to the mobile data interface (`rmnet_data1`). Auto-resizes oversized files.

**When to use:** Google Messages Send button is not found; carrier is AT&T; you want to bypass Android MMS stack entirely.

**Environment overrides:**
```bash
OPENCLAW_MMS_INTERFACE=rmnet_data2  # mobile interface (default: rmnet_data1)
OPENCLAW_MMS_MMSC_PROXY=proxy.mobile.att.net  # MMSC endpoint
OPENCLAW_MMS_MAX_BYTES=290000       # resize threshold in bytes
```

---

### Legacy: `mms-send-auto` (coordinate-based)

```bash
mms-send-auto <phone> <file>                            # opens Messages, manual attach
mms-send-auto <phone> <file> --coords x1,y1,...,x5,y5  # fully automated
```

Uses hardcoded screen coordinates to tap through the attachment UI. Device-specific. Requires calibration with `mms-calibrate`. Use only as a last resort.

---

## Supported File Types

| Type | Extensions | Notes |
|------|-----------|-------|
| Audio | mp3, m4a, wav, ogg, aac | WAV auto-converts to MP3 |
| Images | jpg, jpeg, png, gif, webp, bmp | |
| Video | mp4, m4v, webm, mkv | ~300 KB carrier limit applies |

Size limit: ~300 KB for audio/video on most carriers. `mms-http-send` auto-resizes; `mms-send-smart` does not — convert to a smaller format first if needed.

---

## How `mms-send-smart` Works

1. **Stage** — copies file to `/storage/emulated/0/Download/mms_send_<ts>_<name>`
2. **MediaStore** — runs `termux-media-scan`, then polls `content://media/external/...` via `/system/bin/content query` (up to 3 × 2s attempts) to obtain a content URI
3. **Wake screen** — sends `KEYCODE_WAKEUP` so the intent can reach the foreground
4. **Intent** — `am start android.intent.action.SEND` targets `ShareIntentActivity` in Google Messages with `STREAM` extra (content URI) and `address` extra (phone number)
5. **UI probe** — `uiautomator dump` captures the live UI tree; Python parses XML for the Send button by resource-id (`Compose:Draft:Send`) or content-desc (`Send MMS`)
6. **Send** — `input tap <cx> <cy>` triggers the send

The Send button is found dynamically, not by hardcoded coordinates — making this approach robust across screen sizes and orientations.

---

## How `mms-http-send` Works

Encodes a WAP binary MMS M-Send.req PDU and POSTs it directly to the carrier MMSC:

1. Reads file; auto-resizes if over `OPENCLAW_MMS_MAX_BYTES`
2. Generates SMIL presentation layer
3. Assembles multipart MMS body (headers + SMIL part + media part)
4. Encodes WAP binary envelope (content-type, from, to, date, transaction-id, message-class, priority)
5. Opens raw socket, calls `SO_BINDTODEVICE` on the mobile interface (requires CAP_NET_RAW / root), POSTs to `http://proxy.mobile.att.net:80`

---

## Prerequisites

| Requirement | Used by | How to check |
|-------------|---------|--------------|
| Root (`su`) | all | `su -c id` |
| Google Messages | mms-send-smart | `pm list packages \| grep messaging` |
| `ffmpeg` | WAV→MP3 conversion | `which ffmpeg` |
| `python3` | XML/UI parsing | `which python3` |
| `termux-media-scan` | MediaStore indexing | `which termux-media-scan` |
| Screen unlocked | mms-send-smart | n/a — lock screen blocks intent |
| Mobile data on | mms-http-send | `ip link show rmnet_data1` |

---

## Limitations

| Constraint | Impact | Workaround |
|-----------|--------|------------|
| Screen must be unlocked | Lock screen blocks `am start` | Use `KEYCODE_WAKEUP` + no PIN lock, or `mms-http-send` |
| Google Messages UI changes | Send button resource-id or content-desc may shift | Fall back to `mms-http-send` |
| ~300 KB attachment limit | Large audio/video rejected by carrier | Convert before send; `mms-http-send` auto-resizes |
| AT&T MMSC only | `mms-http-send` is carrier-specific | `mms-send-smart` works on any carrier |
| aarch64 device | Android SDK build-tools unavailable | Scripts use `am`/`uiautomator`/`input` directly — no SDK needed |

---

## Why Not `SmsManager.sendMultimediaMessage()`?

Android gates MMS APIs to system apps:
- Non-system apps cannot call `sendMultimediaMessage()` without user confirmation
- `app_process` + reflection requires DEX bytecode, not JVM bytecode
- Android SDK build-tools are x86_64-only; this device is aarch64

These scripts bypass the problem:
- `mms-send-smart` drives Google Messages (a system-privileged app) via intents and UI automation
- `mms-http-send` encodes and posts the MMSC PDU directly, skipping the Android MMS stack entirely

---

## Related Skills

- **mms-audio** — generates TTS speech from text, then calls `mms-send-smart`
- **mms-fetch** — reads received MMS from the telephony database (requires root)
- **sms-message** — sends plain text SMS via `termux-sms-send`
- **termux-communications** — umbrella reference for all Termux messaging tools

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `File not indexed in MediaStore` | MediaStore flush lag | Run `termux-media-scan <file>` manually; retry |
| `Could not find Send button` | Google Messages UI updated | Try `mms-http-send` |
| `Failed to add attachment` | MIME type rejected | Convert to mp3; verify extension matches content |
| Script exits 0, no MMS received | Screen was locked at intent time | Unlock screen; rerun |
| `mms-http-send` 4xx error | MMSC proxy unreachable | Confirm mobile data is active; check `OPENCLAW_MMS_INTERFACE` |
| WAV not converting | `ffmpeg` not on PATH | `pkg install ffmpeg` or use a pre-converted mp3 |
