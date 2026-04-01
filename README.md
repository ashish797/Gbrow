<div align="center">

# 🌐 Gbrow

### The Browser Your AI Agent Actually Needs

**Full-featured headless browser for [OpenClaw](https://github.com/openclaw/openclaw) agents.**

Navigate websites, read pages via accessibility tree, click elements by @ref, manage tabs, run JavaScript — all without expensive vision models.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Bun](https://img.shields.io/badge/Bun-1.3+-fbf0df)](https://bun.sh)
[![Playwright](https://img.shields.io/badge/Playwright-latest-2EAD33)](https://playwright.dev)
[![OpenClaw](https://img.shields.io/badge/OpenClaw-Skill-purple)](https://github.com/openclaw/openclaw)

[Setup](#-quick-setup) • [Commands](#-commands) • [How It Works](#-how-it-works) • [API](#-http-api) • [Contributing](#-contributing)

</div>

---

## The Problem

AI agents need to browse the web. But most browser tools for agents either:

1. **Take screenshots + send to vision models** → Slow, expensive ($0.01/page), breaks on API errors
2. **Parse raw HTML** → Fragile, messy, hard to understand
3. **Use simple text extraction** → Loses structure, can't interact

## The Solution

Gbrow uses Playwright's **accessibility tree** — the same structured data that screen readers use. It's:

- ⚡ **Fast** — < 100ms per page read
- 💰 **Free** — no vision model API calls
- 🎯 **Reliable** — no API keys to break
- 🖱️ **Interactive** — click by `@ref`, not fragile CSS selectors

```
┌─────────────┐     HTTP      ┌──────────────────┐
│  OpenClaw   │ ──────────▶  │  Gbrow Server    │
│  Agent      │              │  (Bun + Playwright)│
└─────────────┘              └────────┬─────────┘
                                      │
                                      ▼
                              ┌──────────────────┐
                              │ Accessibility     │
                              │ Tree (ariaSnapshot)│
                              └──────────────────┘
```

## 🚀 Quick Setup

### One-liner (recommended)

```bash
cd ~/.openclaw/workspace/skills
git clone https://github.com/ashish797/Gbrow.git
cd Gbrow && bash setup.sh
```

### Manual

```bash
# Prerequisites: Bun (https://bun.sh)
curl -fsSL https://bun.sh/install | bash

# Clone
git clone https://github.com/ashish797/Gbrow.git
cd Gbrow

# Install
bun install
npx playwright install chromium

# Start server
bun run src/server.ts
```

### Docker / VPS

Gbrow works in Docker containers. The setup script handles `--no-sandbox` automatically for Chromium.

## 📖 How It Works

### 1. Navigate to a page
```
goto https://news.ycombinator.com
→ Navigated to https://news.ycombinator.com (200)
```

### 2. Read the page (accessibility tree)
```
snapshot -i
→
@e1 [link] "Hacker News"
@e2 [link] "new"
@e3 [link] "past"
@e4 [link] "comments"
@e5 [link] "Ask HN: ..."
@e6 [link] "example.com"
@e7 [link] "2 hours ago"
@e8 [link] "42 comments"
```

### 3. Click by ref
```
click @e5
→ Clicked @e5 → now at https://news.ycombinator.com/item?id=12345
```

**No vision model. No API calls. Just structured text.**

## 🎮 Commands

### Navigation
| Command | Description |
|---------|-------------|
| `goto <url>` | Navigate to URL |
| `back` / `forward` / `reload` | History navigation |
| `url` | Print current URL |

### Reading
| Command | Description |
|---------|-------------|
| `snapshot [-i\|-c\|-d N\|-s sel]` | Accessibility tree with @refs |
| `text` | Cleaned page text |
| `html [selector]` | Raw HTML |
| `links` | All links as "text → href" |
| `forms` | Form fields as JSON |

### Interaction
| Command | Description |
|---------|-------------|
| `click <ref>` | Click element (`@e3`) |
| `fill <ref> <text>` | Fill input |
| `type <ref> <text>` | Type with keyboard |
| `select <ref> <value>` | Select dropdown |
| `press <key>` | Press key (Enter, Tab, etc.) |
| `scroll <direction>` | Scroll page |

### Inspection
| Command | Description |
|---------|-------------|
| `js <expr>` | Run JavaScript |
| `css <sel> <prop>` | Computed CSS value |
| `attrs <ref>` | Element attributes |
| `is <prop> <ref>` | State check (visible/hidden/enabled) |

### Tabs
| Command | Description |
|---------|-------------|
| `tabs` | List tabs |
| `tab N` | Switch tab |
| `newtab` / `closetab` | Open/close tabs |

### Visual
| Command | Description |
|---------|-------------|
| `screenshot` | Take screenshot |
| `pdf` | Save as PDF |
| `responsive <w> <h>` | Set viewport |

### Snapshot Flags
| Flag | Description |
|------|-------------|
| `-i` | Interactive elements only |
| `-c` | Compact (no empty nodes) |
| `-d N` | Limit depth |
| `-s <sel>` | Scope to selector |
| `-D` | Diff against previous |
| `-a` | Annotated screenshot |

## 🔌 HTTP API

All commands go through HTTP:

```bash
# Get credentials
PORT=$(python3 -c "import json; print(json.load(open('.gstack/browse.json'))['port'])")
TOKEN=$(python3 -c "import json; print(json.load(open('.gstack/browse.json'))['token'])")

# Navigate
curl -s -X POST "http://127.0.0.1:${PORT}/command" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"command":"goto","args":["https://example.com"]}'

# Snapshot
curl -s -X POST "http://127.0.0.1:${PORT}/command" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"command":"snapshot","args":["-i"]}'

# Click
curl -s -X POST "http://127.0.0.1:${PORT}/command" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"command":"click","args":["@e3"]}'
```

## 🔧 Configuration

| Environment Variable | Default | Description |
|---------------------|---------|-------------|
| `BROWSE_PORT` | random | Server port |
| `BROWSE_IDLE_TIMEOUT` | 1800000 | Auto-shutdown (ms) |
| `BROWSE_STATE_FILE` | `.gstack/browse.json` | State file path |

## 🏗️ Architecture

```
Gbrow/
├── src/
│   ├── server.ts          # HTTP server (Bun.serve)
│   ├── browser-manager.ts # Playwright lifecycle
│   ├── snapshot.ts        # Accessibility tree + @refs
│   ├── read-commands.ts   # Page reading
│   ├── write-commands.ts  # Page interaction
│   ├── meta-commands.ts   # Tabs, status, etc.
│   ├── commands.ts        # Command registry
│   ├── cli.ts             # CLI client
│   └── ...
├── setup.sh               # One-command install
├── SKILL.md               # OpenClaw skill definition
└── package.json
```

## 🤝 Comparison

| Feature | Gbrow | Browsectl | Puppeteer | Playwright raw |
|---------|-------|-----------|-----------|----------------|
| Accessibility tree | ✅ | ❌ | ❌ | ✅ (manual) |
| @ref click system | ✅ | ❌ | ❌ | ❌ |
| Tab management | ✅ | ❌ | Manual | Manual |
| HTTP API | ✅ | ✅ | ❌ | ❌ |
| OpenClaw integration | ✅ | ✅ | ❌ | ❌ |
| Vision model needed | ❌ | ✅ | ❌ | ❌ |
| Cookie import | ✅ | ❌ | ❌ | ❌ |
| JS execution | ✅ | ❌ | ✅ | ✅ |
| Setup complexity | One command | Medium | High | High |

## 🌟 Use Cases

- **AI Agent Web Research** — Browse and extract structured data
- **Automated Testing** — QA flows with ref-based assertions
- **Web Scraping** — Structured extraction via accessibility tree
- **Agent-to-Web Interaction** — Agents that can navigate, click, fill forms
- **Content Monitoring** — Watch pages for changes via diff mode

## 📦 Part of the OpenClaw Ecosystem

Gbrow is designed as a skill for [OpenClaw](https://github.com/openclaw/openclaw) — the open-source AI agent platform.

Other OpenClaw skills:
- [ClawHub](https://clawhub.ai) — Skill marketplace
- [browsectl](https://github.com/openclaw/openclaw) — Lightweight browser fallback

## 🛣️ Roadmap

- [ ] Cookie import from Chrome/Firefox
- [ ] Headed mode (connect to real browser)
- [ ] Watch mode (continuous page monitoring)
- [ ] MCP server integration
- [ ] A2A protocol support
- [ ] Mobile viewport presets

## 📄 License

MIT — Built on [gstack](https://github.com/garrytan/gstack) by Gary Tan.

## 💬 Community

- [OpenClaw Discord](https://discord.com/invite/clawd)
- [OpenClaw Docs](https://docs.openclaw.ai)
- [GitHub Issues](https://github.com/ashish797/Gbrow/issues)

---

<div align="center">

**⭐ Star this repo if Gbrow helps your agent browse the web!**

Made with ⚡ by [devD](https://github.com/ashish797) for the OpenClaw community

</div>
