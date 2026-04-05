<div align="center">

# Asyre File

[![Python 3.8+](https://img.shields.io/badge/python-3.8+-3776ab?style=flat-square&logo=python&logoColor=white)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/license-MIT-22c55e?style=flat-square)](LICENSE)
[![GitHub release](https://img.shields.io/github/v/release/yzha0302/asyre-file?style=flat-square&color=7c3aed)](https://github.com/yzha0302/asyre-file/releases)
[![Docker](https://img.shields.io/badge/docker-ready-2496ed?style=flat-square&logo=docker&logoColor=white)](Dockerfile)

**连接服务器上的 AI 与浏览器中的人类 — 基于浏览器的文件工作台**

[快速开始](#快速开始) · [Agent API](#agent-api) · [English](README.md) · [更新日志](CHANGELOG.md)

</div>

---

## 解决什么问题？

你在远程服务器上跑 AI Agent（Claude、GPT、自研系统）。它们生成报告、写文档、产出代码。但是：

- **看不到产出** — 要 SSH 进去 `cat` 才能看
- **无法精确反馈** — 想说"第 42 行太正式了"，只能整段复制粘贴到聊天框
- **客户看不到** — 他们没有服务器权限，你得把文件邮件来回发

## 解决方案

Asyre File 部署在和 AI Agent 同一台服务器上。人类通过浏览器查看/编辑/标注/分享，Agent 通过 REST API 读写 — 共享同一个文件系统。

```
 ┌──────────────────┐          ┌──────────────────────────────┐
 │  浏览器           │          │  你的服务器                    │
 │  (任何设备)       │◄────────►│                              │
 │                  │  HTTP    │  ┌────────────┐              │
 │  • 查看文件       │          │  │ Asyre File │◄──REST API──►│ AI Agent
 │  • 编辑保存       │          │  │ (port 8765)│              │
 │  • 标注反馈       │          │  └────────────┘              │
 │  • 分享链接       │          │       │                      │
 └──────────────────┘          │       ▼                      │
                               │  ~/data/  (共享文件)           │
 ┌──────────────────┐          │                              │
 │  你的客户         │          └──────────────────────────────┘
 │  (分享链接)       │
 │  • 只读查看       │
 │  • 或可编辑       │
 └──────────────────┘
```

<p align="center">
<img src="docs/assets/dark-mode.png" width="80%" alt="Asyre File">
</p>

## 工作流程

### 1. AI 写文件 → 你在浏览器里看到

Agent 通过 REST API 创建/更新文件，打开浏览器就能看到。

### 2. 你标注具体行 → 复制给 AI

选中行，写反馈（如"太正式了，改口语化"），点 **Copy** — 标注包含**服务器完整路径**，任何 AI 都能定位到文件。

![标注功能](docs/assets/annotate-lines.png)

**复制内容包含完整路径：**

![标注复制](docs/assets/annotate-copy.png)

粘贴到 Claude、GPT 或任何 AI 对话中，它知道该改哪个文件的哪几行。

### 3. 批量标注多个文件

**Cmd+Click** 多选文件，点 **Annotate**，写反馈，然后 **Copy** 或 **Save**。

<p align="center">
<img src="docs/assets/annotate-batch-a.png" width="48%" alt="批量标注">
<img src="docs/assets/annotate-batch.png" width="48%" alt="批量标注结果">
</p>

### 4. 分享给任何人 — 无需注册

右键文件 → **Copy link**，或分享整个文件夹。对方看到干净的只读（或可编辑）视图。

![分享链接](docs/assets/share-link.png)

## 快速开始

### 方式一：Git Clone

```bash
git clone https://github.com/yzha0302/asyre-file.git
cd asyre-file
python3 server.py
```

访问 `http://localhost:8765` — 安装向导会引导你创建管理员账号。

### 方式二：Docker

```bash
git clone https://github.com/yzha0302/asyre-file.git
cd asyre-file
docker compose up -d
```

### 方式三：一键安装

```bash
curl -fsSL https://raw.githubusercontent.com/yzha0302/asyre-file/main/install.sh | bash
```

## Agent API

安装时生成 API Token，Agent 即可通过 REST API 操作文件：

```bash
# 写入文件
curl -X PUT -H "Authorization: Bearer asf_your_token" \
  -H "Content-Type: application/json" \
  -d '{"content": "# 周报\n\n由 Agent 生成。"}' \
  http://your-server:8765/api/v1/files/reports/week-14.md

# 读取文件
curl -H "Authorization: Bearer asf_your_token" \
  http://your-server:8765/api/v1/files/reports/week-14.md

# 列出所有文件
curl -H "Authorization: Bearer asf_your_token" \
  http://your-server:8765/api/v1/files

# 搜索
curl -H "Authorization: Bearer asf_your_token" \
  http://your-server:8765/api/v1/search?q=keyword

# 删除（移入回收站）
curl -X DELETE -H "Authorization: Bearer asf_your_token" \
  http://your-server:8765/api/v1/files/drafts/old.md
```

Token 支持细粒度权限：`read`、`write`、`delete`。

[完整 API 文档 →](docs/api.md)

## 功能特性

### 编辑器
- 实时 Markdown 编辑 + **20+ 语言**语法高亮
- 实时预览，支持 **Mermaid 图表**、代码高亮、数学公式
- **暗色/亮色主题** — 编辑器、预览、图表全部适配

<p align="center">
<img src="docs/assets/dark-mode.png" width="48%" alt="暗色模式">
<img src="docs/assets/light-mode.png" width="48%" alt="亮色模式">
</p>

### 文件管理
- **文件树** + 彩色文件类型图标
- **拖拽移动** 文件到不同文件夹
- **上传** — 按钮 / 拖拽到页面 / 右键 "Upload here"
- **右键菜单** — 打开、重命名、复制路径、复制链接、下载、选择、删除
- **多选** Cmd/Ctrl+Click → 批量分享、标注、复制、删除
- **搜索** + **回收站**（管理员可清空）

### 协作
- **多用户认证** — Admin / Editor / Viewer 三种角色
- **路径权限** — 每个用户限定在特定文件夹
- **分享链接** — 只读或可编辑，单文件或整个文件夹
- **PDF & Word 导出** — 自定义署名和主题

<p align="center">
<img src="docs/assets/export-pdf.png" width="48%" alt="PDF 导出">
<img src="docs/assets/export-word.png" width="48%" alt="Word 导出">
</p>

### AI 协作
- **行级标注** — 选中行写反馈，复制包含服务器完整路径
- **批量文件标注** — 多选文件一次性批注
- **REST API** — Token 认证，Agent 直接读写文件
- **活动日志** — 追踪所有文件变更（管理员面板）
- **标注持久化** — 保存到服务器，Agent 可通过 API 读取

## 角色权限

| 操作 | Admin | Editor | Viewer |
|------|:-----:|:------:|:------:|
| 查看文件 | 全部 | 指定路径 | 指定路径 |
| 编辑/保存 | ✓ | 指定路径 | — |
| 新建/上传 | ✓ | 指定路径 | — |
| 移动/重命名/删除 | ✓ | 指定路径 | — |
| 分享链接 | ✓ | 指定路径 | — |
| 清空回收站 | ✓ | — | — |
| 用户管理 | ✓ | — | — |

## 配置

```bash
# 环境变量（覆盖 config.json）
ASF_SERVER_PORT=9000
ASF_WORKSPACE_PATH=/path/to/files
ASF_AI_ENABLED=true
ASF_AI_APIKEY=sk-...
```

或复制 `config.example.json` → `config.json`。详见 [docs/configuration.md](docs/configuration.md)。

## 技术栈

| 层 | 技术 | 说明 |
|----|------|------|
| 后端 | Python 3 stdlib | 零必需依赖 |
| 编辑器 | CodeMirror 5 | CDN，无需构建 |
| 预览 | marked.js + highlight.js + Mermaid | CDN |
| UI | Tailwind CSS + 自定义组件 | CDN |
| 架构 | 单文件服务器 | HTML/CSS/JS 全部内联在 `server.py` |

## 导出（可选）

PDF 和 Word 导出需要额外依赖：

```bash
pip install weasyprint python-docx
```

不安装也完全不影响核心功能。

## 文档

- [API 参考](docs/api.md) — Agent REST API
- [配置说明](docs/configuration.md) — 所有配置项
- [部署指南](docs/deployment.md) — Docker / Nginx / systemd / PM2

## 开源协议

[MIT](LICENSE)

---

<div align="center">

由 [Asyre](https://github.com/yzha0302) 构建，为使用 AI 的人而生。

</div>
