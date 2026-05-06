# HLAgent — Windows 通用 Agent 产品需求（PRD v0.2）

> 融合 OpenClaw 与 Claude Code 设计哲学，面向 Windows 个人消费者的开源 + 市场化通用 AI 助理。
> 核心理念：**一个本地 daemon + 多前端**；**多 Agent 双模式**；**模型/工具可插拔**；**Windows 原生体验**。

**v0.2 变更摘要**：删除 Workflow 模式（多 Agent 协作只保留 Delegate / Peer 两模式）；Provider 收敛为 OpenAI / Anthropic 两套协议适配，本地模型走 OpenAI 兼容 endpoint；补漏 9 组（架构隔离 / Agent 元数据 / 多 Agent 协作 / Provider 能力面 / TodoWrite·Skills·MCP·Bash / Hooks / Memory / 权限审批 / 仓库与里程碑）。

---

## 1. Context（为什么做）

- 个人桌面侧已有 Claude Code（编码强，桌面/系统操作弱、Windows 体验差）和 OpenClaw 风格的 CLI Agent（多 provider、可扩展），但**没有一款 Windows 原生、面向通用桌面任务、支持多 Agent 不同模型/配置、并以开源 + 市场形式分发的产品**。
- 用户痛点：
  - Claude Code 在 Windows 需要 WSL，且面向纯编码场景，无法直接驱动 Office/浏览器/系统操作；
  - OpenClaw 等开源 CLI 配置门槛高、缺统一桌面 UX；
  - 现有产品大多绑定单一 provider，多模型按 agent 维度独立配置非常生硬。
- 期望产出：一款**资源占用低、启动快、Windows 原生**的通用 Agent 桌面应用，把 Claude 的「Skills/Subagents/Hooks/Memory/MCP」体系与 OpenClaw 的「multi-provider gateway/exec-approvals/插件生态」合并落地，并补齐 Windows 侧能力（UIA/Win32/PowerShell/托盘/快捷键/通知）。
- 商业模式：**开源 + 市场**（Skills / Sub-agents / MCP 插件 / 模型 Profile）。

