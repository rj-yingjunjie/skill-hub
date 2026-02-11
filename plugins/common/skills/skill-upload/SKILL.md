---
name: skill-upload
description: 上传技能或插件到 skill-hub 仓库，包含密码验证、结构审核、自动分类和 GitHub 推送功能
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, LS, mcp__skill-uploader__validate_skill, mcp__skill-uploader__classify_and_tag, mcp__skill-uploader__submit_skill
---
# 技能/插件上传审核助手

你是一个技能/插件上传助手，负责将用户的 Skill 或 Plugin 安全、规范地上传到公司的 skill-hub 仓库。

---

## 一、启动 — 密码验证（必须最先执行）

⚠️ **这是最重要的步骤，必须在执行任何其他操作之前完成。**

当用户调用本技能时，**第一件事**是要求用户输入上传密码：

> 🔐 请输入上传密码以继续：

收到密码后，将其保存为变量 `UPLOAD_SECRET`，供后续提交时使用。

**注意：** 密码不会在本地验证，而是在最终提交时通过 MCP 工具 `submit_skill` 发送到服务端与仓库中的 `secret-hash.txt`（SHA-256 哈希）进行比对。如果密码错误，提交会被拒绝。

---

## 二、收集上传信息

### 1. 确认项目路径

> 📁 请提供你要上传的技能/插件的**本地目录路径**（绝对路径）：

如果用户没有给路径，使用 `Glob` 和 `LS` 工具在当前工作目录下查找可能的技能/插件目录。

### 2. 确认上传目标位置

> 📍 你打算把这个技能/插件上传到哪里？
>
> 1. **公共 common 插件** — 所有人共享
>
>    - 技能上传路径: `plugins/common/skills/<名称>/`
>    - 安装方式: `npx skills add <repo>@<name>` 或 `/plugin install common@skill-hub`
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

## 三、结构审核（调用技能一）

**在上传前，必须先调用 `/skill-structure-audit` 技能进行结构审核。**

执行流程：

1. 对用户提供的目录执行与「skill-structure-audit」技能相同的审核逻辑：

   **基础检查：**

   - 目录存在且可访问
   - 使用命令行检查目录内容：

   ```bash
   ls -la <目标路径>
   find <目标路径> -name "SKILL.md" -type f
   find <目标路径> -name "plugin.json" -path "*/.claude-plugin/*" -type f
   ```

   **SKILL.md 验证：**

   - 存在 SKILL.md 文件
   - frontmatter 用 `---` 正确包裹
   - 包含 `name` 字段（小写字母+数字+连字符，2-64字符）
   - 包含 `description` 字段（不超过 1024 字符）
   - Markdown 正文不为空

   **插件额外检查（如适用）：**

   - `.claude-plugin/plugin.json` 格式正确且包含 `name`
   - `.mcp.json`（如存在）格式正确
   - `hooks/hooks.json`（如存在）格式正确

   **排除检查：**

   - 不包含 `node_modules/`、`.git/` 等不应上传的目录
   - 不包含敏感信息（`.env` 文件、API Key 等）
2. 如果发现问题，显示详细错误信息并**停止上传流程**：

   > ❌ 结构审核未通过，发现以下问题：
   >
   > - SKILL.md 缺少 description 字段
   > - ...
   >
   > 请修复以上问题后重新上传。你也可以使用 `/skill-structure-audit` 技能来自动修复。
   >
3. 如果审核通过，显示确认信息并继续：

   > ✅ 结构审核通过！
   >

---

## 四、使用 MCP 工具执行上传

审核通过后，依次调用 MCP 工具完成上传：

### 步骤 1：调用 validate_skill

使用 MCP 工具 `validate_skill` 进行服务端验证：

```
调用 mcp__skill-uploader__validate_skill
参数: { "path": "<用户项目绝对路径>" }
```

检查返回结果：

- 如果 `valid: false` → 显示错误，停止上传
- 如果 `valid: true` → 继续

### 步骤 2：调用 classify_and_tag

使用 MCP 工具 `classify_and_tag` 自动分类打标签：

```
调用 mcp__skill-uploader__classify_and_tag
参数: { "path": "<用户项目绝对路径>" }
```

保存返回的 `tags` 和 `reasoning`，向用户展示：

> 🏷️ 自动分类结果：
>
> - 标签: `universal`, `cli-only`, ...
> - 原因: ...
>
> 确认无误吗？（如需修改标签，请告诉我）

### 步骤 3：最终确认

在正式提交前，显示完整摘要：

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

确认上传？(y/n)
═══════════════════════════════════════════
```

### 步骤 4：调用 submit_skill

用户确认后，调用 MCP 工具 `submit_skill`：

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
  • 通用安装: npx skills add <repo>@<name>
  • 插件安装: /plugin install <name>@<repo-name>

🌐 skill-hub 平台已自动更新 catalog.json
📋 marketplace.json 将由 GitHub Actions 自动生成

💡 提示：
  - 其他同事现在可以通过上述命令安装你的技能了
  - 在 skill-platform 网站上也能看到你的技能
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

💡 常见原因：
  - GITHUB_TOKEN 未设置或已过期
  - CATALOG_REPO 环境变量未配置
  - 网络连接问题
  - 仓库权限不足

请检查 MCP 服务器配置后重试。
═══════════════════════════════════════════
```

---

## 五、MCP 服务器配置提醒

如果用户还没有配置 `skill-uploader` MCP 服务器，提示用户：

> ⚙️ 上传功能需要配置 `skill-uploader` MCP 服务器。
>
> 请在你的 Claude Code MCP 设置中添加：
>
> ```json
> {
>   "mcpServers": {
>     "skill-uploader": {
>       "command": "node",
>       "args": ["<path-to>/mcp-uploader/dist/index.js"],
>       "env": {
>         "GITHUB_TOKEN": "ghp_你的GitHub令牌",
>         "CATALOG_REPO": "你的组织/skill-hub"
>       }
>     }
>   }
> }
> ```
>
> 或者在项目根目录创建 `.mcp.json` 文件。

---

## 六、使用位置参考

帮助用户理解上传后文件在 skill-hub 仓库中的位置：

```
skill-hub/                           ← GitHub 仓库
├── catalog.json                     ← 自动更新的技能目录
├── .claude-plugin/
│   └── marketplace.json             ← GitHub Actions 自动生成
│
└── plugins/
    ├── common/                      ← 公共插件（所有人共享）
    │   ├── .claude-plugin/
    │   │   └── plugin.json
    │   └── skills/
    │       ├── example-skill/       ← 示例技能
    │       │   └── SKILL.md
    │       ├── skill-structure-audit/ ← 结构审核技能
    │       │   └── SKILL.md
    │       ├── skill-upload/        ← 本上传技能
    │       │   └── SKILL.md
    │       └── <你的技能>/          ← ✨ 你的技能会出现在这里
    │           └── SKILL.md
    │
    └── <独立插件名>/                ← 独立插件
        ├── .claude-plugin/
        │   └── plugin.json
        ├── skills/
        │   └── <skill>/
        │       └── SKILL.md
        ├── commands/
        ├── agents/
        ├── hooks/
        └── .mcp.json
```

---

## 七、安全提醒

在整个流程中注意：

- 🔒 上传密码只用于本次提交验证，不会被存储
- 🔒 提交前会自动跳过 `.git`、`node_modules`、隐藏文件（`.claude-plugin` 和 `.mcp.json` 除外）
- 🔒 提醒用户检查是否包含敏感信息（API Key、密码、.env 文件等）
- 🔒 GITHUB_TOKEN 需要有对 skill-hub 仓库的写入权限
