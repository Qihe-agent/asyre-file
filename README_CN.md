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

## 为什么做这个？

我们在给客户部署 AI Agent 系统时，反复遇到同一个问题：

**AI 在服务器上跑得很好，但人类和 AI 之间缺一个沟通界面。**

Agent 生成了一份报告，你想看？SSH 进去 `cat report.md`。觉得第三段写得太正式？复制整段到聊天框，说"改口语化"，AI 改完再粘回去。客户想看产出？你得导出、邮件发送、等反馈、再改。

这个流程每天重复几十次。我们需要的不是一个更好的编辑器，而是一个**让人和 AI Agent 共享文件系统的浏览器界面**。

## 这是什么？

Asyre File 部署在和你的 AI Agent 同一台服务器上。它做两件事：

1. **给人类一个浏览器界面** — 查看、编辑、标注、分享服务器上的文件
2. **给 AI Agent 一套 REST API** — 读写同一批文件

两边操作同一个文件夹，实时同步。

```
 ┌──────────────────┐          ┌──────────────────────────────┐
 │  浏览器           │          │  你的服务器                    │
 │  (任何设备)       │◄────────►│                              │
 │                  │  HTTP    │  ┌────────────┐              │
 │  • 查看文件       │          │  │ Asyre File │◄──REST API──►│ AI Agent
 │  • 编辑保存       │          │  │ (port 8765)│              │ (Claude, GPT,
 │  • 标注反馈       │          │  └────────────┘              │  或自研系统)
 │  • 分享链接       │          │       │                      │
 └──────────────────┘          │       ▼                      │
                               │  ~/data/  (共享文件)           │
 ┌──────────────────┐          │                              │
 │  你的客户         │          └──────────────────────────────┘
 │  (通过分享链接)    │
 │  • 只读查看       │
 │  • 或可编辑       │
 └──────────────────┘
```

<p align="center">
<img src="docs/assets/dark-mode.png" width="80%" alt="Asyre File 主界面">
</p>

## 核心工作流

### 1. AI 写文件 → 你在浏览器里立即看到

Agent 通过 `PUT /api/v1/files/report.md` 创建文件。你刷新浏览器，文件树里就出现了。不需要 SSH，不需要 `scp`。

### 2. 你标注具体行 → 复制给你的 AI

这是我们解决的核心痛点。以前要反馈"第 12-18 行写得太正式"，你得：复制那几行 → 粘贴到聊天框 → 说"这是文件 xxx 的第 12 行" → AI 还不一定找得到。

现在：选中第 12-18 行 → 写"改口语化" → 点 **Copy**。复制出来的内容长这样：

```markdown
# Annotation: /home/ubuntu/data/report.md

Date: 2026-04-05T10:30:00Z
By: Asher

---
**[1] Lines 12-18**
```
The quarterly results demonstrate a significant...
```
Feedback: 太正式了，改口语化，加具体数字
```

**包含服务器上的完整路径**。粘贴到 Claude、GPT、或任何 AI 对话框，它能精确定位到哪个文件的哪几行要改什么。

![标注功能](docs/assets/annotate-lines.png)

![复制内容包含完整路径](docs/assets/annotate-copy.png)

### 3. 批量标注多个文件

客户发来一批文档要审核？**Cmd+Click** 多选，点 **Annotate**，写一段统一的批注意见，然后 **Copy** 或 **Save**。

Save 会把标注保存到服务器的 `annotations/` 目录，你的 AI Agent 可以通过 API 直接读取这些反馈并批量处理。

<p align="center">
<img src="docs/assets/annotate-batch-a.png" width="48%" alt="批量标注">
<img src="docs/assets/annotate-batch.png" width="48%" alt="批量标注结果">
</p>

### 4. 分享给客户 — 零门槛

右键文件 → **Share**，选择只读或可编辑，生成链接。客户打开链接就能看到文件，不需要注册账号，不需要安装任何东西。

文件夹也能整个分享。客户看到的是一个干净的编辑器界面，可以浏览文件树、预览 Markdown。如果你给了可编辑权限，他们还能直接改。

![分享链接](docs/assets/share-link.png)

## 我们在开发中遇到的问题

### 问题 1：单文件 vs 多文件架构

**纠结**：5000 行代码放一个文件？这不是反模式吗？

