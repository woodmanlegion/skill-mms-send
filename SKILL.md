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

> **Platform:** Android/Termux only (rooted or unrooted, AT&T SIM). Before proceeding, confirm:
> ```bash
> [[ "$(uname -o)" == "Android" ]] && echo "OK" || echo "ERROR: not Android — skill unavailable here"
> ```

---

## Agent Dispatch Guide

> **For agent frameworks that load `AGENTS.md`** (OpenClaw and others as the standard evolves): the full dispatch guide — trigger conditions, tool priority, failure handling, and timing — is in [`AGENTS.md`](./AGENTS.md). The summary below is provided for frameworks that do not yet load it.

**Activate this skill when the user asks to:**
- send an audio / voice message / recording to a phone number
- MMS a file to a number
- send a photo or video via SMS/MMS

**Do not activate for:**
- plain text SMS → use `sms-message`
- text-to-speech generation → use `mms-audio`
- reading a received MMS → use `mms-fetch`
- a URL rather than a local file → download it first

**Required before calling:** phone number (E.164) and a local file path that exists.

**Tool priority:** `mms-http-send` first (headless, no root, any screen state) → `mms-send-smart` fallback (non-AT&T carriers, needs screen unlocked).

---

## Quick Start

```bash
mms-http-send +17066228333 /path/to/audio.mp3
```

Encodes a WAP MMS PDU and posts it directly to the AT&T MMSC over the mobile data interface. Exits in 3–10 seconds. No UI. No Google Messages. Works while the screen is off and locked.

---

## Setup

Scripts must be on your `PATH`. Options:

```bash
# Option A — add this repo's bin/ to PATH (add to .bashrc)
export PATH="$PATH:/path/to/skill-mms-send/bin"

# Option B — symlink into an existing PATH directory
ln -s /path/to/skill-mms-send/bin/mms-http-send ~/.local/bin/mms-http-send

# Option C — Termux (scripts already in ~/Scripts if installed via tclaw)
# ~/Scripts is on PATH by default
```

**Termux dependencies:**
```bash
pkg install python curl ffmpeg termux-api
```

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

**Verify effective configuration:**

```bash
mms-http-send --show-config
```

Prints a JSON object showing every setting and exactly which source it came from (`file`, `env:VARNAME`, or `default`).

**Configuration — resolution order (highest priority first):**

| Priority | Source | How to set |
|----------|--------|-----------|
| 1 | `~/.config/mms-send` | Edit the file directly — see [Configuration File](#configuration-file) |
| 2 | `OPENCLAW_MMS_FROM` env var | `export OPENCLAW_MMS_FROM="+1XXXXXXXXXX"` in `.bashrc` |
| 3 | `PHONE_NUMBER` env var | `export PHONE_NUMBER="+1XXXXXXXXXX"` in `.bashrc` |
| 4 | Built-in default | Warns on stderr; edit the config file to fix |

**Config file keys and their env overrides:**

| Config key | Env override | Default | Description |
|-----------|-------------|---------|-------------|
| `FROM_NUMBER` | `OPENCLAW_MMS_FROM`, `PHONE_NUMBER` | `+17624346188` | Your SIM's E.164 number |
| `INTERFACE` | `OPENCLAW_MMS_INTERFACE` | `rmnet_data1` | Mobile data interface |
| `MMSC` | `OPENCLAW_MMS_MMSC` | `http://mmsc.mobile.att.net` | AT&T MMSC endpoint |
| `PROXY_IP` | `OPENCLAW_MMS_PROXY_IP` | `172.26.39.1` | WAP proxy (pre-resolved) |
| `PROXY_PORT` | `OPENCLAW_MMS_PROXY_PORT` | `80` | WAP proxy port |
| `MAX_BYTES` | `OPENCLAW_MMS_MAX_BYTES` | `900000` | Image resize threshold |

---

## Configuration File

`~/.config/mms-send` — INI-style, `KEY = value`, `#` comments. All fields optional; unset fields fall through to env vars then built-in defaults.

```ini
# ~/.config/mms-send
FROM_NUMBER = +17624346188   # your SIM's E.164 number
INTERFACE   = rmnet_data1    # ip link show | grep rmnet
MMSC        = http://mmsc.mobile.att.net
PROXY_IP    = 172.26.39.1    # pre-resolved AT&T WAP proxy
PROXY_PORT  = 80
MAX_BYTES   = 900000         # images auto-resize above this; audio/video do not
```

After editing, verify with:
```bash
mms-http-send --show-config
```

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

## Framework Compatibility

| File | Purpose | Who reads it |
|------|---------|-------------|
| `SKILL.md` | Skill definition, usage, dispatch summary | All agent frameworks (pi, OpenClaw, others) |
| `AGENTS.md` | Full dispatch guide (triggers, failure handling, timing) | OpenClaw; frameworks implementing the evolving AgentSkills standard |
| `tool.json` | Machine-readable schema (MCP 1.0 / OpenAI-functions format) | OpenClaw; MCP-compatible runtimes |
| `package.json` | Package manifest for `pi install git:...` | Pi |

`tool.json` and `AGENTS.md` are retained even when a specific runtime ignores them — both target the converging cross-framework standard and serve as the forward-compatible contract for this skill.

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
