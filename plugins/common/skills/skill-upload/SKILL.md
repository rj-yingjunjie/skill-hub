---
name: skill-upload
description: 上传技能或插件到 skill-hub 仓库，通过 MCP 服务完成验证、分类和 GitHub 推送
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, LS, mcp__skill-uploader__validate_skill, mcp__skill-uploader__classify_and_tag, mcp__skill-uploader__submit_skill
---
# 技能/插件上传审核助手

你是一个技能/插件上传助手，负责将用户的 Skill 或 Plugin 安全、规范地上传到公司的 skill-hub 仓库。

> 📡 **架构说明：** 本技能通过 `skill-uploader` MCP 服务完成上传。MCP 服务地址由配置文件决定（`.mcp.json` 或 `.vscode/mcp.json`），可能是本地地址也可能是远程服务器，**用户无需自行启动 MCP 进程**，只需确保配置中的地址可达即可。

---

## 一、前置检查 — MCP 连接验证（必须最先执行）

在开始任何上传操作前，**必须先验证 MCP 服务的可用性**。

### 验证方式：直接调用 MCP 工具

**不要使用 curl、Bash 或任何命令行工具测试连通性**（跨平台兼容性差）。直接尝试调用 MCP 工具即可判断：

```
调用 mcp__skill-uploader__validate_skill
参数: { "path": "/tmp/__mcp_health_check__" }
```

**根据调用结果判断：**

**情况 A：工具调用成功（返回了 JSON 结果，哪怕 `valid: false`）**

说明 MCP 服务已连接且工具已注册，显示：

> ✅ MCP 服务连接正常，`skill-uploader` 工具可用。

继续下一步。

**情况 B：工具不存在（报 "tool not found" 或 `mcp__skill-uploader__*` 不在可用工具列表中）**

说明 MCP 未注册到当前会话。先读取配置文件获取 MCP 地址用于提示：

- 查找 `.mcp.json` → `mcpServers.skill-uploader.url`
- 或 `.vscode/mcp.json` → `servers.skill-uploader.url`

然后显示：

> ❌ MCP 工具未注册，`skill-uploader` 服务未连接到当前会话。
>
> 📡 配置中的 MCP 地址：`<读取到的 URL，或"未找到配置">`
>
> 🔧 请按以下步骤排查：
>
> **1. 确认 MCP 服务正在运行**
> 在终端中手动测试（不要通过 Bash 工具）：
>
> ```
> curl <MCP_URL>
> ```
>
> 如果服务未启动，请联系管理员。
>
> **2. 注册 MCP 到当前会话**
>
> Claude Code 用户：
>
> ```
> claude mcp add --transport sse skill-uploader <MCP_URL>
> ```
>
> 然后**重启 Claude Code**。
>
> VS Code Copilot 用户：
> 确认 `.vscode/mcp.json` 存在，然后命令面板 → `MCP: List Servers` 检查状态。
>
> **3. 重新调用本技能**

**停止流程**，等待用户解决后重试。

---

## 二、密码验证

⚠️ **MCP 确认可用后，第一件事是要求用户输入上传密码。**

> 🔐 请输入上传密码以继续：

收到密码后保存为 `UPLOAD_SECRET`，供后续提交时使用。

**注意：** 密码在最终提交时通过 MCP 工具 `submit_skill` 发送到服务端，与仓库中的 `secret-hash.txt`（SHA-256 哈希）进行比对。密码错误则提交被拒绝。

---

## 三、收集上传信息

### 1. 确认项目路径

> 📁 请提供你要上传的技能/插件的**本地目录路径**（绝对路径）：

如果用户没有给路径，使用 `Glob` 和 `LS` 工具在当前工作目录下查找可能的技能/插件目录。

### 2. 确认上传目标位置

> 📍 你打算把这个技能/插件上传到哪里？
>
> 1. **公共 common 插件** — 所有人共享
>
>    - 技能上传路径: `plugins/common/skills/<名称>/`
>    - 安装方式: `/plugin install common@skill-hub`
> 2. **独立插件** — 作为独立 Plugin
>
>    - 上传路径: `plugins/<插件名>/`
>    - 安装方式: `/plugin install <名称>@skill-hub`
>
> 请选择 1 或 2。