**决定**：保持单文件。因为目标用户（部署 AI 系统的工程师）需要的是 `scp server.py target:` 一个文件就部署完成。拆成 20 个文件会让部署复杂度暴增。HTML/CSS/JS 内联意味着不需要 npm、不需要 webpack、不需要任何构建工具。

**权衡**：维护一个 5000 行文件确实痛苦。我们的做法是用清晰的注释分区（`// ==================== SECTION ====================`）和严格的函数命名让代码可导航。

### 问题 2：Tailwind CSS 重置破坏 CodeMirror

**现象**：引入 Tailwind 后，CodeMirror 编辑器的所有样式都乱了 — 行高不对、光标位置错、选中高亮消失。

**原因**：Tailwind 的 CSS reset（`*, *::before, *::after { box-sizing: border-box }`）覆盖了 CodeMirror 依赖的 `content-box` 模型。

**解决**：在 Tailwind 之后加一段保护样式，把 CodeMirror 的所有元素强制回 `content-box`：
```css
.CodeMirror, .CodeMirror * { box-sizing: content-box !important; }
```

### 问题 3：亮色/暗色主题切换

**难点**：不只是改背景色。CodeMirror 有自己的主题系统，Mermaid 图表有自己的配色，highlight.js 代码高亮也有独立的主题。三套系统要同步切换。

**解决**：
- CodeMirror：不切换主题（保持 material-darker），用 CSS `[data-theme="light"] .cm-s-material-darker *` 覆盖所有颜色
- Mermaid：在渲染后用 JS 遍历 SVG 节点，根据当前主题动态换色 — 暗色用深底浅边（`#1e3a5f` + `#58a6ff`），亮色用浅底深边（`#dbeafe` + `#2563eb`）
- highlight.js：用 `[data-theme="light"] .hljs` 覆盖

### 问题 4：图片预览巨慢

**现象**：点击一张 2MB 的图片，要等 5-10 秒才显示。

**原因**：后端把图片 base64 编码后包在 JSON 里返回。2MB 图片 → 2.7MB base64 → 再包一层 JSON → 浏览器解析 JSON → 渲染 data URL。

**解决**：图片直接用 `<img src="/api/raw?path=xxx">` 加载，浏览器原生处理，秒开。同时加了 `Cache-Control: max-age=3600` 避免重复请求。

### 问题 5：HTTP 环境下剪贴板不可用

**现象**：Copy path、Copy link 功能在部署后全部失效。本地开发没问题。

**原因**：`navigator.clipboard.writeText()` 是 Secure Context API，只在 HTTPS 或 localhost 下可用。我们的服务器是 HTTP。

**解决**：写了一个 fallback 函数，优先尝试 `navigator.clipboard`，失败则用经典的 `document.execCommand('copy')` + 隐藏 textarea 方案：
```javascript
function copyText(text, msg) {
  if (navigator.clipboard && window.isSecureContext) {
    navigator.clipboard.writeText(text).then(...).catch(() => fallback(text));
  } else {
    fallback(text); // textarea + execCommand
  }
}
```

### 问题 6：权限系统的漏洞

**问题**：最初只在前端隐藏按钮（viewer 看不到删除按钮），但后端 API 没有做角色检查。一个 viewer 用 curl 直接调 `/api/delete` 就能删除任何文件。

**解决**：对所有 7 个写操作端点（save、move、delete、restore、trash-empty、share、new）逐一加上：
1. 角色检查（viewer 拒绝写操作，trash-empty 仅 admin）
2. 路径权限检查（`check_path_access`，editor 只能操作自己路径内的文件）
3. 操作日志（`log_activity`，记录谁在什么时候做了什么）

### 问题 7：Multipart 上传崩溃

**现象**：上传文件时服务器返回 500。

**原因**：`do_POST` 入口有一行 `body = json.loads(raw_body)`，在所有路由判断之前执行。上传的 multipart form data 不是 JSON，直接崩。

**解决**：改为检测 Content-Type 后再决定是否解析 JSON：
```python
if 'multipart' not in content_type and 'x-www-form-urlencoded' not in content_type:
    body = json.loads(raw_body)
```

### 问题 8：JavaScript 变量声明顺序

**现象**：页面加载报错 `Cannot access 'IS_READONLY' before initialization`。

