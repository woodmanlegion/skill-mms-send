# AGENTS.md — mms-send

Dispatch guide for the AI agent. Read this to know when and how to send MMS from this device.

## Trigger Conditions

Activate this skill when the user or an automation asks to:

- "Send an audio / voice message to [phone number]"
- "Text [person] this [recording / image / file]"
- "MMS [number] [file]"
- "Send this as an attachment over SMS"
- "Forward this audio to [number]"

**Do NOT activate for:**
- Plain text SMS → use `sms-message`
- Generating speech from text → use `mms-audio` (handles TTS + send in one step)
- Reading a received MMS → use `mms-fetch`
- A URL rather than a local file → download it first, then call `mms-http-send`

---

## Required Inputs

Before calling, confirm:

1. **Phone number** in E.164 format (`+1XXXXXXXXXX`). If only a name is given, look it up or ask.
2. **Local file path** that exists on device. If the user gave a URL, download it first.

---

## Which Tool to Call

```
mms-http-send    →  PRIMARY — always try first; no screen, no app, no root
mms-send-smart   →  FALLBACK — if http-send fails or carrier is not AT&T
mms-send-auto    →  do not use without explicit --coords; opens Messages but does not send
mms-audio        →  use when input is text, not a file (generates speech first)
```

---

## Call Pattern

```bash
# Primary — any audio, image, or video file
mms-http-send +17066228333 /path/to/file.mp3

# Fallback — non-AT&T carrier or http-send fails
mms-send-smart +17066228333 /path/to/file.mp3

# Text-to-speech then send (delegates to http-send internally)
mms-audio +17066228333 "Your spoken message here"
```

---

## Success / Failure Handling

**Success:** exits 0; output contains `[mms] Sent successfully.` and/or `m-send-conf received ✓`

**Failure responses:**

| Error / symptom | What to tell the user | Next step |
|---|---|---|
| `curl error` / network timeout | "Mobile data seems off — is airplane mode on?" | Ask user to enable mobile data; retry |
| HTTP 4xx from MMSC | "Carrier proxy rejected the message" | Check `PROXY_IP_FALLBACK`; try `mms-send-smart` |
| Sent OK but recipient didn't receive | "Sent — may be a from-number mismatch. Is `OPENCLAW_MMS_FROM` set correctly?" | Verify env var matches SIM number |
| `Could not find Send button` (smart) | "Google Messages UI changed — switching to carrier-direct send" | Use `mms-http-send` |
| `File not indexed` (smart) | "MediaStore indexing timed out — switching to carrier-direct send" | Use `mms-http-send` |
| File too large (audio/video) | "File is over the 900 KB carrier limit — can I compress it?" | Use ffmpeg to reduce bitrate; retry |

---

## Timing

| Tool | Typical duration |
|------|----------------|
| `mms-http-send` | 3–10 seconds |
| `mms-send-smart` | 15–30 seconds |

Do not time out before 35 seconds.

---

## Constraints

- `mms-http-send` requires mobile data to be active (`rmnet_data1` up). WiFi alone won't reach the MMSC.
- `mms-send-smart` requires the screen to be unlocked.
- Do not call `mms-send-auto` without screen coordinates; it opens Messages but does not send.
- Images over 900 KB are auto-resized by `mms-http-send`; audio/video are not — pre-convert if needed.

---

## Context: Why Two Paths Exist

**`mms-http-send`** implements OMA MMS 1.2 (WAP binary PDU) and POSTs directly to the AT&T MMSC via `curl --interface rmnet_data1`. It bypasses Android's MMS API entirely. No root, no screen, no app — just curl and Python. AT&T-specific (hardcoded proxy + MMSC).

**`mms-send-smart`** drives Google Messages via Android intents and `uiautomator` UI probe. Works on any carrier because Google Messages handles the carrier protocol. Requires root (for `am start`, `input tap`) and an unlocked screen (for intent delivery).

Both avoid `SmsManager.sendMultimediaMessage()`, which is gated to system apps on Android.
