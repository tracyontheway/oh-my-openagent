# Oh My OpenCode 完整功能文档

> 本文档详细讲解 oh-my-opencode 项目的完整功能架构

**技术规模**: 1268 个 TypeScript 文件，约 16 万行代码

---

## 目录

1. [项目概述](#一项目概述)
2. [架构设计](#二架构设计)
3. [Agents 系统](#三agents-系统)
4. [Category 系统](#四category-系统)
5. [Hooks 系统](#五hooks-系统)
6. [Tools 系统](#六tools-系统)
7. [Features 模块](#七features-模块)
8. [MCP 系统](#八mcp-系统)
9. [CLI 命令系统](#九cli-命令系统)
10. [配置系统](#十配置系统)
11. [Skills 技能系统](#十一skills-技能系统)
12. [Commands 命令系统](#十二commands-命令系统)
13. [核心工作流程](#十三核心工作流程)
14. [Hashline 编辑工具](#十四hashline-编辑工具)
15. [后台代理引擎](#十五后台代理引擎)
16. [Tmux 集成](#十六tmux-集成)
17. [模型解析流水线](#十七模型解析流水线)
18. [会话恢复系统](#十八会话恢复系统)
19. [OAuth 2.0 + PKCE 系统](#十九oauth-20--pkce-系统)
20. [上下文注入系统](#二十上下文注入系统)
21. [迁移系统](#二十一迁移系统)

---

## 一、项目概述

Oh My OpenCode 是一个为 OpenCode 设计的多模型 AI 代理编排框架（Harness）。它将单一 AI 代理转变为协调工作的开发团队，实现真正的"无需人工干预即可完成工作"。

### 核心理念

- **不锁定任何模型提供商** - Claude、GPT、Gemini、Kimi、GLM 都可以协同工作
- **人工干预是失败信号** - 理想状态下，代理应自主完成任务
- **代码不可区分** - 代理生成的代码应与资深工程师编写的代码无异

### 技术规模

| 组件 | 数量 |
|------|------|
| TypeScript 文件 | 1268 个 |
| 代码行数 | ~160k LOC |
| 专业代理 (Agents) | 11 个 |
| 生命周期钩子 (Hooks) | 48 个 |
| 工具 (Tools) | 26 个 |
| 功能模块 (Features) | 19 个 |

---

## 二、架构设计

### 初始化流程

```
OhMyOpenCodePlugin(ctx)
  ├─→ loadPluginConfig()         # JSONC 解析 → 项目/用户配置合并 → Zod 验证 → 迁移
  ├─→ createManagers()           # TmuxSessionManager, BackgroundManager, SkillMcpManager, ConfigHandler
  ├─→ createTools()              # SkillContext + AvailableCategories + ToolRegistry (26 工具)
  ├─→ createHooks()              # 三层: Core(39) + Continuation(7) + Skill(2) = 48 钩子
  └─→ createPluginInterface()    # 8 个 OpenCode 钩子处理器 → PluginInterface
```

### 三层架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     Planning Layer (规划层)                      │
│   Prometheus (规划师) + Metis (差距分析) + Momus (审查员)          │
└─────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Execution Layer (执行层)                      │
│                    Atlas (指挥官/协调器)                          │
└─────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Worker Layer (工作层)                         │
│   Sisyphus-Junior + Oracle + Explore + Librarian + 前端专家       │
└─────────────────────────────────────────────────────────────────┘
```

### 8 个 OpenCode 钩子处理器

| Handler | Purpose |
|---------|---------|
| `config` | 6阶段: provider → plugin-components → agents → tools → MCPs → commands |
| `tool` | 26 个注册工具 |
| `chat.message` | 首条消息变体、会话设置、关键词检测 |
| `chat.params` | Anthropic effort 级别调整 |
| `chat.headers` | Copilot x-initiator 头注入 |
| `event` | 会话生命周期 (created, deleted, idle, error) |
| `tool.execute.before` | 前置工具钩子 (文件保护、标签截断、规则注入) |
| `tool.execute.after` | 后置工具钩子 (输出截断、元数据存储) |
| `experimental.chat.messages.transform` | 上下文注入、思考块验证 |

---

## 三、Agents 系统

项目包含 **11 个专业代理**，每个代理有特定的职责和模型配置。

### 核心代理

| 代理 | 模型 | 温度 | 模式 | Fallback 链 | 用途 |
|------|------|------|------|-------------|------|
| **Sisyphus** | claude-opus-4-6 max | 0.1 | all | k2p5 → kimi-k2.5 → gpt-5.4 medium → glm-5 → big-pickle | 主编排器，规划+委托 |
| **Hephaestus** | gpt-5.4 medium | 0.1 | all | — | 自主深度工作者 |
| **Oracle** | gpt-5.4 high | 0.1 | subagent | gemini-3.1-pro high → claude-opus-4-6 max | 只读架构咨询 |
| **Librarian** | minimax-m2.7 | 0.1 | subagent | minimax-m2.7-highspeed → claude-haiku-4-5 → gpt-5-nano | 外部文档/代码搜索 |
| **Explore** | grok-code-fast-1 | 0.1 | subagent | minimax-m2.7-highspeed → minimax-m2.7 → claude-haiku-4-5 → gpt-5-nano | 快速代码库 grep |
| **Multimodal-Looker** | gpt-5.3-codex medium | 0.1 | subagent | k2p5 → gemini-3-flash → glm-4.6v → gpt-5-nano | PDF/图像分析 |

### 规划代理

| 代理 | 模型 | 温度 | 模式 | Fallback 链 | 用途 |
|------|------|------|------|-------------|------|
| **Prometheus** | claude-opus-4-6 max | 0.1 | — | gpt-5.4 high → gemini-3.1-pro | 战略规划师（内部） |
| **Metis** | claude-opus-4-6 max | **0.3** | subagent | gpt-5.4 high → gemini-3.1-pro high | 预规划顾问 |
| **Momus** | gpt-5.4 xhigh | 0.1 | subagent | claude-opus-4-6 max → gemini-3.1-pro high | 计划审查员 |

### 编排代理

| 代理 | 模型 | 温度 | 模式 | Fallback 链 | 用途 |
|------|------|------|------|-------------|------|
| **Atlas** | claude-sonnet-4-6 | 0.1 | primary | gpt-5.4 medium | Todo 列表编排器 |
| **Sisyphus-Junior** | claude-sonnet-4-6 | 0.1 | all | 用户可配置 | 分类任务执行器 |

### 代理模式说明

- **primary**: 尊重 UI 选择的模型，使用 fallback 链
- **subagent**: 使用自己的 fallback 链，忽略 UI 选择
- **all**: 在两种上下文中都可用（Sisyphus-Junior）

### 工具权限限制

| 代理 | 禁用工具 |
|------|----------|
| Oracle | write, edit, task, call_omo_agent |
| Librarian | write, edit, task, call_omo_agent |
| Explore | write, edit, task, call_omo_agent |
| Multimodal-Looker | 除 read 外所有工具 |
| Atlas | task, call_omo_agent |
| Momus | write, edit, task |

### 模型解析

模型解析遵循 4 步流程：

```
resolveModel(input)
  1. Override: UI 选择的模型（仅 primary agents）
  2. Category default: 从分类配置继承
  3. Provider fallback: AGENT_MODEL_REQUIREMENTS 链
  4. System default: 最终回退
```

---

## 四、Category 系统

Category 的本质：**不选择模型名称，而是选择任务类型**，系统自动映射到最佳模型。

### 8 个内置分类

| 分类 | 默认模型 | 用途 |
|------|----------|------|
| **visual-engineering** | gemini-3.1-pro high | 前端、UI/UX、设计、动画 |
| **ultrabrain** | gpt-5.4 xhigh | 深度逻辑推理、复杂架构决策 |
| **deep** | gpt-5.3-codex medium | 自主问题解决、深入研究后行动 |
| **artistry** | gemini-3.1-pro high | 高度创意/艺术任务、新颖想法 |
| **quick** | gpt-5.4-mini | 琐碎任务、单文件修改、拼写修正 |
| **unspecified-low** | claude-sonnet-4-6 | 不适合其他分类的任务，低难度 |
| **unspecified-high** | claude-opus-4-6 max | 不适合其他分类的任务，高难度 |
| **writing** | gemini-3-flash | 文档、散文、技术写作 |

### Category + Skill 组合策略

```javascript
// 设计师组合
task({
  category: "visual-engineering",
  load_skills: ["frontend-ui-ux", "playwright"],
  prompt: "..."
}); // → 实现美观 UI 并直接在浏览器验证

// 架构师组合
task({
  category: "ultrabrain",
  load_skills: [],
  prompt: "..."
}); // → 纯推理，深度系统架构分析

// 维护者组合
task({
  category: "quick",
  load_skills: ["git-master"],
  prompt: "..."
}); // → 快速修复代码并生成整洁提交
```

---

## 五、Hooks 系统

项目包含 **48 个生命周期钩子**，分为五层。

### 钩子分层架构

```
createHooks()
  ├─→ createCoreHooks()           # 39 个钩子
  │   ├─ createSessionHooks()     # 23 个
  │   ├─ createToolGuardHooks()   # 12 个
  │   └─ createTransformHooks()   # 4 个
  ├─→ createContinuationHooks()   # 7 个
  └─→ createSkillHooks()          # 2 个
```

### Session Hooks（23 个）

| 钩子 | 事件 | 功能 |
|------|------|------|
| contextWindowMonitor | session.idle | 追踪上下文窗口使用率 |
| preemptiveCompaction | session.idle | 在达到硬限制前触发压缩 |
| sessionRecovery | session.error | 自动恢复可恢复的会话错误 |
| sessionNotification | session.idle | 完成时发送操作系统通知 |
| thinkMode | chat.params | 模型变体切换（扩展思考） |
| anthropicContextWindowLimitRecovery | session.error | 多策略上下文恢复（截断、压缩） |
| autoUpdateChecker | session.created | 检查 npm 插件更新 |
| agentUsageReminder | chat.message | 提醒可用代理 |
| nonInteractiveEnv | chat.message | 为 run 命令调整行为 |
| interactiveBashSession | tool.execute | 为交互式工具提供 tmux 会话 |
| ralphLoop | event | 自引用开发循环 |
| editErrorRecovery | tool.execute.after | 重试失败的文件编辑 |
| delegateTaskRetry | tool.execute.after | 重试失败的委托任务 |
| startWork | chat.message | /start-work 命令处理器 |
| prometheusMdOnly | tool.execute.before | 强制 Prometheus 只写 .md 文件 |
| sisyphusJuniorNotepad | chat.message | 为子代理注入记事本 |
| questionLabelTruncator | tool.execute.before | 截断长问题标签 |
| taskResumeInfo | chat.message | 恢复任务时注入上下文 |
| anthropicEffort | chat.params | 调整推理努力级别 |
| modelFallback | chat.params | 提供商级别模型回退 |
| noSisyphusGpt | chat.message | 阻止 Sisyphus 使用 GPT 模型 |
| noHephaestusNonGpt | chat.message | 阻止 Hephaestus 使用非 GPT 模型 |
| runtimeFallback | event | API 提供商错误时自动切换模型 |

### Tool Guard Hooks（12 个）

| 钩子 | 事件 | 功能 |
|------|------|------|
| commentChecker | tool.execute.after | 阻止 AI 生成的注释模式 |
| toolOutputTruncator | tool.execute.after | 截断过大的工具输出 |
| directoryAgentsInjector | tool.execute.before | 将目录 AGENTS.md 注入上下文 |
| directoryReadmeInjector | tool.execute.before | 将目录 README.md 注入上下文 |
| emptyTaskResponseDetector | tool.execute.after | 检测空任务响应 |
| rulesInjector | tool.execute.before | 条件规则注入 |
| tasksTodowriteDisabler | tool.execute.before | 任务系统激活时禁用 TodoWrite |
| writeExistingFileGuard | tool.execute.before | 要求写入现有文件前先读取 |
| hashlineReadEnhancer | tool.execute.after | 增强读取输出，添加行哈希 |
| jsonErrorRecovery | tool.execute.after | 检测 JSON 解析错误，注入修正提醒 |

### Transform Hooks（4 个）

| 钩子 | 事件 | 功能 |
|------|------|------|
| claudeCodeHooks | messages.transform | Claude Code settings.json 兼容层 |
| keywordDetector | messages.transform | 检测 ultrawork/search/analyze 模式 |
| contextInjectorMessagesTransform | messages.transform | 将 AGENTS.md/README.md 注入上下文 |
| thinkingBlockValidator | messages.transform | 验证思考块结构 |

### Continuation Hooks（7 个）

| 钩子 | 事件 | 功能 |
|------|------|------|
| stopContinuationGuard | chat.message | /stop-continuation 命令处理 |
| compactionContextInjector | session.compacted | 压缩后重新注入上下文 |
| compactionTodoPreserver | session.compacted | 通过压缩保留 todos |
| todoContinuationEnforcer | session.idle | Boulder: 强制未完成 todos 时继续 |
| unstableAgentBabysitter | session.idle | 监控不稳定代理行为 |
| backgroundNotificationHook | event | 后台任务完成通知 |
| atlasHook | event | Boulder/后台会话的主编排器 |

### Skill Hooks（2 个）

| 钩子 | 事件 | 功能 |
|------|------|------|
| categorySkillReminder | chat.message | 提醒分类+技能委托 |
| autoSlashCommand | chat.message | 自动检测用户输入中的 /command |

### 复杂钩子详解

#### anthropic-context-window-limit-recovery

31 个文件，约 2232 行代码。最复杂的钩子。通过多种策略恢复上下文窗口限制错误。

**恢复策略（按优先级）**：

| 策略 | 机制 |
|------|------|
| Empty content recovery | 处理消息中的空/null 内容块 |
| Deduplication | 从上下文中移除重复的工具结果 |
| Target-token truncation | 将最大的工具输出截断到目标比例 |
| Aggressive truncation | 最后手段截断，最小化输出保留 |
| Summarize retry | 压缩 + 摘要后重试 |

**重试配置**：
- 最大尝试次数: 2
- 初始延迟: 2s，backoff ×2，最大 30s
- 最大截断尝试: 20
- 目标令牌比例: 0.5（截断到限制的 50%）

#### atlas (Boulder 编排器)

17 个文件，约 1976 行代码。Boulder 会话的主编排器。

**决策门控流程**：

```
session.idle 事件
  → 是 boulder/ralph/atlas 会话?
  → 有 abort 信号?
  → 失败次数 < 最大?
  → 没有运行中的后台任务?
  → 代理匹配预期?
  → 计划完成?
  → 冷却时间已过? (5s)
  → 注入继续提示
```

#### ralph-loop

14 个文件，约 1687 行代码。自引用开发循环。

**循环生命周期**：

```
/ralph-loop → startLoop(sessionID, prompt, options)
  → loopState.startLoop() → 持久化状态到 .sisyphus/ralph-loop.local.md
  → session.idle 事件 → createRalphLoopEventHandler()
    → completionPromiseDetector: 扫描输出中的 <promise>DONE</promise>
    → 未完成: 注入继续提示 → 循环
    → 完成或达到 maxIterations: cancelLoop()
```

#### todo-continuation-enforcer (Boulder 机制)

14 个文件，约 2061 行代码。"Boulder" 机制：强制 Sisyphus 在 todos 未完成时继续工作。

**工作机制**：

```
session.idle
  → 是主会话（非 prometheus/compaction）?
  → 最近没有检测到 abort?
  → Todos 仍然未完成?
  → 没有运行中的后台任务?
  → 冷却时间已过? (30s)
  → 失败次数 < 最大? (5)
  → 开始 2 秒倒计时 toast → 注入 CONTINUATION_PROMPT
```

**常数**：

```typescript
DEFAULT_SKIP_AGENTS = ["prometheus", "compaction", "plan"]
CONTINUATION_COOLDOWN_MS = 30_000     // 30s 注入间隔
MAX_CONSECUTIVE_FAILURES = 5          // 然后 5 分钟暂停
FAILURE_RESET_WINDOW_MS = 5 * 60_000  // 5 分钟失败重置窗口
COUNTDOWN_SECONDS = 2
ABORT_WINDOW_MS = 3000                // abort 信号后的宽限期
```

---

## 六、Tools 系统

项目包含 **26 个工具**，分为多个类别。

### 任务管理工具（4 个）

| 工具 | 工厂 | 参数 |
|------|------|------|
| task_create | createTaskCreateTool | subject, description, blockedBy, blocks, metadata, parentID |
| task_list | createTaskList | 无 |
| task_get | createTaskGetTool | id |
| task_update | createTaskUpdateTool | id, subject, description, status, addBlocks, addBlockedBy, owner, metadata |

### 委托工具（1 个）

| 工具 | 工厂 | 参数 |
|------|------|------|
| task | createDelegateTask | description, prompt, category, subagent_type, run_in_background, session_id, load_skills, command |

### 代理调用工具（1 个）

| 工具 | 工厂 | 参数 |
|------|------|------|
| call_omo_agent | createCallOmoAgent | description, prompt, subagent_type, run_in_background, session_id |

**与 delegate-task 的区别**：

| 方面 | call_omo_agent | delegate-task (task) |
|------|----------------|----------------------|
| 代理选择 | 命名代理 (explore/librarian) | 分类或 subagent_type |
| 技能加载 | 无 | load_skills[] 支持 |
| 模型选择 | 从代理的 fallback 链 | 从分类配置 |
| 用例 | 快速上下文 grep | 带技能的完整委托 |

### 后台任务工具（2 个）

| 工具 | 工厂 | 参数 |
|------|------|------|
| background_output | createBackgroundOutput | task_id, block, timeout, full_session, include_thinking, message_limit |
| background_cancel | createBackgroundCancel | taskId, all |

**使用模式**：

```javascript
background_output(task_id, block=false)  // 检查当前状态/结果
background_output(task_id, block=true)   // 等待完成（默认超时 120s）
background_output(task_id, full_session=true) // 返回完整会话记录
background_output(task_id, message_limit=N)   // 仅最后 N 条消息
background_output(task_id, include_thinking=true) // 包含思考块
```

### LSP 重构工具（6 个）

| 工具 | 参数 |
|------|------|
| lsp_goto_definition | filePath, line, character |
| lsp_find_references | filePath, line, character, includeDeclaration |
| lsp_symbols | filePath, scope (document/workspace), query, limit |
| lsp_diagnostics | filePath, severity |
| lsp_prepare_rename | filePath, line, character |
| lsp_rename | filePath, line, character, newName |

### 代码搜索工具（4 个）

| 工具 | 工厂 | 参数 |
|------|------|------|
| ast_grep_search | createAstGrepTools | pattern, lang, paths, globs, context |
| ast_grep_replace | createAstGrepTools | pattern, rewrite, lang, paths, globs, dryRun |
| grep | createGrepTools | pattern, path, include (60s 超时, 10MB 限制) |
| glob | createGlobTools | pattern, path (60s 超时, 100 文件限制) |

### 会话历史工具（4 个）

| 工具 | 工厂 | 参数 |
|------|------|------|
| session_list | createSessionManagerTools | 无 |
| session_read | createSessionManagerTools | session_id, include_todos, limit |
| session_search | createSessionManagerTools | query, session_id, case_sensitive, limit |
| session_info | createSessionManagerTools | session_id |

### 技能/命令工具（2 个）

| 工具 | 工厂 | 参数 |
|------|------|------|
| skill | createSkillTool | name, user_message |
| skill_mcp | createSkillMcpTool | mcp_name, tool_name/resource_name/prompt_name, arguments, grep |

### 系统工具（2 个）

| 工具 | 工厂 | 参数 |
|------|------|------|
| interactive_bash | Direct | tmux_command |
| look_at | createLookAt | file_path, image_data, goal |

### 编辑工具（1 个，条件性）

| 工具 | 工厂 | 参数 |
|------|------|------|
| hashline_edit | createHashlineEditTool | file, edits[] |

---

## 七、Features 模块

项目包含 **19 个功能模块**，按复杂度分类。

### 高复杂度模块

| 模块 | 文件数 | 复杂度 | 功能 |
|------|--------|--------|------|
| opencode-skill-loader | 33 | HIGH | 4 作用域 YAML frontmatter 技能加载 |
| background-agent | 31 | HIGH | 任务生命周期、并发控制（5/模型）、轮询 |
| tmux-subagent | 30 | HIGH | Tmux 面板管理、网格规划、会话编排 |
| mcp-oauth | 18 | HIGH | OAuth 2.0 + PKCE + DCR 用于 MCP 服务器 |

### 中等复杂度模块

| 模块 | 文件数 | 复杂度 | 功能 |
|------|--------|--------|------|
| skill-mcp-manager | 12 | MEDIUM | 每会话 MCP 客户端生命周期 |
| claude-code-plugin-loader | 10 | MEDIUM | 从 .opencode/plugins/ 统一插件发现 |
| claude-tasks | 7 | MEDIUM | 任务 schema + 文件存储 + OpenCode todo 同步 |
| claude-code-mcp-loader | 6 | MEDIUM | 带有 ${VAR} 环境变量展开的 .mcp.json 加载 |
| context-injector | 6 | MEDIUM | AGENTS.md/README.md 注入上下文 |

### 低复杂度模块

| 模块 | 文件数 | 复杂度 | 功能 |
|------|--------|--------|------|
| builtin-skills | 17 | LOW | 6 个内置技能 |
| builtin-commands | 11 | LOW | 命令模板 |
| run-continuation-state | 5 | LOW | run 命令的持久状态 |
| boulder-state | 5 | LOW | 多步操作的持久状态 |
| tool-metadata-store | 3 | LOW | 工具执行元数据缓存 |

---

## 八、MCP 系统

### 三层架构

| 层级 | 来源 | 机制 |
|------|------|------|
| 内置 MCP | src/mcp/ | 3 个远程 HTTP: websearch (Exa/Tavily), context7, grep_app |
| Claude Code MCP | .mcp.json | 通过 claude-code-mcp-loader 进行 ${VAR} 环境变量展开 |
| 技能嵌入 MCP | SKILL.md YAML | 由 SkillMcpManager 管理 (stdio + HTTP) |

### 内置 MCP 服务器

| 名称 | URL | 环境变量 | 工具 |
|------|-----|----------|------|
| websearch | mcp.exa.ai 或 mcp.tavily.com | EXA_API_KEY, TAVILY_API_KEY | 网页搜索 |
| context7 | mcp.context7.com/mcp | CONTEXT7_API_KEY | 库文档 |
| grep_app | mcp.grep.app | 无 | GitHub 代码搜索 |

### 启用/禁用配置

```jsonc
// 方法 1: disabled_mcps 数组
{ "disabled_mcps": ["websearch", "context7"] }

// 方法 2: enabled 标志
{ "mcp": { "websearch": { "enabled": false } } }
```

---

## 九、CLI 命令系统

### 5 个命令

| 命令 | 功能 |
|------|------|
| install | 交互式/非交互式安装设置 |
| run \<message\> | 非交互式会话启动器 |
| doctor | 4 类健康检查 |
| get-local-version | 版本检测 |
| mcp-oauth | OAuth 令牌管理 |

### Doctor 检查类别

| 类别 | 检查项 |
|------|--------|
| System | 二进制文件、插件注册、版本匹配 |
| Config | JSONC 有效性、Zod schema |
| Tools | AST-Grep、LSP 服务器、GH CLI、MCP 服务器 |
| Models | 缓存存在、模型解析、代理/分类覆盖 |

### CLI 命令详解

```bash
bunx oh-my-opencode install                    # 交互式设置
bunx oh-my-opencode run "fix the bug"          # 非交互式会话
bunx oh-my-opencode doctor                     # 健康诊断
bunx oh-my-opencode mcp-oauth login <url>      # OAuth 登录
bunx oh-my-opencode mcp-oauth logout <url>     # OAuth 登出
bunx oh-my-opencode mcp-oauth status           # 查看令牌状态
```

### Agent 解析优先级

```
1. --agent CLI 标志
2. OPENCODE_AGENT 环境变量
3. default_run_agent 配置
4. "sisyphus" (默认)
```

---

## 十、配置系统

### 配置加载优先级

```
用户配置 (~/.config/opencode/oh-my-opencode.jsonc)
    ↓ 深度合并
项目配置 (.opencode/oh-my-opencode.jsonc)
    ↓ Zod 默认值
最终配置
```

**合并规则**：
- `agents`, `categories`, `claude_code`: 递归深度合并
- `disabled_*` 数组: 集合并集

### 根 Schema 字段（28 个）

```
$schema
new_task_system_enabled
default_run_agent
disabled_mcps
disabled_agents
disabled_skills
disabled_hooks
disabled_commands
disabled_tools
hashline_edit
agents
categories
claude_code
sisyphus_agent
comment_checker
experimental
auto_update
skills
ralph_loop
background_task
notification
babysitting
git_master
browser_automation_engine
websearch
tmux
sisyphus
start_work
_migrations
```

### Agent Override 字段（21 个）

```
model
variant
category
skills
temperature
top_p
prompt
prompt_append
tools
disable
description
mode
color
permission
maxTokens
thinking
reasoningEffort
textVerbosity
providerOptions
```

---

## 十一、Skills 技能系统

### 内置技能（6 个）

| 技能 | 功能 |
|------|------|
| git-master | Git 专家：原子提交、变基手术、历史考古 |
| playwright | 通过 Playwright MCP 进行浏览器自动化 |
| playwright-cli | 通过 Playwright CLI 进行浏览器自动化 |
| agent-browser | 通过 agent-browser CLI 进行浏览器自动化 |
| dev-browser | 有状态浏览器脚本 |
| frontend-ui-ux | 设计师风格 UI/UX 实现 |

### 自定义技能格式

```markdown
---
name: my-skill
description: What this skill does
tools: [Bash, Read, Write]
mcp:
  - name: my-mcp
    type: stdio
    command: npx
    args: [-y, my-mcp-server]
---

Skill content (instructions for the agent)...
```

### 技能加载位置（优先级从高到低）

1. `.opencode/skills/*/SKILL.md`（项目，OpenCode 原生）
2. `~/.config/opencode/skills/*/SKILL.md`（用户，OpenCode 原生）
3. `.claude/skills/*/SKILL.md`（项目，Claude Code 兼容）
4. `.agents/skills/*/SKILL.md`（项目，Agents 约定）
5. `~/.agents/skills/*/SKILL.md`（用户，Agents 约定）

### 4 作用域优先级

```
1. Project (.opencode/skills/)     # 最高优先级
2. OpenCode config (~/.config/opencode/skills/)
3. User (~/.config/opencode/oh-my-opencode/skills/)
4. Global (built-in skills)        # 最低优先级
```

---

## 十二、Commands 命令系统

### 内置命令（8 个）

| 命令 | 功能 |
|------|------|
| /init-deep | 初始化分层 AGENTS.md 知识库 |
| /ralph-loop | 启动自引用开发循环直到完成 |
| /ulw-loop | 启动 ultrawork 循环 |
| /cancel-ralph | 取消活动的 Ralph Loop |
| /refactor | 智能 LSP + AST-grep 重构 |
| /start-work | 从 Prometheus 计划开始 Sisyphus 工作会话 |
| /stop-continuation | 停止所有继续机制 |
| /handoff | 创建详细上下文摘要以在新会话中继续工作 |

---

## 十三、核心工作流程

### 1. Ultrawork 模式

```
用户输入: "ulw" 或 "ultrawork"
    ↓
[keyword-detector] 检测 ultrawork 模式
    ↓
注入 ultrawork 系统提示
    ↓
[Sisyphus] 激活所有代理
    ↓
并行执行直到完成
```

### 2. Prometheus 规划模式

```
用户: "我想要重构认证系统"
    ↓
[Prometheus] 进入面试模式
    ↓
研究代码库 → 提出澄清问题
    ↓
[Metis] 差距分析（发现遗漏）
    ↓
[Momus] 计划审查（可选，高精度模式）
    ↓
生成计划: .sisyphus/plans/{name}.md
    ↓
用户: /start-work
    ↓
[Atlas] 执行计划
```

### 3. Ralph Loop 自引用循环

```
/ralph-loop "构建 REST API"
    ↓
持久化状态到 .sisyphus/ralph-loop.local.md
    ↓
循环:
  执行 → 检测 <promise>DONE</promise>
  ↓ 未完成
  注入继续提示 → 继续执行
    ↓ 完成
取消循环
```

### 4. Boulder 继续机制

```
session.idle 事件
    ↓
[todo-continuation-enforcer]
    ↓
未完成 todos 存在?
    ↓ YES
2 秒倒计时 toast → 注入 CONTINUATION_PROMPT
```

---

## 十四、Hashline 编辑工具

### 核心问题

传统编辑工具依赖模型精确复制行内容，失败率高。

### 解决方案：LINE#ID 内容哈希

**读取输出**：
```
11#VK| function hello() {
22#XJ|   return "world";
33#MB| }
```

**编辑请求**：
```json
{ "op": "replace", "pos": "11#VK", "lines": "function hello(name: string) {" }
```

如果文件自读取后已更改，哈希不匹配，编辑在损坏前被拒绝。

### 三操作模型

| Op | pos | end | lines | 效果 |
|----|-----|-----|-------|------|
| replace | 必需 | 可选 | 必需 | 替换单行或范围 |
| append | 可选 | 可选 | 必需 | 在锚点后插入（或 EOF） |
| prepend | 可选 | 可选 | 必需 | 在锚点前插入（或 BOF） |

`lines: null` 或 `lines: []` 配合 `replace` = 删除

### 执行流水线

```
hashline-edit-executor.ts
  → normalize-edits.ts       # 解析 RawHashlineEdit → HashlineEdit
  → validation.ts            # 验证 LINE#ID 引用（哈希匹配）
  → edit-ordering.ts         # 从下到上排序（按行号降序）
  → edit-deduplication.ts    # 移除重复操作
  → edit-operations.ts       # 应用每个操作
  → autocorrect-replacement-lines.ts  # 自动修复缩进/格式
  → hashline-edit-diff.ts    # 生成 diff 输出
```

### 哈希计算

- Hash alphabet: `ZPMQVRWSNKTXJBYH` (16 个 CID 字母)
- 分隔符: `#` (井号)
- 内容分隔符: `|` (竖线)
- 示例: `42#VK` 表示第 42 行，哈希为 `VK`

### 效果

Grok Code Fast 1 成功率从 **6.7% → 68.3%**

---

## 十五、后台代理引擎

### 任务生命周期

```
LaunchInput → pending → [ConcurrencyManager queue] → running → polling → completed/error/cancelled/interrupt
```

### 并发模型

- **Key 格式**: `{providerID}/{modelID}` (例如 `anthropic/claude-opus-4-6`)
- **默认限制**: 每个键 5 个并发（可通过 `background_task` 配置）
- **FIFO 队列**: 槽位满时任务按顺序等待
- **槽位释放**: 完成、错误、取消时

### 完成检测

两个信号组合：

1. **Session idle 事件** - OpenCode 报告会话变为空闲
2. **稳定性检测** - 消息数 10 秒不变（3+ 次稳定轮询，间隔 3s）

两者必须同时满足才标记任务完成，防止过早完成检测。

### 通知流程

```
task completed → result-handler → parent-session-notifier → 注入系统消息到父会话
```

---

## 十六、Tmux 集成

### 功能

- 后台代理在独立 tmux 面板中运行
- 实时观察多个代理工作
- 自动清理完成的代理

### 架构

```
TmuxSessionManager (manager.ts)
  ├─→ DecisionEngine: 是否应该 spawn/close 面板?
  ├─→ ActionExecutor: 执行 spawn/close/replace 操作
  ├─→ PollingManager: 监控面板健康
  └─→ EventHandlers: 响应会话 create/delete
```

### 布局约束

- `MIN_PANE_WIDTH`: 52 字符
- `MIN_PANE_HEIGHT`: 11 行
- 主面板受保护（不会拆分到最小以下）
- 代理面板从剩余空间拆分

### 面板生命周期

```
session.created → spawn-action-decider → grid-planning → action-executor → 跟踪会话
session.deleted → 清理跟踪的会话 → 如果空则关闭面板
```

### 配置

```jsonc
{
  "tmux": {
    "enabled": true,
    "layout": "main-vertical",
    "main_pane_size": 60,
    "main_pane_min_width": 120,
    "agent_pane_min_width": 40
  }
}
```

---

## 十七、模型解析流水线

### 5 步解析流程

```
resolveModel(input)
  1. Override: UI 选择的模型（仅 primary agents）
  2. Category default: 从分类配置继承
  3. 用户 fallback_models: 用户配置的 fallback 列表
  4. 提供商 fallback 链: OmO 源码中的内置链
  5. 系统默认: OpenCode 配置的默认模型
```

### 模型设置兼容性

规范化字段（目标模型不支持时自动处理）：

| 字段 | 处理方式 |
|------|----------|
| variant | 降级到最接近的支持值 |
| reasoningEffort | 降级或移除 |
| temperature | 不支持时移除 |
| top_p | 不支持时移除 |
| maxTokens | 限制到模型报告的最大输出限制 |
| thinking | 目标模型不支持时移除 |

### 模型可用性检查

- 模糊匹配模型名称
- 缓存可用性结果
- 自动 fallback 到可用模型

---

## 十八、会话恢复系统

### 恢复策略

| 错误类型 | 恢复动作 |
|----------|----------|
| tool_result_missing | 从存储重建缺失的工具结果 |
| thinking_block_order | 重新排序畸形思考块 |
| thinking_disabled_violation | 禁用时移除思考块 |
| empty_content_message | 处理空/空内容块 |
| 上下文窗口限制 | 多策略恢复（截断、压缩、摘要） |

### 关键组件

| 文件 | 功能 |
|------|------|
| hook.ts | createSessionRecoveryHook() - 错误检测、策略调度、恢复 |
| detect-error-type.ts | detectErrorType(error) → RecoveryErrorType \| null |
| resume.ts | resumeSession() - 重建会话上下文，触发重试 |
| storage.ts | 每会话消息存储用于恢复重建 |

### 存储子目录

```
storage/
  ├── message-store.ts    # 内存 + 文件消息缓存
  ├── part-store.ts       # 单个消息部分存储
  └── index.ts            # 桶导出
```

---

## 十九、OAuth 2.0 + PKCE 系统

### 授权流程

```
1. discovery.ts → 获取 /.well-known/oauth-authorization-server
2. dcr.ts → 动态客户端注册（如果服务器支持）
3. oauth-authorization-flow.ts → 生成 PKCE verifier/challenge
4. callback-server.ts → 本地 HTTP 服务器监听重定向
5. 打开浏览器 → 授权 URL
6. callback-server.ts → 接收 code + state
7. provider.ts → 用 PKCE verifier 交换令牌
8. storage.ts → 持久化令牌到 ~/.config/opencode/mcp-oauth/
```

### PKCE 实现

- **Code verifier**: 32 随机字节 → base64url（无填充）
- **Code challenge**: SHA-256(verifier) → base64url
- **Method**: S256

### 令牌存储

位置: `~/.config/opencode/mcp-oauth/` - 每个 MCP 服务器一个 JSON 文件（按键为服务器 URL 哈希）

字段: `access_token`, `refresh_token`, `expires_at`, `client_id`

### CLI 命令

```bash
bunx oh-my-opencode mcp-oauth login <server-url>   # 完整 PKCE 流程
bunx oh-my-opencode mcp-oauth logout <server-url>  # 撤销 + 删除令牌
bunx oh-my-opencode mcp-oauth status               # 列出存储的令牌
```

---

## 二十、上下文注入系统

### 注入内容

- **AGENTS.md** - 目录级别的项目知识
- **README.md** - 项目说明
- **条件规则** - 基于配置的条件注入

### 距离优先级

1. 与目标文件同目录
2. 向上查找父目录直到项目根
3. 项目根本身

同距离时全部注入，每会话去重防止重复注入。

### 关键文件

| 文件 | 功能 |
|------|------|
| hook.ts | createRulesInjectorHook() - 连接缓存 + 注入器 |
| injector.ts | createRuleInjectionProcessor() - 编排查找 → 缓存 → 注入 |
| finder.ts | findRuleFiles() + calculateDistance() |
| cache.ts | createSessionCacheStore() - 每会话注入去重 |

### 截断

使用 `DynamicTruncator` - 根据模型上下文窗口自适应注入大小（1M 上下文模型获得完整内容，较小模型获得截断摘要）。

---

## 二十一、迁移系统

自动在配置加载时转换遗留配置。

### 迁移类型

| 文件 | 功能 |
|------|------|
| agent-names.ts | 旧代理名 → 新名 (例如 `junior` → `sisyphus-junior`) |
| hook-names.ts | 旧钩子名 → 新名 |
| model-versions.ts | 旧模型 ID → 当前版本 |
| agent-category.ts | 遗留代理配置 → 分类系统 |

### 自动迁移

`migrateConfigFile()` 在加载时自动运行，无需用户干预。

---

## 附录

### 项目结构概览

```
oh-my-opencode/
├── src/
│   ├── index.ts              # 插件入口
│   ├── plugin-config.ts      # JSONC 多级配置
│   ├── agents/               # 11 个代理
│   ├── hooks/                # 48 个生命周期钩子
│   ├── tools/                # 26 个工具
│   ├── features/             # 19 个功能模块
│   ├── shared/               # 95+ 工具文件
│   ├── config/               # Zod v4 schema 系统
│   ├── cli/                  # CLI 命令
│   ├── mcp/                  # 3 个内置远程 MCP
│   ├── plugin/               # 8 个 OpenCode 钩子处理器
│   └── plugin-handlers/      # 6 阶段配置加载流水线
├── packages/                 # Monorepo: cli-runner, 12 个平台二进制
└── docs/                     # 文档
```

### 约定

- **运行时**: 仅 Bun — 从不使用 npm/yarn
- **TypeScript**: strict 模式, ESNext, bundler moduleResolution
- **测试模式**: Bun test, 同目录 `*.test.ts`, given/when/then 风格
- **工厂模式**: 所有工具、钩子、代理使用 `createXXX()` 工厂
- **配置格式**: JSONC 带注释, Zod v4 验证, snake_case 键
- **文件命名**: 所有文件/目录使用 kebab-case
- **模块结构**: index.ts 桶导出，无 catch-all 文件

### 反模式

- 永不使用 `as any`, `@ts-ignore`, `@ts-expect-error`
- 永不抑制 lint/type 错误
- 除非用户明确要求，否则不向代码/注释添加表情符号
- 未经明确请求永不提交
- 永不直接运行 `bun publish` — 使用 GitHub Actions
- 测试: given/when/then — 永不使用 Arrange-Act-Assert 注释
- 永不创建 catch-all 文件 (`utils.ts`, `helpers.ts`, `service.ts`)
- 空 catch 块 `catch(e) {}` — 必须处理错误
- index.ts 仅作为入口点 — 永不倾倒业务逻辑

---

*文档生成时间: 2026-03-28*
