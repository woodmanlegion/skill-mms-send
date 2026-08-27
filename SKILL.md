---
name: mms-send
description: "Send an MMS attachment (audio, image, video) to a phone number from the command line on Android/Termux. Primary path posts directly to the carrier MMSC — no screen, no UI, no Google Messages required."
metadata:
  status: "stable"
  platform: "android"
  requires:
    bins: ["mms-http-send", "curl", "python3", "ffmpeg"]
    root: false
    packages: []
  openclaw:
    emoji: "📨"
    platform: "android"
    notes: "AT&T carrier. Primary path (mms-http-send) is fully headless — no screen or Google Messages needed. mms-send-smart is the fallback for non-AT&T carriers."
---

# mms-send

Send an MMS attachment — audio, image, or video — to any phone number from the terminal. No screen interaction, no app required.

## Quick Start

```bash
mms-http-send +17066228333 /path/to/audio.mp3
```

Encodes a WAP MMS PDU and posts it directly to the AT&T MMSC over the mobile data interface. Exits in 3–10 seconds. No UI. No Google Messages. Works while the screen is off and locked.

---

## When To Use Each Tool

| Situation | Tool |
|-----------|------|
| Send audio / image / video to a phone number | `mms-http-send <phone> <file>` ← **default** |
| Carrier is not AT&T | `mms-send-smart <phone> <file>` |
| Generate speech from text then send | `mms-audio <phone> "text"` (wraps http-send) |
| Read a received MMS | `mms-fetch` |
| Plain text SMS | `sms-message` |

---

## Usage

### Primary: `mms-http-send`

```bash
mms-http-send <phone> <file> [mime-type]
```

**Arguments:**
- `phone` — E.164 format (`+17066228333`)
- `file` — local path; extension determines MIME type automatically
- `mime-type` — optional override if extension is ambiguous

**Examples:**
```bash
mms-http-send +17066228333 /sdcard/voice.m4a
mms-http-send +17066228333 /sdcard/photo.jpg
mms-http-send +17066228333 /tmp/audio.mp3 audio/mpeg
```

**Expected output:**
```
[mms] To:   +17066228333
[mms] File: /sdcard/voice.m4a (42.1 KB, audio/mp4)
[mms] Via:  proxy.mobile.att.net:80 → http://mmsc.mobile.att.net
[mms] Encoding PDU...
[mms] PDU size: 43218 bytes  txn-id: xk7m2qr9dple
[mms] Posting via rmnet_data1...
[mms] Response: HTTP/1.1 200 OK
[mms] Carrier confirmed: m-send-conf received ✓
[mms] Sent successfully.
```

Exits 0 on success, non-zero with a clear error on failure.

**Configuration (environment variables):**

| Variable | Default | Description |
|----------|---------|-------------|
| `OPENCLAW_MMS_FROM` | `+17624346188` | Your phone number (sender address in PDU) |
| `OPENCLAW_MMS_INTERFACE` | `rmnet_data1` | Mobile data interface name |
| `OPENCLAW_MMS_MMSC_PROXY` | `proxy.mobile.att.net` | AT&T WAP proxy host |
| `OPENCLAW_MMS_MAX_BYTES` | `900000` | Auto-resize threshold for images |

Set `OPENCLAW_MMS_FROM` to your number — it is the sender identity embedded in the PDU. The default is the original device number; override it to match your SIM.

---

### Fallback: `mms-send-smart` (non-AT&T / Google Messages path)

```bash
mms-send-smart <phone> <file>
```

Drives Google Messages via Android intents and UI automation. Works on any carrier but requires the screen to be unlocked.

**When to use:** carrier is not AT&T, or `mms-http-send` returns a non-200 status.

**How it works:**
1. Copies file to `/storage/emulated/0/Download/`
2. Registers in MediaStore via `termux-media-scan` (polls up to 3 × 2s)
3. Sends `android.intent.action.SEND` to Google Messages `ShareIntentActivity`
4. `uiautomator dump` finds the Send button dynamically (by resource-id or content-desc)
5. `input tap` triggers send

Requires `root` for `am start`, `uiautomator`, and `input tap`. Screen must be unlocked.

---

### Legacy: `mms-send-auto` (coordinate-based)

Use only if `mms-send-smart` fails due to a Google Messages UI change. Requires hardcoded screen coordinates from `mms-calibrate`. Not recommended for automation.

---

## Supported File Types

