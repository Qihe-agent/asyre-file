<div align="center">

# Asyre File

[![Python 3.8+](https://img.shields.io/badge/python-3.8+-3776ab?style=flat-square&logo=python&logoColor=white)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/license-MIT-22c55e?style=flat-square)](LICENSE)
[![GitHub release](https://img.shields.io/github/v/release/yzha0302/asyre-file?style=flat-square&color=7c3aed)](https://github.com/yzha0302/asyre-file/releases)
[![Docker](https://img.shields.io/badge/docker-ready-2496ed?style=flat-square&logo=docker&logoColor=white)](Dockerfile)

**A browser-based file workspace that bridges AI agents on your server with humans on any device.**

[Quick Start](#quick-start) · [Agent API](#agent-api) · [中文文档](README_CN.md) · [Changelog](CHANGELOG.md)

</div>

---

## Why We Built This

We kept hitting the same problem when deploying AI agent systems for clients:

**AI runs great on the server, but there's no interface between the AI and the humans.**

The agent generates a report — want to read it? SSH in, `cat report.md`. Think paragraph 3 is too formal? Copy the whole thing into a chat, say "line 12 is too stiff," hope the AI finds it. Client wants to see the output? Export, email, wait for feedback, edit, repeat.

This happens dozens of times a day. We didn't need a better editor — we needed a **browser interface that lets humans and AI agents share a file system**.

## What It Is

Asyre File runs on the same server as your AI agents. It does two things:

1. **Gives humans a browser UI** — view, edit, annotate, and share files on the server
2. **Gives AI agents a REST API** — read and write the same files

Both sides work on the same folder, in real-time.

```
 ┌──────────────────┐          ┌──────────────────────────────┐
 │  Browser          │          │  Your Server                 │
 │  (any device)     │◄────────►│                              │
 │                  │  HTTP    │  ┌────────────┐              │
 │  • View files    │          │  │ Asyre File │◄──REST API──►│ AI Agent
 │  • Edit & save   │          │  │ (port 8765)│              │ (Claude, GPT,
 │  • Annotate      │          │  └────────────┘              │  or custom)
 │  • Share links   │          │       │                      │
 └──────────────────┘          │       ▼                      │
                               │  ~/data/  (shared files)     │
 ┌──────────────────┐          │                              │
 │  Your Client     │          └──────────────────────────────┘
 │  (via share link)│
 │  • Read-only     │
 │  • Or editable   │
 └──────────────────┘
```

<p align="center">
<img src="docs/assets/dark-mode.png" width="80%" alt="Asyre File">
</p>

## Core Workflow

### 1. AI writes a file → You see it in your browser

Your agent calls `PUT /api/v1/files/report.md`. Refresh the browser — it's in the file tree. No SSH, no `scp`.

### 2. You annotate specific lines → Copy to your AI

This is the core pain point we solved. Before, to say "lines 12-18 are too formal," you had to: copy those lines → paste into chat → say "this is from file xxx line 12" → hope the AI finds it.

Now: select lines 12-18 → write "make it conversational" → click **Copy**. What gets copied:

```markdown
# Annotation: /home/ubuntu/data/report.md

Date: 2026-04-05T10:30:00Z
By: Asher

---
**[1] Lines 12-18**
```
The quarterly results demonstrate a significant...
```
Feedback: Too formal. Make it conversational, add specific numbers.
```

**Includes the full server path.** Paste into Claude, GPT, or any AI — it knows exactly which file and which lines to fix.

![Annotate lines](docs/assets/annotate-lines.png)

![Copy includes full paths](docs/assets/annotate-copy.png)

### 3. Batch annotate multiple files

Reviewing a batch of docs for a client? **Cmd+Click** to select multiple files, hit **Annotate**, write your feedback, then **Copy** or **Save**.

Save persists annotations to the server's `annotations/` directory — your AI agent can read them via API and process feedback automatically.

<p align="center">
<img src="docs/assets/annotate-batch-a.png" width="48%" alt="Batch annotate">
<img src="docs/assets/annotate-batch.png" width="48%" alt="Batch annotate result">
</p>

### 4. Share with anyone — zero friction

Right-click → **Share**, choose read-only or editable, get a link. Your client opens it and sees a clean editor view — no account needed, no install required.

Works for entire folders too. If you grant edit permission, they can modify files directly.

![Share link](docs/assets/share-link.png)

## Problems We Solved Along the Way

### Tailwind CSS reset breaking CodeMirror

Tailwind's `box-sizing: border-box` reset destroyed CodeMirror's layout — wrong line heights, broken cursor, missing selection highlights. Fix: force CodeMirror back to `content-box`:
```css
.CodeMirror, .CodeMirror * { box-sizing: content-box !important; }
```

### Dark/light theme with three rendering engines

Switching themes means syncing CodeMirror, Mermaid, and highlight.js — each has its own color system. We kept CodeMirror on `material-darker` and overrode colors with CSS specificity. Mermaid nodes get recolored via JS after render. highlight.js uses `[data-theme="light"] .hljs` overrides.

### Image preview was incredibly slow

A 2MB image took 5-10 seconds. Why? The backend base64-encoded it, wrapped it in JSON, sent it to the browser, which parsed JSON, then rendered a data URL. Fix: serve images directly via `<img src="/api/raw?path=xxx">` with `Cache-Control` headers. Instant.

### Clipboard API requires HTTPS

`navigator.clipboard.writeText()` silently fails on HTTP. Every "Copy path" and "Copy link" feature broke in production. Fix: fallback to `document.execCommand('copy')` with a hidden textarea when `window.isSecureContext` is false.

### Permission enforcement was frontend-only

We hid buttons from viewers in the UI but forgot to check roles on the backend. A viewer with `curl` could delete any file. Fix: added role + path checks to all 7 write endpoints, plus activity logging.

### Multipart upload crashed the server

`body = json.loads(raw_body)` ran before routing — when `raw_body` was multipart form data, `json.loads` threw an exception. Fix: check `Content-Type` before attempting JSON parse.

### JavaScript `const` temporal dead zone

`const IS_READONLY` was declared at line 2906 but referenced at line 1225 (CodeMirror init). JavaScript's TDZ means accessing a `const` before its declaration throws `ReferenceError`. Fix: move declarations to the top of `<script>`.

## Quick Start

### Option 1: Git Clone (recommended)

```bash
git clone https://github.com/yzha0302/asyre-file.git
cd asyre-file
python3 server.py
```

Visit `http://localhost:8765` — the setup wizard creates your admin account and optionally generates an API token.

### Option 2: Docker

```bash
git clone https://github.com/yzha0302/asyre-file.git
cd asyre-file
docker compose up -d
```

### Option 3: One-Line Install

```bash
curl -fsSL https://raw.githubusercontent.com/yzha0302/asyre-file/main/install.sh | bash
```

### First-Run Setup

On first visit, the setup wizard asks for:
- Admin username and password
- Site name
- Whether to generate an API token for agents

For headless servers: `python3 server.py --setup`

## Agent API

### Authentication

Tokens use format `asf_<32hex>`, stored as SHA-256 hashes (even if the token file leaks, the original token can't be recovered).

### Endpoints

```bash
# Health check
curl -H "Authorization: Bearer asf_xxx" http://server:8765/api/v1/status

# List files
curl -H "Authorization: Bearer asf_xxx" http://server:8765/api/v1/files

# Read file
curl -H "Authorization: Bearer asf_xxx" http://server:8765/api/v1/files/report.md

# Write file
curl -X PUT -H "Authorization: Bearer asf_xxx" \
  -d '{"content": "# Report\nGenerated by agent."}' \
  http://server:8765/api/v1/files/report.md

# Search
curl -H "Authorization: Bearer asf_xxx" http://server:8765/api/v1/search?q=keyword

# Move/rename
curl -X POST -H "Authorization: Bearer asf_xxx" \
  -d '{"to": "archive/old.md"}' \
  http://server:8765/api/v1/files/report.md/move

# Delete (trash)
curl -X DELETE -H "Authorization: Bearer asf_xxx" \
  http://server:8765/api/v1/files/old.md
```

### Token Permissions

| Permission | Operations |
|------------|-----------|
| `read` | List files, read content, search |
| `write` | Create, update, move/rename |
| `delete` | Move to trash |

[Full API Reference →](docs/api.md)

## Features

### Editor
- **20+ languages** with syntax highlighting
- **Live preview** with Mermaid diagrams, code highlighting
- **Dark & light themes** — editor, preview, and diagrams all sync

<p align="center">
<img src="docs/assets/dark-mode.png" width="48%" alt="Dark mode">
<img src="docs/assets/light-mode.png" width="48%" alt="Light mode">
</p>

### File Management
- **File tree** with colored SVG icons by file type
- **Drag-and-drop** between folders with reference detection warnings
- **Upload** via button, drag-to-page, or right-click "Upload here"
- **Right-click menu** — Open, Rename, Copy path, Copy link, Download, Select, Delete
- **Multi-select** with Cmd/Ctrl+Click → batch Annotate, Share, Copy, Delete
- **Search** and **trash with restore**

### Collaboration
- **Three roles** — Admin (everything), Editor (scoped read/write), Viewer (scoped read-only)
- **Path permissions** — assign a client to `clients/acme/`, they only see that folder
- **Share links** — read-only or editable, files or folders, no registration needed
- **PDF & Word export** with custom signatures and themes

<p align="center">
<img src="docs/assets/export-pdf.png" width="48%" alt="PDF export">
<img src="docs/assets/export-word.png" width="48%" alt="Word export">
</p>

### AI Collaboration Tools
- **Line-level annotations** — select lines, write feedback, copy with full server paths
- **Batch file annotations** — annotate multiple files, save to server for agent consumption
- **Activity log** — track all changes in admin panel
- **Recent files** and **file statistics** (word count, reading time)

## Roles & Permissions

| Action | Admin | Editor | Viewer |
|--------|:-----:|:------:|:------:|
| View files | All | Scoped | Scoped |
| Edit / Save | ✓ | Scoped | — |
| Create / Upload | ✓ | Scoped | — |
| Move / Rename / Delete | ✓ | Scoped | — |
| Drag-and-drop | ✓ | ✓ | — |
| Multi-select operations | ✓ | ✓ | — |
| Share links | ✓ | Scoped | — |
| Restore from trash | ✓ | ✓ | — |
| Empty trash | ✓ | — | — |
| User management | ✓ | — | — |

## Configuration

### Environment Variables

```bash
ASF_SERVER_PORT=9000
ASF_WORKSPACE_PATH=/data
ASF_AI_ENABLED=true
ASF_AI_APIKEY=sk-...
```

### Config File

Copy `config.example.json` to `config.json`. Priority: env vars > config.json > defaults.

See [docs/configuration.md](docs/configuration.md).

## Architecture

| Layer | Technology | Why |
|-------|-----------|-----|
| Backend | Python 3 stdlib | Zero deps, `python3 server.py` on any machine |
| Editor | CodeMirror 5 | CDN, mature, 20+ language modes |
| Preview | marked.js + highlight.js + Mermaid | CDN, covers mainstream Markdown |
| UI | Tailwind CSS + custom components | CDN, no build tools |
| Architecture | Single-file server | One `server.py`, all HTML/CSS/JS inline |

### Optional Dependencies

```bash
pip install weasyprint python-docx  # PDF + Word export
pip install anthropic               # AI assistant
```

## Documentation

| Doc | Content |
|-----|---------|
| [API Reference](docs/api.md) | Full Agent REST API |
| [Configuration](docs/configuration.md) | All config options |
| [Deployment](docs/deployment.md) | Docker / Nginx / systemd / PM2 |

## License

[MIT](LICENSE) — free to use, modify, distribute.

---

<div align="center">

Built by [Asyre](https://github.com/yzha0302) for humans who work with AI.

</div>