**原因**：`const IS_READONLY` 在代码第 2906 行声明，但 CodeMirror 初始化（第 1225 行）已经引用了它。JavaScript 的 `const` 有暂时性死区（TDZ），声明前访问直接报错。

**解决**：把 `IS_READONLY` 和 `AUTH_USER` 的声明移到 `<script>` 最顶部，在任何引用之前。

## 快速开始

### 方式一：Git Clone（推荐）

```bash
git clone https://github.com/yzha0302/asyre-file.git
cd asyre-file
python3 server.py
```

访问 `http://localhost:8765` — 安装向导引导你创建管理员账号，可选生成 API Token。

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

### 首次安装

无论哪种方式，第一次访问会看到安装向导。填写：
- 管理员用户名和密码
- 站点名称
- 是否生成 API Token（给 AI Agent 用）

也支持命令行安装（SSH 无头部署）：`python3 server.py --setup`

## Agent API

### 认证

安装时勾选"Generate API token"，或之后用 CLI 生成：

```bash
python3 server.py --setup
```

Token 格式：`asf_<32位hex>`，存储为 SHA-256 哈希（即使 token 文件泄露也无法逆推原始 token）。

### 端点

```bash
# 健康检查
curl -H "Authorization: Bearer asf_xxx" http://server:8765/api/v1/status
# => {"ok": true, "version": "1.0.0", "name": "Asyre File"}

# 列出文件
curl -H "Authorization: Bearer asf_xxx" http://server:8765/api/v1/files
# => {"ok": true, "files": [{"path": "report.md", "size": 1234, "modified": 1712345678}], "count": 1}

# 读取文件
curl -H "Authorization: Bearer asf_xxx" http://server:8765/api/v1/files/report.md
# => {"ok": true, "path": "report.md", "content": "# Report\n...", "size": 1234}

# 创建/覆盖文件
curl -X PUT -H "Authorization: Bearer asf_xxx" \
  -H "Content-Type: application/json" \
  -d '{"content": "# 周报\n\n由 Agent 生成。"}' \
  http://server:8765/api/v1/files/reports/week-14.md

# 全文搜索
curl -H "Authorization: Bearer asf_xxx" http://server:8765/api/v1/search?q=keyword

# 移动/重命名
curl -X POST -H "Authorization: Bearer asf_xxx" \
  -H "Content-Type: application/json" \
  -d '{"to": "archive/old-report.md"}' \
  http://server:8765/api/v1/files/report.md/move

# 删除（移入回收站）
curl -X DELETE -H "Authorization: Bearer asf_xxx" \
  http://server:8765/api/v1/files/drafts/old.md
```

### 权限

每个 Token 可以设置细粒度权限：

| 权限 | 可执行操作 |
|------|----------|
| `read` | 列出文件、读取内容、搜索 |
| `write` | 创建、更新、移动/重命名 |
| `delete` | 删除（移入回收站） |

[完整 API 文档 →](docs/api.md)

## 功能详情

### 编辑器

- **20+ 语言**语法高亮 — Markdown、JavaScript、Python、HTML、JSON、YAML、Go、Rust、SQL 等
- **实时预览** — Markdown 渲染、Mermaid 图表、代码块高亮
- **暗色/亮色主题** — 一键切换，编辑器、预览区、图表全部联动

<p align="center">
<img src="docs/assets/dark-mode.png" width="48%" alt="暗色模式">
<img src="docs/assets/light-mode.png" width="48%" alt="亮色模式">
</p>

### 文件管理

- **文件树** — 按类型显示彩色 SVG 图标（类似 VS Code）
- **拖拽移动** — 直接拖文件到目标文件夹，有引用检测警告
- **上传** — 三种方式：侧边栏按钮、拖文件到页面、右键文件夹 "Upload here"
- **右键菜单** — Open、Rename、Copy path、Copy link、Download、Select、Delete
- **多选** — Cmd/Ctrl+Click 选择多个文件 → 批量 Annotate、Share、Copy、Delete
- **搜索** — 侧边栏搜索框，实时过滤
- **回收站** — 删除的文件进回收站，可恢复，管理员可清空

### 协作与分享

- **三种角色** — Admin（全部权限）、Editor（指定路径内读写）、Viewer（指定路径只读）
- **路径权限** — 给客户分配 `clients/acme/` 路径，他只能看到这个文件夹
- **分享链接** — 只读或可编辑，单文件或整个文件夹，无需注册即可访问
- **PDF & Word 导出** — 多种主题和页面尺寸，支持自定义署名和头像