| Type | Extensions | Notes |
|------|-----------|-------|
| Audio | mp3, m4a, ogg, aac, wav, amr | All sent as-is; no auto-conversion in http-send |
| Images | jpg, jpeg, png, gif, webp | Auto-resized if over 900 KB |
| Video | mp4, 3gp, mkv | Carrier size limit ~900 KB |
| Contact | vcf | vCard |
| Text | txt | Plain text |

**Size limit:** AT&T soft limit is ~1 MB; script targets 900 KB. Images are auto-resized; audio/video are not — pre-convert to a smaller format if needed (`ffmpeg -q:a 5 -ar 16000`).

---

## How `mms-http-send` Works

`mms-http-send` implements the OMA MMS 1.2 / WAP-209 protocol by hand in Python:

1. **MIME detection** — maps file extension to MIME type
2. **SMIL generation** — builds a minimal SMIL presentation layer (layout varies by media category: image/audio/video/text)
3. **PDU assembly** — encodes WAP binary envelope: message-type, transaction-id, MMS version, date, from/to addresses (E.164 + `/TYPE=PLMN`), message-class, expiry, priority, content-type
4. **Multipart body** — two parts: SMIL part (`application/smil`) + media part with content-id and content-location matching Android's naming convention (`audio000000.mp3`, etc.)
5. **POST** — writes PDU to a temp file, calls `curl --interface rmnet_data1 --proxy <proxy_ip>:80` targeting `http://mmsc.mobile.att.net`. The `--interface` flag forces the request over mobile data (bypassing WiFi). Root is not required for this step; curl handles interface binding via the normal socket API.
6. **Response check** — expects HTTP 200 or 204; also checks for WAP binary `m-send-conf` (0x81) in the response body as carrier confirmation

---

## Prerequisites

| Requirement | Used by | Notes |
|-------------|---------|-------|
| `python3` | mms-http-send | Runs the PDU encoder |
| `curl` | mms-http-send | POSTs to MMSC; must support `--interface` |
| `ffmpeg` | image auto-resize | Optional; only for oversized images |
| Mobile data active | mms-http-send | WiFi alone is not enough — `rmnet_data1` must be up |
| Root (`su`) | mms-send-smart only | Not needed for http-send |
| Screen unlocked | mms-send-smart only | Not needed for http-send |
| Google Messages | mms-send-smart only | Not needed for http-send |

---

## Limitations

| Constraint | Impact | Workaround |
|-----------|--------|------------|
| AT&T MMSC only | `mms-http-send` uses hardcoded AT&T proxy | Use `mms-send-smart` on other carriers |
| `OPENCLAW_MMS_FROM` must match your SIM | Wrong sender identity in PDU | Set the env var to your E.164 number |
| ~900 KB attachment limit | Large audio/video rejected | Pre-convert; images auto-resize |
| Mobile data must be active | http-send fails if on WiFi-only | Disable WiFi or verify `rmnet_data1` is up |
| `mms-send-smart` needs screen unlocked | Lock screen blocks intents | Use http-send instead |

---

## Why Not `SmsManager.sendMultimediaMessage()`?

Android reserves that API for system apps. Non-system apps require user confirmation. `mms-http-send` bypasses the Android MMS stack entirely by implementing the carrier protocol directly. `mms-send-smart` works around the restriction by driving Google Messages (which has the system privilege) via intents and UI automation.

---

## Related Skills

- **mms-audio** — TTS from text → `mms-http-send` (one step)
- **mms-fetch** — reads received MMS from the telephony database
- **sms-message** — plain text SMS via `termux-sms-send`
- **termux-communications** — full Termux messaging reference

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `curl error` / network timeout | Mobile data off or wrong interface | Confirm `ip link show rmnet_data1`; toggle mobile data |
| HTTP 4xx from MMSC | Proxy IP stale or MMSC reject | Check `PROXY_IP_FALLBACK` in script; try a fresh DNS lookup of `proxy.mobile.att.net` |
| Sent successfully but not received | Wrong `OPENCLAW_MMS_FROM` | Set env var to match your SIM's E.164 number |
| Image not showing | MIME mismatch or oversized | Verify extension; pre-resize: `ffmpeg -i in.jpg -vf scale=960:-2 out.jpg` |
| `mms-send-smart`: `Could not find Send button` | Google Messages UI changed | Use `mms-http-send` instead |
| `mms-send-smart`: `File not indexed` | MediaStore flush lag | Run `termux-media-scan <file>` manually; retry |