参考来源（关键能力对标）：
- OpenClaw：[`openclaw/openclaw`](https://github.com/openclaw/openclaw)、[`docs.openclaw.ai`](https://docs.openclaw.ai)（sub-agents、hooks、skills、slash-commands、MCP、memory、providers、exec-approvals、gateway）、[`awesome-openclaw-agents`](https://github.com/mergisi/awesome-openclaw-agents)、[OpenClaw Desktop](https://github.com/agentkernel/openclaw-desktop)
- Claude Code：[Overview](https://code.claude.com/docs/en/overview)、[Sub-agents](https://code.claude.com/docs/en/sub-agents)、[Hooks](https://code.claude.com/docs/en/hooks)、[Skills](https://code.claude.com/docs/en/skills)、[Memory](https://code.claude.com/docs/en/memory)、[Settings](https://docs.anthropic.com/en/docs/claude-code/settings)

---

## 2. 目标用户与场景

### 2.1 首期目标用户
- **个人消费者**（Power user / 开发者 / 内容创作者 / 研究人员）。
- 设备：Windows 10 1809+ / Windows 11，x64 / ARM64。

### 2.2 典型场景（首发覆盖）
1. **通用桌面助理**：自然语言驱动文件批量整理、Office 文档生成/汇总、浏览器自动化、邮件/微信草稿。
2. **编码工程**：仓库阅读、Bash/PowerShell、Git、跑测试、生成补丁、本地 MCP 工具链。
3. **知识/内容工作流**：网页研究 → 笔记 → Markdown/PPT/报告产出，支持本地 RAG。
4. **多 Agent 协作**：「研究员 + 写作员 + 审稿员」、「计划员 + 执行员 + 检查员」等组合。

### 2.3 非目标（v1 不做）
- 服务器/云端多租户托管；移动端；Linux/macOS 原生壳（仅保留 CLI 跨平台）；企业 SSO/审计/集中策略下发（留 v2）。

---

## 3. 产品形态与技术栈

### 3.1 形态：Windows 桌面应用（主） + CLI（辅）
- **桌面壳**：Tauri 2（WebView2，安装包 < 15 MB，常驻 < 120 MB）。
- **本地 daemon**：随桌面/CLI 启动，监听 `127.0.0.1` HTTP + WS（也可 Named Pipe），所有前端共享同一会话。
- **CLI**：`hlagent` 子命令（参考 `claude` / `openclaw` CLI），可在桌面外独立使用，与桌面共享配置。

### 3.2 技术栈（性能/资源优先）
- **核心 daemon**：Rust（Tokio + Axum + serde + sqlx/SQLite）。
- **前端**：TypeScript + React + Vite + shadcn/ui + TailwindCSS + Lucide。
- **打包**：Tauri 2（含自动更新、代码签名、托盘、全局快捷键）。
- **跨进程**：Tauri IPC（前端↔壳） + HTTP/WS（壳/CLI ↔ daemon）。
- **存储**：SQLite（会话/消息/记忆/审计），文件层 `%APPDATA%\HLAgent\`。
- **凭证**：Windows DPAPI（`CryptProtectData`）加密 API Key，落盘文件 + 系统凭据管理器双写。

理由：
- Rust + Tauri 在冷启动、内存、单 exe 分发、Win32 互操作上综合最佳；
- React 前端复用 Claude/openclaw 桌面生态组件（diff viewer / Monaco / xterm.js）。

### 3.3 运行时与隔离
- **IPC 安全**：本地 daemon 默认 **Named Pipe + Windows ACL**（仅当前用户 SID 可访问）；HTTP/WS 仅作为开发模式或显式开启时启用，必须 `127.0.0.1` + Origin 校验 + 启动期生成 bearer token。
- **多用户**：每用户独立 daemon；数据放 `%LOCALAPPDATA%\HLAgent\`（漫游不带 secret），配置可选 `%APPDATA%\HLAgent\`。
- **Daemon 生命周期**：PID lock + 健康检查 + 崩溃自启 + 僵尸清理；空闲休眠（无活跃会话 N 分钟）唤醒延迟目标 < 200ms。
- **Schema migration**：SQLite 用 `sqlx migrations`；版本兼容矩阵记入 `docs/migrations.md`。
- **配置/数据迁移**：导入导出 `.hla` 包；卸载时保留数据策略（询问用户）；版本回退兼容窗口 = 当前 + 1 minor。
- **日志**：路径 `%LOCALAPPDATA%\HLAgent\logs\`，按日轮转，单文件 ≤ 10MB，保留 14 天，自动脱敏 secret 模式；用户一键打包反馈。
- **进程隔离**：Bash / Browser / UIA / Office 等工具运行在独立子进程，崩溃不波及 daemon；崩溃域文档化。
- **资源边界**：单会话 CPU/内存上限可配；API 并发与 token-per-minute 上限。

---

## 4. 总体架构

```
┌────────────────────────┐    ┌────────────────────────┐    ┌────────────────────┐
│ Tauri Desktop (React)  │    │ CLI (hlagent)          │    │ VSCode Ext (v2)    │
└──────────┬─────────────┘    └──────────┬─────────────┘    └─────────┬──────────┘
           │ Tauri IPC + HTTP/WS         │ HTTP/WS                    │
           └──────────────┬──────────────┴────────────────────────────┘
                          ▼
                ┌──────────────────────┐
                │  HLAgent Daemon      │  Rust core, single process
                │  ┌──────────────┐    │
                │  │ Session/Run  │    │
                │  │ Orchestrator │ ◄──┼──── Multi-Agent Modes (Delegate/Peer)
                │  ├──────────────┤    │
                │  │ Agent Runtime│ ◄──┼──── Per-agent: model/tools/permissions/memory/budget
                │  ├──────────────┤    │
                │  │ Tool Bus     │ ◄──┼──── Built-in tools + MCP clients + Skills loader
                │  ├──────────────┤    │
                │  │ Provider GW  │ ◄──┼──── OpenAI/Anthropic 双协议出入，多 provider 路由
                │  ├──────────────┤    │
                │  │ Hook Engine  │ ◄──┼──── 12 lifecycle events + scripts
                │  ├──────────────┤    │
                │  │ Memory Store │ ◄──┼──── 分层记忆 + 自动摘要 + 向量检索
                │  ├──────────────┤    │
                │  │ Permission/  │ ◄──┼──── allow/deny/ask + sandbox + audit log
                │  │ Approval     │    │
                │  └──────────────┘    │
                └──────────────────────┘
```

---

## 5. 核心功能需求

### 5.1 Agent 模型（每个 Agent 独立配置）
**每个 Agent 是一个 markdown 文件，YAML frontmatter + 系统提示**（融合 Claude/openclaw 习惯）：

```markdown
---
name: code-reviewer
description: 在变更落盘前执行代码审查，关注安全、风格、错误处理
model: anthropic/claude-sonnet-4              # 形如 <protocol>/<model> 或 <protocol>@<endpoint>/<model>
fallback_models: [openai/gpt-5, openai@local/qwen3-coder-30b]
tools: [Read, Grep, Bash(git diff:*), mcp__github]   # 与 Claude 兼容的工具白名单语法
permissions:
  bash: ask        # allow | deny | ask
  edit: deny
  network: allow
memory:
  scope: project   # global | user | project | session
  pin: ["CODING_STYLE.md"]
budget:
  max_tokens_per_run: 200000
  max_cost_usd: 0.50
hooks:
  PreToolUse: ./hooks/scan-secrets.ps1
color: "#7C3AED"
icon: shield-check
---
你是高级代码审查员……（系统提示）
```

**`model` 字段约定**：
- `<protocol>/<model>`：使用默认 endpoint（如 `openai/gpt-5-mini` → 走 `https://api.openai.com/v1`；`anthropic/claude-sonnet-4` → 走 `https://api.anthropic.com`）。
- `<protocol>@<endpoint-id>/<model>`：使用 Profile 中预定义的命名 endpoint（如 `openai@local/qwen3-coder-30b` → 走 `profiles.local.base_url`）。
- `protocol` 仅 `openai` / `anthropic` 两种取值（详见 §5.3）。

存放位置（沿用 Claude/openclaw 习惯，三层叠加）：
- `%APPDATA%\HLAgent\agents\` — 用户级
- `<repo>\.hlagent\agents\` — 项目级
- 内置 builtin（随安装包）

**Agent 元数据扩展（v0.2 新增）**：
- 描述性字段：`version` / `requires`（hlagent 最低版本、所需 Skill / MCP）/ `tags` / `license` / `author` / `maintainer` / `homepage` / `changelog`。
- **输入/输出契约**：可选 `inputs` / `outputs` JSON Schema（首版用于 Delegate 嵌套调用与未来扩展，便于参数校验与自动文档）。
- **测试 fixture**：约定 Agent 同目录 `tests/` 放黄金对话样本与期望工具调用，Marketplace 收录前 CI 必跑。
- **状态字段**：`enabled` / `visibility`（`private` / `public`）/ `callable_by_others`（防递归滥用，控制是否可被其他 Agent 通过 `Task` 调用）。
- **递归与深度**：子 Agent 调用深度上限默认 3；运行时做循环检测；上下文继承策略默认隔离，可白名单继承（如 `inherit_context: [memory, todos]`）。
- **模板向导**：UI New Agent 提供 `reviewer` / `researcher` / `planner` / `desktop-helper` 四类 starter 模板。

### 5.2 多 Agent 协作（**双模式可切换，会话内锁定**）

| 模式 | 说明 | 调度器实现 |
|---|---|---|
| **Delegate（主-子委派）** | 主 Agent 接收用户输入，按需 spawn 子 Agent；子 Agent 上下文隔离，完成后只回传结果摘要 | 默认；对标 Claude/openclaw sub-agents |
| **Peer（对等多体）** | 同会话内多个对等 Agent 通过消息总线相互发送消息（如 Debater A/B + Judge），用户可观摩或介入 | 消息总线 + 轮转策略 + 终止条件 |

**关键约束**：会话创建时二选一，会话生命周期内**外层模式不可切换**（避免上下文语义紊乱）；切换需新建会话或显式 fork。但 **Delegate 内允许临时起 Peer 子局**，**Peer 内的某个 Agent 也可临时 Delegate 子 Agent**（嵌套深度 ≤ 3）。

**协作运行时细节（v0.2 新增）**：
- **嵌套规则**：Delegate ⇄ Peer 可互相嵌套，但同一层级不可混用；嵌套深度 ≤ 3，超限报错并提示用户重构。
- **Peer 终止条件**（必须显式至少其一）：①最大轮数 ②达成投票 / Judge 宣告 ③用户中止 ④无新增信息 N 轮（语义停滞检测）⑤总 token / 成本超限。无任一条件配置则会话拒绝启动。
- **失败回退**：子 Agent 失败 → 重试 → 降级模型 → 降级 Agent → 通知用户 → 中止；触发 hook `SubagentStop` 携带原因。
- **资源争用**：共享 token 预算与 API quota；调度器支持优先级与抢占；同一文件锁互斥。
- **可视化追踪**：主-子调用树、Peer 消息时序，可折叠展开；transcript 内联展示子 Agent 内部 prompt / 响应（可隐藏）。
- **可逆性**：Peer 模式下用户可随时插话注入消息或暂停；Delegate 模式下用户可中止当前子 Agent 而保留主 Agent 状态。

### 5.3 模型 / Provider Gateway（双协议适配）
- **协议适配层**：仅实现 ① **OpenAI Chat Completions / Responses** ② **Anthropic Messages** 两套；其他厂商通过其官方 OpenAI 兼容 endpoint 接入，或不在 v1 范围。
- **官方原生支持**：
  - Anthropic API（`https://api.anthropic.com`）
  - OpenAI API（`https://api.openai.com/v1`）
- **OpenAI 兼容 endpoint（统一走 OpenAI 协议）**：
  - 本地：Ollama、llama.cpp（`server` 模式）、vLLM、LM Studio
  - 私有：自建 / 企业内 OpenAI-Compatible Gateway（如 LiteLLM、One-API、New API）
- **不在 v1 范围**：Gemini、Bedrock、Vertex、Azure OpenAI、DeepSeek、Qwen DashScope、Mistral、xAI、Moonshot 的**原生协议**。用户可通过自建 OpenAI 兼容代理接入；官方协议留 v1.x roadmap。
- **Profile 概念**（每个 Agent / 每次调用都可引用）：
  ```yaml
  profiles:
    fast:    { protocol: openai,    base_url: https://api.openai.com/v1,   model: gpt-5-mini,        temperature: 0.2 }
    deep:    { protocol: anthropic, base_url: https://api.anthropic.com,   model: claude-sonnet-4,   max_tokens: 8000 }
    local:   { protocol: openai,    base_url: http://127.0.0.1:11434/v1,   model: qwen3-coder-30b }
  ```
- **路由策略**：默认 → 失败降级（fallback chain）→ 按成本上限切换。
- **能力**：流式、tool_use、vision、prompt cache、并发限流、重试退避、token 计数与成本估算（统一计价表，可热更新）。
- **隐私模式**：「仅本地 endpoint」开关，强制只调用白名单（`127.0.0.1` / `localhost` / 用户自定义内网段）的 endpoint，违反则拒绝。

**Gateway 能力面（v0.2 新增）**：
- **协议差异填平**：tool / function calling、stream chunk、reasoning / thinking、image part、PDF part 在 OpenAI 与 Anthropic 之间统一抽象；不能映射的能力在 Agent 元数据 `requires_capabilities` 声明，调度时校验。
- **能力探测与降级**：vision / tool_use / long-context / cache / json_mode 不支持时按 Agent 策略「降级 / 警告 / 拒绝」三档处理。
- **上下文窗口**：每 model ctx 上限做表，超限自动 split / compact / 切换更长上下文模型。
- **流式取消**：用户中止 / 超时 / 网络掉线干净结束并标注原因；不留半成品工具调用。
- **配额限流**：per-endpoint RPS、并发、token-per-minute；429 指数退避；多 key 池轮询。
- **失败转移**：fallback chain（主 → 备 1 → 备 2）+ 失败原因细分（速率 / 鉴权 / 上下文 / 内容审查 / 网络）。
- **价格表**：本地 JSON 维护，可手改；CI 周期更新；UI 显示成本估算。
- **本地模型管理**：模型下载（断点续传 + 镜像源）、量化、显存预算、CPU/GPU 后端选择、热加载、共享同一 Ollama / llama.cpp 实例。
- **本地模型许可**：标识 license（Llama 社区 / 商用 / Apache 等）；Marketplace 引用本地模型需声明。
- **Prompt cache 一致性**：OpenAI / Anthropic cache key 计算与命中提示统一展示给用户。
- **隐私分级**：endpoint 标 `cloud` / `private-cloud` / `local`，Agent 声明最低隐私级别，gateway 强制满足，违反则拒绝。
- **离线行为**：完全离线时仅本地 endpoint；网络中断时未完成请求按 fallback 策略处理与重试。

### 5.4 工具体系（Tool Bus）

#### 5.4.1 内置工具(与 Claude 工具名兼容,便于复用 prompt)
- 文件:`Read` / `Write` / `Edit` / `MultiEdit` / `Glob` / `Grep`
- Shell:`Bash`(PowerShell / cmd / Git Bash / WSL,自动选择,逐命令白名单)
- Web:`WebFetch` / `WebSearch`(可配置搜索 provider)
- Plan:`TodoWrite`(任务清单,UI 同步显示)
- 任务:`Task`(spawn 子 Agent;Delegate 模式核心)
- **Windows 原生**(差异化亮点):
  - `Desktop`(启动应用、窗口控制、剪贴板、通知、Toast)
  - `UIA`(UI Automation 树读取与点击/输入)
  - `Screen`(截图、OCR、视觉理解输入)
  - `Office`(COM 自动化 Word/Excel/PowerPoint/Outlook)
  - `Browser`(Playwright,Chrome/Edge profile 复用)
  - `Schedule`(任务计划程序)
  - `Hotkey`(注册/响应全局快捷键,用户许可下)

**`Bash` 工具规范（v0.2 新增）**：
- **多 shell 后端**：可选 PowerShell / cmd / Git Bash / WSL，每条命令可指定后端，未指定时按平台默认（Windows → PowerShell）。
- **Long-running 实时回显**：行缓冲、UI 实时滚动；`Ctrl+C` 信号正确传递到子进程。
- **环境变量**：默认 inherit + 显式 `env:` override；敏感变量（含 `KEY` / `TOKEN` / `SECRET`）默认从 inherit 中剔除，可显式声明放行。
- **当前目录策略**：默认工作区根；显式 `cwd:` 覆盖；不允许穿越权限白名单。
- **PSReadLine 干扰**：PowerShell 调用使用 `-NoProfile -NonInteractive`，避免主机配置干扰。
- **编码**：UTF-8 优先；自动检测 GBK / OEM 解码失败时回退；输出强制 UTF-8。
- **命令行长度上限**：超长自动分批或落盘临时脚本执行。
- **白名单粒度**：细到子命令（`git diff:*`、`npm run:test*`、`gh pr:view`）。

**`TodoWrite` 工具规范（v0.2 新增）**：
- **持久化**：写入 SQLite，会话级隔离；可显式提升到 project 级。
- **跨 Agent 共享**：同会话主-子 Agent 共享同一 todo 列表；Peer 模式下每 Agent 独立列表 + 共享只读视图。
- **撤销/回退**：每次写入产生 revision，可一键回退最近 N 步。
- **UI 双向同步**：用户在 UI 勾选 / 重排会反向写入 Agent 上下文（下次轮次注入）。
- **批量导入导出**：Markdown checklist / JSON。

#### 5.4.2 MCP 集成（双向）
- **MCP 客户端**：连接外部 MCP server（stdio / SSE / streamable-http），工具自动并入 Tool Bus，命名空间 `mcp__<server>__<tool>`。
- **MCP 服务端**：HLAgent 自身工具可暴露为 MCP server，被其他 IDE/Agent 复用。
- 配置文件：`%APPDATA%\HLAgent\mcp.json`（与 Claude Desktop / openclaw 兼容格式）。

**MCP 运行时规范（v0.2 新增）**：
- **服务发现**：mDNS（局域网）+ HTTP `/mcp/manifest`（私有）+ 配置文件三种来源叠加，去重按 server id。
- **健康检查与自动重连**：周期 ping；连续失败超阈值标记 unhealthy 并通知用户；重连指数退避。
- **超时上限**：单次 tool 调用默认 60s，可配上限；流式调用按 chunk 心跳。
- **stdio server 子进程**：生命周期由 daemon 托管，崩溃自动重启（带退避），僵尸回收，资源边界（CPU/RSS）。
- **TLS 证书校验**：默认严格校验；企业可导入私有 CA，禁止跳过校验。
- **命名空间冲突**：多 server 同名工具显式提示，按 `server priority` 解析；用户可在 UI 重命名 alias。

#### 5.4.3 Skills（可插拔知识/流程包，progressive disclosure）
- 目录结构（沿用 Claude `SKILL.md`）：
  ```
  skills/<name>/
    SKILL.md          # frontmatter: name, description, when_to_use, allowed_tools
    references/       # 进一步检索的引用
    scripts/          # 可执行脚本
    examples/
  ```
- 触发：description + when_to_use 经索引，按需懒加载，避免吃满上下文。
- 可由 Agent frontmatter 显式 `skills: [pdf-form-fill, excel-pivot]`。

**Skills 运行时规范（v0.2 新增）**：
- **索引性能**：千级 Skills 走前缀树 + 关键字倒排 + 向量预筛三段式，命中后才完整载入 SKILL.md。
- **热加载**：文件系统监听（`%APPDATA%\HLAgent\skills\` + `<repo>\.hlagent\skills\`），变更秒级生效。
- **脚本沙箱**：`scripts/` 内可执行文件继承 Permission（同 Bash 工具），不可绕过审批。
- **`when_to_use` 调试**：UI 显示当次命中的 Skills 与命中原因（关键字 / 向量分数），用户可手动启用/禁用以调试触发。
- **依赖解析**：Skill 间可声明 `requires: [other-skill]`；构建依赖图，循环依赖在加载期报错并定位。

### 5.5 Hooks（生命周期挂钩，对标 Claude 12 事件）
- 事件：`SessionStart` / `SessionEnd` / `UserPromptSubmit` / `PreToolUse` / `PostToolUse` / `Notification` / `Stop` / `SubagentStop` / `PreCompact` / `PostCompact` / `Error` / `BudgetExceeded`
- 处理器：PowerShell / Bash / Python / 任意可执行；以 JSON over stdio 通信，可阻断/改写工具调用、注入消息。
- 配置位置（叠加）：用户级 `%APPDATA%\HLAgent\hooks\`、项目级 `.hlagent\hooks\`。
- 应用：Secret 扫描、敏感目录拦截、自动 lint、审计上报、Toast 通知。

**Hook 运行时规范（v0.2 新增）**：
- **执行模型**：单 hook 默认超时 5s，可配上限 30s；同事件多 hook 默认顺序串行（保证可预测），可显式声明 `parallel: true`。
- **失败语义**：阻断类（`PreToolUse` / `UserPromptSubmit` / `PreCompact` 等）默认 **fail-closed**（失败=阻断）；通知类（`Notification` / `PostToolUse` / `SessionEnd`）默认 **fail-open**（失败=放行 + 警告）。
- **JSON 协议版本**：stdin / stdout schema 加 `version` 字段，向后兼容字段新增策略；不兼容变更走 major 升级。
- **可观测**：hook 输出在 UI 高亮显示、独立日志文件、内置调试器（可重放某次事件用于排查）；性能预算指标（每事件总耗时 P95）。
- **沙箱**：hook 同样受 Permission 约束，不可绕过审批；可在 hook frontmatter 声明所需工具 / 网络。
- **预置 hook 库**：secret 扫描、敏感目录拦截、git auto-commit、Toast 通知、Telegram / IM 通知、企业 DLP（v1.1）。

### 5.6 Slash Commands
- 自定义命令：`<scope>/commands/*.md`，frontmatter 指定 `description / tools / model`。
- 内置：`/agents`、`/skills`、`/mcp`、`/memory`、`/cost`、`/clear`、`/compact`、`/model`、`/permissions`、`/diff`、`/replay`、`/export`。

### 5.7 Memory（分层记忆）
- 层级：**Global（系统级）→ User（个人偏好）→ Project（仓库级 `HLAGENT.md`）→ Session（会话级）**。
- 写入方式：
  - 显式：`#` 前缀直接写入；`/memory edit`；UI 面板。
  - 自动：会话结束抽取「关键事实/偏好/术语」入库（可关闭）。
- 检索：上下文注入 + 向量召回（嵌入模型走 provider gateway，可选本地 `bge-m3` 等）。
- 上下文压缩：`/compact` 显式 + 接近窗口阈值自动分级摘要（保留任务计划与未决 TODO）。

**Memory 运行时规范（v0.2 新增）**：
- **优先级与冲突**：Session > Project > User > Global；冲突时显式提示并显示来源，让用户决定保留哪一层。
- **TTL / 衰减**：自动遗忘机制（默认 90 天未命中衰减权重，半年未命中归档）；`pin` 项不衰减；命中次数加权。
- **可解释性 UI**：每条注入的记忆显示「来源 / 命中原因 / 相关度分数」，便于用户调试。
- **隐私脱敏**：禁止入库正则名单（API key / 密码 / 身份证 / 手机号 / 信用卡），命中即提示用户决定是否手动 redact。
- **同步**：默认关闭；可选用户云盘 / Git / WebDAV 通道，端到端加密（用户密码派生密钥）。
- **向量库**：首选 `sqlite-vec`（零外部依赖、与主库共存）；可选 `lancedb`（更大规模、更高吞吐）。
- **嵌入模型**：默认本地 `bge-m3` 或 `text-embedding-qwen`，跟随 endpoint 隐私级别匹配；云端 endpoint 仅在隐私模式允许时调用。
- **多语言**：embedding 跨中英检索效果做评测样本进入 §10 评测集，回归监控。

### 5.8 权限与审批（Permission / Exec Approvals）
- 三态：`allow` / `ask` / `deny`，对每类工具/每个具体命令可配。
- 审批 UI：弹窗显示工具、参数、影响范围（文件/网络/进程/注册表），「本次/本会话/永久 + 范围」。
- 沙箱：
  - 文件：白名单根目录，默认仅工作区与 `%TEMP%`；
  - 进程：低完整性令牌 + Job Object 限制；
  - 网络：可选 SOCKS/HTTP 代理 + 域名白名单（企业可下发）。
- 审计日志：每次工具调用、模型请求、修改的文件 hash 入 SQLite，可导出。

**审批与策略增强（v0.2 新增）**：
- **风险分级（自动判定）**：依据工具 + 参数自动归类 `high / medium / low`。`rm -rf` / `reg add HKLM\*` / 外发文件 / 注册表写 / UAC 触发 → 强制 `ask` 且醒目提示（配色 + 可选振铃 / Toast）。
- **批量审批**：一次审批同类（按通配 / 正则记忆为规则），UI 显示当前会话已豁免列表，可单条撤销。
- **策略 DSL**：路径 glob、命令正则、网络域名 / 端口 ACL；项目级 `.hlagent/permissions.yaml`，可被 Marketplace 资产声明（安装时双向校验）。
- **远程审批**（v1.1）：长任务推送审批到手机 / 邮件 / IM；占位 API + 协议规范。
- **审批超时**：默认 60s 后拒绝；超时时间可配；后台静默时降级行为为「暂停而非默拒」（避免误中止）。
- **审计完整性**：日志 append-only + 每日 **hash chain** 摘要（前一日 hash 嵌入今日记录）；导出 `.audit.jsonl` 可独立校验。
- **dry-run / plan 模式**：所有可变更工具支持先输出影响计划（要写哪些文件 / 要发哪些请求），用户确认后再真正执行。
- **可逆性标识**：每个工具调用标 `reversible` / `irreversible`，UI 显著区分；不可逆操作（删除、`git push --force`、外发邮件）强制 `ask` 且禁用「永久允许」。
- **配额限制**：单次 / 单会话 / 单天文件写入数量、网络外发字节数、新建进程数上限（可配）。

### 5.9 桌面 UX（关键界面）
- **主窗口**：左 Sidebar（会话/Agent/Skills/MCP/记忆/任务/成本），中 对话区（消息 + 流式 + diff/图像/表格 inline），右 上下文检查器（注入的系统提示/记忆/工具/token 统计）。
- **Agent Picker**：会话顶栏切换 Agent，悬停预览能力 + 模型 + 工具白名单；新建会话时强制选择**协作模式**。
- **任务计划面板**：TodoWrite 实时同步，可手动勾选/重排，主-子 Agent 任务可视化树。
- **审批弹窗**：醒目的「拒绝/允许一次/允许本会话/永久允许」按钮。
- **diff 查看器**：基于 Monaco；变更可批量回滚。
- **托盘 + 全局快捷键**：`Ctrl+Alt+Space` 唤起命令栏（类似 Raycast）；选中文本 → 快捷送入 Agent。
- **多窗口/会话**：每个会话独立 token & cost 计数；一键 Pin 置顶；会话可导出 `.hla` 包（含设置但脱敏 secret）。
- **暗色/浅色 + 中英文 i18n**；可访问性（键盘全操作、屏幕阅读器友好）。

### 5.10 市场（Marketplace）
- 资产类型：**Agent / Skill / MCP Server / Slash Command / Hook 包 / Provider Profile**。
- 发布：CLI `hlagent publish`，签名（minisign / Sigstore）。
- 安装：UI 一键安装；自动校验 manifest、声明的工具/网络权限，二次确认。
- 治理：分级（Verified / Community / Experimental），举报 + 静默禁用清单。

### 5.11 可观测性
- Token & 成本仪表板（按 Agent / 会话 / 日 / 月聚合）。
- Trace（OpenTelemetry 兼容）：每条消息 → 工具调用 → 模型请求 全链路；可导出 / 私有 OTLP 接收端。
- Replay：保存会话 + 模型响应原文，可回放（脱离网络）。

---

## 6. 关键易遗漏点（已补齐）

1. **会话级模式锁定**：协作模式不可中途切换，避免子 Agent 遗留态污染对等模式（嵌套见 §5.2）。
2. **预算与成本止损**：每 Agent + 每会话双重预算，超出走 BudgetExceeded hook（默认暂停 + 通知）。
3. **上下文压缩可控**：`PreCompact` 钩子让用户决定要保留什么；压缩前自动落盘 transcript。
4. **多 Agent 消息总线**：Peer 模式需显式终止条件（轮数 / 一致性 / 用户中止 / 语义停滞 / 成本上限），避免无限对话。
5. **隐私模式 / 数据驻留**：一键「不发送代码片段到云端」「仅本地 endpoint」「禁用遥测」。
6. **凭证保管**：API Key 走 DPAPI；导出 `.hla` 时自动脱敏；剪贴板 token 检测告警。
7. **Secret 扫描**：内置 PreToolUse 钩子，扫描即将发往模型的内容（git secrets 规则）。
8. **网络环境**：HTTP/SOCKS 代理、PAC、企业 CA 证书导入；离线安装包（含本地模型 manifest）。
9. **Vision/语音**：截图 → 视觉模型；语音输入（Windows Speech / Whisper.cpp 本地）；TTS 输出。
10. **WSL/Cygwin 互通**：Bash 工具可指定 shell 后端；路径自动 `\\wsl$\` ↔ `/mnt/c/` 转换。
11. **崩溃报告 & 自动更新**：Tauri Updater + 可选崩溃上报（默认关闭）。
12. **代码签名**：发行版 EV 证书签名，避免 SmartScreen 警告；安装器（MSIX + EXE 双发）。
13. **冷启动 / 占用目标**：托盘冷启动 < 1.5s；空闲常驻 ≤ 120MB；会话窗口打开 < 300ms。
14. **国际化**：UI 中英；提示词自动语种适配；错误信息本地化。
15. **可访问性**：所有交互键盘可达；高对比度主题；屏幕阅读器 ARIA。
16. **遥测/隐私**：默认仅本地，开启需明示；遵守 GDPR/PIPL 风格的清晰可关。
17. **崩溃恢复**：会话自动断点；意外退出后下次启动询问续跑。
18. **测试基线**：内置 Agent 评测脚本（小型任务集），用于切换模型/Profile 时回归。

---

## 7. 安全与合规

- 所有外部进程/网络调用过 Permission 中间件，默认拒绝高危（删除、注册表、HKLM、Win+R 输入、UAC 弹窗）。
- 内置 SmartScreen / Defender 友好（签名 + 公开发布渠道）。
- 用户数据**默认全部本地**；遥测最小化、可关闭、文档透明。
- 漏洞报告流程（SECURITY.md）；安装包 SBOM。

---

## 8. 非功能性需求（NFR）

| 维度 | 目标 |
|---|---|
| 安装包大小 | ≤ 25 MB（不含本地模型） |
| 冷启动 | ≤ 1.5s（首屏） |
| 内存占用 | 空闲 ≤ 120MB；活跃会话 ≤ 350MB |
| 并发 | 单会话 ≥ 3 子 Agent 并行；全局 ≥ 8 |
| 模型切换 | 失败降级 ≤ 1.5s |
| 崩溃恢复 | 100% 会话断点续跑 |
| 离线 | 配齐本地 provider 后可完全离线运行 |

---

## 9. 仓库与里程碑

### 9.1 仓库布局（建议）
```
hlagent/
  apps/
    desktop/        # Tauri + React
    cli/            # Rust CLI binary
  crates/
    daemon/         # Rust daemon (HTTP/WS server)
    core/           # 调度/Agent runtime/Hook/Memory/Permission
    gateway/        # Provider 抽象 + 路由
    tools/          # 内置工具（含 windows-native 子模块）
    mcp/            # MCP client + server
    skills/         # Skills loader
  packages/
    sdk-ts/         # TS SDK（前端 + 第三方插件）
  marketplace/      # 市场清单与签名工具
  docs/
  examples/
```

### 9.2 里程碑（建议）
- **M-1（2 周）**：威胁模型 + 安全设计评审 + 架构 ADR；选型确认（Tauri / sqlite-vec / 嵌入模型 / IPC 方案）。
- **M0（4 周）**：daemon 雏形 + Provider gateway（OpenAI 协议 + Anthropic 协议；本地经 OpenAI 兼容 endpoint 接 Ollama / llama.cpp）+ 基础内置工具（Read/Write/Edit/Bash/Grep）+ CLI MVP。
- **M1（4 周）**：桌面 Tauri 壳 + 会话 UI + Permission 弹窗 + Memory + Slash Commands + Skills loader。
- **M2（4 周）**：Delegate + Peer 多 Agent + Hook 引擎 + MCP 双向。
- **M2.5（2 周）**：稳定性硬化 sprint（崩溃恢复、IPC 安全、子进程隔离、审计 hash chain、回归评测集）。
- **M3（4 周）**：Windows 原生工具集（Desktop/UIA/Office/Browser/Screen）+ 审批策略 DSL + 成本面板。
- **M4（4 周）**：Marketplace + 自动更新 + OpenAI 兼容 endpoint 适配增强（LiteLLM / One-API 等指南与样例 profile）。
- **v1.0**：稳定 API、签名发行、文档站、首批 20 个官方 Skills/Agents。

### 9.3 工程规范与发布（v0.2 新增）
- **贡献者文档**：`CONTRIBUTING.md` / `CODE_OF_CONDUCT.md` / `SECURITY.md`；默认 **DCO**（不签 CLA）；Issue / PR 模板；架构决策记录目录 `docs/adr/`。
- **发布流程**：**SemVer**；Release Train 每 4 周；Beta / Canary / Stable 三通道；LTS 标识（自 v1.0 起每年一个 LTS，维护 12 个月）。
- **CI 矩阵**：Win10×x64、Win11×x64、Win11×ARM64（M0 不强制 ARM64 跑 E2E，但 CI 必须跑构建）；E2E 在 GitHub Actions Windows runner 上。
- **依赖供应链**：`cargo audit` / `npm audit` 强制 CI；**SBOM**（CycloneDX）随发行包；签名（minisign / Sigstore）；可重现构建目标。

---

## 10. 验证（Verification）

- **单元/集成测试**：Rust（`cargo test`）+ TS（`vitest`）；CI 跑 Windows runner。
- **E2E**：Playwright 驱动 Tauri 窗口；脚本化覆盖：会话创建 → 选择模式 → 多 Agent 协作 → 审批 → 输出导出。
- **Agent 评测集**：`/eval` 目录放 30+ 任务（编码/桌面/知识三类各 10），切 provider 时回归对比。
- **手动验收清单**：
  - [ ] 在断网环境下用本地 endpoint 完整跑通编码 + 桌面 + 知识三类任务
  - [ ] 两种多 Agent 模式（Delegate / Peer）分别完成一个示例会话，崩溃恢复可用
  - [ ] 安装包 SmartScreen 无警告；常驻内存达标
  - [ ] 任意 Agent 切换模型/Profile 后仍能回归通过
  - [ ] 凭证、敏感数据无落盘明文；导出 `.hla` 无 secret
- **关键命令**（开发期）：
  ```powershell
  # 启动 daemon
  cargo run -p hlagent-daemon
  # 启动桌面（开发模式）
  pnpm -C apps/desktop tauri dev
  # 评测
  hlagent eval run --suite default --profile fast
  ```

---

## 11. 关键修改文件 / 待建模块（首版）

- 新建仓库 `e:/AI/HLAgent/`，按 §9.1 布局初始化。
- 关键模块（首批落盘）：
  - `crates/daemon/src/main.rs`：HTTP/WS server bootstrap。
  - `crates/core/src/agent.rs`：Agent 定义解析（frontmatter + system prompt）。
  - `crates/core/src/orchestrator.rs`：Delegate / Peer 两模式调度器（含嵌套与终止条件）。
  - `crates/core/src/permission.rs`：审批中间件。
  - `crates/core/src/memory.rs`：分层记忆。
  - `crates/gateway/src/protocol/{openai,anthropic}.rs`：双协议适配（本地模型经 OpenAI 兼容 endpoint 复用 `openai.rs`）。
  - `crates/tools/src/{fs,bash,web,desktop,uia,office,browser,screen}.rs`：内置工具。
  - `crates/mcp/src/{client,server}.rs`。
  - `apps/desktop/src/`：会话 UI、Agent Picker、审批弹窗、任务计划面板、成本面板。
  - `apps/cli/src/main.rs`：CLI 入口。
  - `marketplace/manifest-schema.json`：市场资产 manifest。

---

## 12. 开放问题（待后续会话确认）

1. **本地 daemon 的进程模型**：常驻 vs 按需 spawn？建议常驻 + 空闲休眠（§3.3）。
2. **Windows ARM64 是否首发**：M0 仅跑 ARM64 构建 CI，正式 E2E 与签名发行从 v1.0 / v1.1 加入？
3. **Marketplace 后端**：自建 vs 复用 GitHub Releases + 索引 JSON？建议先 GitHub。
4. **遥测数据保留期**：默认 0（不收集）；如启用，建议 ≤ 30 天。
5. **是否绑定 `e:/AI/learn-claude-code` 现有学习代码作为参考样例**（仅文档引用，不作为依赖）。
6. **OpenAI 兼容代理样例**：是否官方维护一份 LiteLLM / One-API 接 Gemini / Bedrock / Vertex / Azure / DeepSeek / Qwen 的示例 profile，还是完全社区维护？
7. **默认 embedding 模型**：本地优先 `bge-m3` vs `text-embedding-qwen` vs 让用户首启选择？
