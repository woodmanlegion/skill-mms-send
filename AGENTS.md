# AGENTS.md — mms-send

Dispatch guide for the AI agent. Read this to know when and how to call `mms-send-*` scripts.

## Trigger Conditions

Activate this skill when the user or an automation asks to:

- "Send an audio / voice message to [phone number]"
- "Text [person] this [recording / image / file]"
- "MMS [number] [file]"
- "Send this as an attachment over SMS"
- "Forward this audio to [number]"

**Do NOT activate for:**
- Plain text SMS → use `sms-message` instead
- Generating speech from text → use `mms-audio` (it handles TTS + send in one step)
- Reading received MMS → use `mms-fetch`
- A URL, not a local file → download the file first, then call `mms-send-smart`

---

## Required Inputs

Before calling, confirm you have:

1. **Phone number** in E.164 format (`+1XXXXXXXXXX`). If only a name is given, look it up or ask.
2. **Local file path** that exists on device. If the user referenced a URL, download it first.

---

## Which Variant To Call

```
mms-send-smart   →  default, always try first
mms-http-send    →  if smart fails: "Could not find Send button" or MediaStore timeout
mms-send-auto    →  do not use without explicit --coords; opens Messages but does not send
```

---

## Call Pattern

```bash
# Standard — audio, image, or video file
mms-send-smart +17066228333 /path/to/file.mp3

# Carrier-direct fallback
mms-http-send +17066228333 /path/to/file.mp3

# TTS text-to-MMS (prefer mms-audio for this)
mms-audio +17066228333 "Your spoken message here"
```

---

## Success / Failure Handling

**Success:** exits 0, last line is `[mms] Done. MMS sent to <phone>`

**Failure responses:**

| Error message | What to tell the user | What to try next |
|---|---|---|
| `File not indexed in MediaStore` | "MMS indexing timed out — trying carrier-direct send" | `mms-http-send` |
| `Could not find Send button` | "Google Messages UI changed — trying carrier-direct send" | `mms-http-send` |
| `Failed to add attachment` | "File format rejected — trying as MP3" | Convert to mp3 first, retry |
| Screen locked (inferred from failure) | "MMS needs the screen unlocked — please unlock and I'll retry" | Ask user to unlock; retry |
| `mms-http-send` 4xx | "Carrier MMSC unreachable — is mobile data on?" | Confirm data is active |

---

## Timing

| Variant | Typical duration |
|---------|----------------|
| `mms-send-smart` | 10–25 seconds |
| `mms-http-send` | 3–10 seconds |

Do not time out before 30 seconds. Longer runs usually mean MediaStore is slow to index.

---

## Constraints

- Do not quote file paths with shell special characters without escaping
- Do not assume the file is pre-indexed — the script handles MediaStore registration
- Do not call `mms-send-auto` without screen coordinates; it opens Messages but leaves attachment to the user

---

## Context: Why This Requires Root on LineageOS

Android reserves MMS send APIs (`SmsManager.sendMultimediaMessage()`) for system apps. Non-system apps need user confirmation. These scripts use root (`su`) to drive system-level tools directly:

- `am start` — intent dispatch with STREAM extra (content URI)
- `uiautomator` + `input tap` — UI probe and gesture injection into Google Messages
- `SO_BINDTODEVICE` — raw socket bound to mobile data interface for direct MMSC POST

This is specific to rooted LineageOS with Magisk. The same approach will not work on stock, unrooted Android.