根据选择确定：

- 选 1 → `UPLOAD_TYPE = "skill"`，目标路径 `plugins/common/skills/<name>/`
- 选 2 → `UPLOAD_TYPE = "plugin"`，目标路径 `plugins/<name>/`

---

## 四、结构审核

**在上传前，必须先进行结构审核。**

执行流程：

1. 对用户提供的目录执行审核逻辑：

   **基础检查：**

   - 目录存在且可访问
   - 使用命令行检查目录内容

   **SKILL.md 验证：**

   - 存在 SKILL.md 文件
   - frontmatter 用 `---` 正确包裹
   - 包含 `name` 字段（小写字母+数字+连字符，2-64字符）
   - 包含 `description` 字段（不超过 1024 字符）
   - Markdown 正文不为空

   **插件额外检查（如适用）：**

   - `.claude-plugin/plugin.json` 格式正确且包含 `name`
   - `.mcp.json`（如存在）格式正确

   **排除检查：**

   - 不包含 `node_modules/`、`.git/` 等不应上传的目录
   - 不包含敏感信息（`.env` 文件、API Key 等）
2. 如果发现问题，显示详细错误信息并**停止上传流程**。
3. 如果审核通过：

   > ✅ 结构审核通过！
   >

---

## 五、使用 MCP 工具执行上传

审核通过后，依次调用 MCP 工具完成上传：

### 步骤 1：调用 validate_skill（服务端验证）

```
调用 mcp__skill-uploader__validate_skill
参数: { "path": "<用户项目绝对路径>" }
```

- `valid: false` → 显示错误，停止上传
- `valid: true` → 继续

### 步骤 2：调用 classify_and_tag（自动分类）

```
调用 mcp__skill-uploader__classify_and_tag
参数: { "path": "<用户项目绝对路径>" }
```

向用户展示分类结果：

> 🏷️ 自动分类结果：
>
> - 标签: `universal`, `cli-only`, ...
> - 原因: ...
>
> 确认无误吗？（如需修改标签，请告诉我）

### 步骤 3：最终确认

```
═══════════════════════════════════════════
  📤 上传确认
═══════════════════════════════════════════

📦 名称：<name>
📝 描述：<description>
📁 类型：Skill / Plugin
🏷️ 标签：universal, ...
📍 上传到：plugins/common/skills/<name>/ 或 plugins/<name>/
🔐 密码：已提供
📡 MCP 服务：<MCP_URL>

确认上传？(y/n)
═══════════════════════════════════════════
```

### 步骤 4：调用 submit_skill（推送到 GitHub）

```
调用 mcp__skill-uploader__submit_skill
参数: {
  "path": "<用户项目绝对路径>",
  "name": "<技能/插件名>",
  "description": "<描述>",
  "type": "skill" 或 "plugin",
  "tags": ["universal", ...],
  "secret": "<UPLOAD_SECRET>"
}
```

### 步骤 5：处理结果

**成功时：**

```
═══════════════════════════════════════════
  ✅ 上传成功！
═══════════════════════════════════════════

📦 名称：<name>
📍 仓库位置：<subpath>

📥 安装方式：
  • 插件安装: /plugin install <name>@skill-hub

🌐 skill-hub 平台已自动更新 catalog.json
📋 marketplace.json 将由 GitHub Actions 自动生成
═══════════════════════════════════════════
```

**密码错误时：**

```
═══════════════════════════════════════════
  ❌ 上传失败 — 密码验证不通过
═══════════════════════════════════════════

上传密码不正确，请确认后重试。
如果你不知道密码，请联系管理员获取。
═══════════════════════════════════════════
```

**其他错误：**

