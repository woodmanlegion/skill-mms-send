# skill-mms-send

Send MMS attachments (audio, image, video) from the command line on rooted Android/Termux. The primary path posts a WAP binary PDU directly to the AT&T MMSC — no screen, no Google Messages, no root required.

## Install

### Pi
```bash
pi install git:github.com/woodmanlegion/skill-mms-send
```
Scripts must be added to your `PATH` separately (see Setup in `SKILL.md`).

### OpenClaw
```bash
# Clone into your workspace skills directory
git clone https://github.com/woodmanlegion/skill-mms-send.git \
  ~/.openclaw/workspace/skills/mms-send
```

### Manual
```bash
git clone https://github.com/woodmanlegion/skill-mms-send.git
export PATH="$PATH:$(pwd)/skill-mms-send/bin"
```

## Quick test
```bash
mms-http-send --show-config       # verify configuration
mms-http-send +15558675309 /path/to/file.mp3
```

## Platform

**Android/Termux only.** Tested on rooted LineageOS with an AT&T SIM. The `mms-http-send` path requires no root; `mms-send-smart` (UI automation fallback) does.

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Agent skill definition — read by pi, OpenClaw, and compatible frameworks |
| `AGENTS.md` | Full agent dispatch guide — read by OpenClaw and AgentSkills-conforming frameworks |
| `tool.json` | MCP 1.0 / OpenAI-functions tool schema — for MCP-compatible runtimes |
| `package.json` | Pi package manifest (`pi install`) |
| `bin/` | CLI scripts |

## Dependencies
```bash
pkg install python curl ffmpeg termux-api
```

## Configuration
```bash
# Primary config source
~/.config/mms-send     # FROM_NUMBER, INTERFACE, MMSC, PROXY_IP, MAX_BYTES

# Env fallbacks (set in .bashrc)
export PHONE_NUMBER="+1XXXXXXXXXX"
export OPENCLAW_MMS_FROM="${OPENCLAW_MMS_FROM:-$PHONE_NUMBER}"
```