<p align="center">
<img src="docs/assets/export-pdf.png" width="48%" alt="PDF 导出效果">
<img src="docs/assets/export-word.png" width="48%" alt="Word 导出效果">
</p>

### AI 协作工具

- **行级标注** — 选中任意行，写反馈，复制时自动包含服务器路径
- **批量文件标注** — 多选文件统一批注，Save 保存到服务器供 Agent 读取
- **活动日志** — 管理员面板可查看谁在什么时候做了什么操作
- **最近文件** — 侧边栏显示最近打开的文件
- **文件统计** — 工具栏显示字数、字符数、预计阅读时间

## 角色权限矩阵

| 操作 | Admin | Editor | Viewer |
|------|:-----:|:------:|:------:|
| 查看文件 | 全部 | 指定路径 | 指定路径 |
| 编辑 / 保存 | ✓ | 指定路径 | — |
| 新建 / 上传 | ✓ | 指定路径 | — |
| 移动 / 重命名 / 删除 | ✓ | 指定路径 | — |
| 拖拽移动文件 | ✓ | ✓ | — |
| 多选批量操作 | ✓ | ✓ | — |
| 分享链接 | ✓ | 指定路径 | — |
| 恢复回收站 | ✓ | ✓ | — |
| 清空回收站 | ✓ | — | — |
| 用户管理 | ✓ | — | — |
| API Token 管理 | ✓ | — | — |

## 配置

### 环境变量

```bash
ASF_SERVER_PORT=9000          # 端口
ASF_SERVER_HOST=0.0.0.0       # 绑定地址
ASF_WORKSPACE_PATH=/data      # 文件存储目录
ASF_AI_ENABLED=true           # 启用 AI 助手
ASF_AI_APIKEY=sk-...          # AI API Key
ASF_AUTH_SESSIONTIMEOUTHOURS=72  # 会话过期时间
```

### 配置文件

复制 `config.example.json` → `config.json`：

```json
{
  "server": {"host": "0.0.0.0", "port": 8765},
  "site": {"name": "My Workspace"},
  "workspace": {"path": "./data", "max_upload_mb": 50},
  "auth": {"session_timeout_hours": 72, "allow_registration": false},
  "ai": {"enabled": true, "provider": "anthropic", "api_key": "sk-..."},
  "export": {"pdf_enabled": true, "word_enabled": true}
}
```

优先级：环境变量 > config.json > 内置默认值。

详见 [docs/configuration.md](docs/configuration.md)。

## 技术架构

| 层 | 技术 | 为什么这么选 |
|----|------|------------|
| 后端 | Python 3 stdlib | 零依赖，任何服务器 `python3 server.py` 即用 |
| 编辑器 | CodeMirror 5 | CDN 加载，成熟稳定，20+ 语言模式 |
| 预览 | marked.js + highlight.js + Mermaid | CDN，覆盖主流 Markdown 场景 |
| UI | Tailwind CSS + 自定义组件 | CDN，无需构建工具 |
| 架构 | 单文件服务器 | 一个 `server.py`，所有 HTML/CSS/JS 内联 |

### 可选依赖

核心功能零依赖。以下是可选安装：

```bash
# PDF + Word 导出
pip install weasyprint python-docx

# AI 助手
pip install anthropic  # 或 openai
```

## 部署

支持多种部署方式：

- **直接运行** — `python3 server.py`
- **Docker** — `docker compose up -d`
- **PM2** — `pm2 start server.py --interpreter python3`
- **systemd** — 系统服务
- **Nginx 反向代理** — 生产环境推荐

详见 [docs/deployment.md](docs/deployment.md)。

## 文档

| 文档 | 内容 |
|------|------|
| [API 参考](docs/api.md) | Agent REST API 完整文档 |
| [配置说明](docs/configuration.md) | 所有配置项和环境变量 |
| [部署指南](docs/deployment.md) | Docker / Nginx / systemd / PM2 |

## 开源协议

[MIT](LICENSE) — 自由使用、修改、分发。

---

<div align="center">

由 [Asyre](https://github.com/yzha0302) 构建，为使用 AI 的人而生。

</div>