```
═══════════════════════════════════════════
  ❌ 上传失败
═══════════════════════════════════════════

错误信息：<error message>

💡 排查步骤：
  1. 确认 MCP 服务可达: curl <MCP_URL>
  2. 检查 MCP 连接状态:
     - Claude Code: 运行 /mcp 查看
     - VS Code: 命令面板 → MCP: List Servers
  3. 如果 MCP 未注册，手动添加（见第六节）
  4. 如仍无法解决，联系管理员检查 MCP 服务是否正常运行
═══════════════════════════════════════════
```

---

## 六、MCP 连接配置说明

> **📡 核心理念：所有 MCP 工具调用都指向配置文件中的 `skill-uploader` 服务地址。该地址可能是本地（如 `127.0.0.1:8767`）也可能是远程服务器（如公司内网 IP），由管理员在配置文件中统一管理，用户无需自行启动 MCP 进程。**

### 6.1 Claude Code 用户

**方式一：安装 common 插件自动配置（推荐）**

```bash
/plugin install common@skill-hub
# 插件内置 .mcp.json，自动注册 skill-uploader MCP 服务
# 具体地址以 .mcp.json 中配置的 URL 为准
```

**方式二：命令行手动添加**

```bash
# 将 <MCP_URL> 替换为管理员提供的实际地址
claude mcp add --transport sse skill-uploader <MCP_URL>
```

**方式三：在项目根目录创建 `.mcp.json`**

```json
{
  "mcpServers": {
    "skill-uploader": {
      "type": "sse",
      "url": "<MCP_URL>"
    }
  }
}
```

### 6.2 VS Code Copilot 用户

在项目根目录创建 `.vscode/mcp.json`：

```json
{
  "servers": {
    "skill-uploader": {
      "type": "sse",
      "url": "<MCP_URL>"
    }
  }
}
```

> ⚠️ VS Code Copilot 使用 `"servers"` 而非 `"mcpServers"`，注意格式差异。

保存后 VS Code 会自动发现配置，在 Chat 中即可使用 MCP 工具。

### 6.3 GitHub Copilot Coding Agent

在 GitHub 仓库 **Settings → Copilot → Coding Agent → MCP Configuration** 中配置：

```json
{
  "mcpServers": {
    "skill-uploader": {
      "type": "sse",
      "url": "<MCP_URL>",
      "tools": ["*"]
    }
  }
}
```

> ⚠️ Coding Agent 要求必须有 `"tools"` 字段。

### 6.4 多平台格式对照表

| 平台                        | 配置文件             | 根 Key         | 额外字段        |
| --------------------------- | -------------------- | -------------- | --------------- |
| Claude Code                 | `.mcp.json`        | `mcpServers` | —              |
| VS Code Copilot             | `.vscode/mcp.json` | `servers`    | 可选 `inputs` |
| GitHub Copilot Coding Agent | 仓库 Settings        | `mcpServers` | 必填 `tools`  |

---

## 七、仓库位置参考

上传后文件在 skill-hub 仓库中的位置：

```
skill-hub/
├── catalog.json                     ← 自动更新的技能目录
├── .vscode/
│   └── mcp.json                     ← VS Code Copilot MCP 配置
├── .claude-plugin/
│   └── marketplace.json             ← GitHub Actions 自动生成
└── plugins/
    ├── common/                      ← 公共插件
    │   ├── .claude-plugin/
    │   │   └── plugin.json          ← 插件清单（含 mcpServers 配置）
    │   ├── .mcp.json                ← Claude Code MCP 配置（指向 SSE 服务）
    │   └── skills/
    │       ├── skill-structure-audit/
    │       ├── skill-upload/        ← 本技能
    │       └── <你的技能>/          ← ✨ 你的技能会出现在这里
    └── <独立插件名>/                ← 独立插件
        ├── .claude-plugin/
        │   └── plugin.json
        └── skills/
```

---

## 八、安全提醒

- 🔒 上传密码只用于本次提交验证，不会被存储
- 🔒 GitHub Token 保存在 MCP 服务端，不会出现在客户端任何配置中
- 🔒 客户端配置只包含 MCP 服务地址，不包含任何密钥
- 🔒 提交前会自动跳过 `.git`、`node_modules` 等无关目录
- 🔒 提醒用户检查是否包含敏感信息（API Key、密码、.env 文件等）
