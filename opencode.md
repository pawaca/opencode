# OpenCode 项目工具与 System Prompt 设计调研报告

## 一、项目概述

OpenCode 是一个基于 AI 的代码助手 CLI 工具，采用 Monorepo 架构，使用 TypeScript + Bun 技术栈开发。项目支持多种 LLM 提供商（OpenAI、Anthropic、Google、Azure 等），核心设计围绕 **Agent → Session → Tool → LLM** 执行链展开。

### 核心目录结构

```
packages/opencode/src/
├── tool/           # 工具定义（38+ 个内置工具）
├── session/        # 会话管理 & System Prompt
│   └── prompt/     # 不同模型的 Prompt 模板
├── agent/          # Agent 定义与管理
│   └── prompt/     # Agent 专用 Prompt
├── permission/     # 权限控制系统
├── skill/          # Skill 扩展系统
└── config/         # 配置管理
```

---

## 二、工具设计哲学与规划

### 2.1 工具分类体系

OpenCode 的工具按功能可分为以下几类：

| 类别 | 工具 | 核心职责 |
|------|------|----------|
| **文件操作** | `read`, `write`, `edit`, `glob`, `ls` | 文件读写、搜索、目录列表 |
| **代码搜索** | `grep`, `codesearch` | 内容搜索、API/SDK 文档查询 |
| **命令执行** | `bash` | Shell 命令执行 |
| **网络获取** | `webfetch`, `websearch` | 网页内容抓取、搜索引擎查询 |
| **任务管理** | `todowrite`, `todoread` | 任务列表管理与进度追踪 |
| **子任务委派** | `task` | 启动子 Agent 执行复杂任务 |
| **交互确认** | `question` | 向用户提问确认 |
| **代码编辑** | `apply_patch`, `multiedit` | 批量/补丁式代码修改 |
| **扩展能力** | `skill` | 加载领域特定指令集 |
| **规划模式** | `plan` | 任务规划与方案设计 |

### 2.2 为什么需要这些工具

#### 核心设计原则

1. **专用工具优于通用命令**
   - 不用 `cat` 读文件，用专门的 `Read` 工具
   - 不用 `grep` 命令，用专门的 `Grep` 工具
   - 原因：专用工具可以提供更好的错误处理、权限控制、输出格式化

2. **读写分离，编辑优先**
   - `Read` 只读取，`Write` 创建新文件，`Edit` 修改现有文件
   - 强制要求：编辑前必须先读取（防止盲目修改）

3. **搜索能力分层**
   - `Glob`: 文件名模式匹配（快速定位文件）
   - `Grep`: 文件内容搜索（精确查找代码）
   - `CodeSearch`: 外部 API/文档搜索（获取最新知识）

4. **任务管理可视化**
   - `TodoWrite` 让用户能看到 AI 的工作进度
   - 强制只能同时有一个 `in_progress` 任务

5. **子任务并行化**
   - `Task` 工具可以启动专用子 Agent
   - 支持 `explore`、`general` 等不同类型的子 Agent

---

## 三、核心工具详解

### 3.1 文件读取工具 (Read)

**设计要点**：
- 支持行号偏移和限制（处理大文件）
- 支持多种文件类型（代码、图片、PDF、Jupyter Notebook）
- 返回带行号的内容（便于后续编辑定位）

```typescript
// 参数设计
{
  file_path: string,      // 必须是绝对路径
  offset?: number,        // 起始行号
  limit?: number          // 读取行数
}

// 返回格式：cat -n 风格
"     1→import { Tool } from './tool'"
```

**描述文件设计特点**：
```
- 明确说明路径必须是绝对路径
- 说明默认行为（2000行起始）
- 列举支持的文件类型
- 强调多文件并行读取的最佳实践
```

### 3.2 文件编辑工具 (Edit)

**设计要点**：
- 基于字符串替换而非行号（更健壮）
- 支持多种容错匹配策略（空白、缩进、转义字符）
- 自动检测 LSP 错误并反馈
- 生成 diff 用于权限确认

```typescript
// 参数设计
{
  filePath: string,       // 文件绝对路径
  oldString: string,      // 要替换的文本
  newString: string,      // 替换后的文本
  replaceAll?: boolean    // 是否全局替换
}

// 返回值
{
  output: "Edit applied successfully.",
  metadata: {
    diff: string,         // 变更差异
    filediff: {...},      // 添加/删除行数统计
    diagnostics: {...}    // LSP 错误信息
  }
}
```

**核心容错机制**（按优先级）：
1. `SimpleReplacer` - 精确匹配
2. `LineTrimmedReplacer` - 忽略行首尾空白
3. `BlockAnchorReplacer` - 首尾行锚定 + 中间模糊匹配
4. `WhitespaceNormalizedReplacer` - 空白标准化
5. `IndentationFlexibleReplacer` - 缩进弹性匹配
6. `EscapeNormalizedReplacer` - 转义字符处理
7. `TrimmedBoundaryReplacer` - 边界修剪
8. `ContextAwareReplacer` - 上下文感知

### 3.3 文件写入工具 (Write)

**描述设计特点**（从 write.txt 提取）：
```
Writes a file to the local filesystem.

Usage:
- This tool will overwrite the existing file if there is one at the provided path.
- If this is an existing file, you MUST use the Read tool first to read the file's contents.
  This tool will fail if you did not read the file first.
- ALWAYS prefer editing existing files in the codebase. NEVER write new files unless explicitly required.
- NEVER proactively create documentation files (*.md) or README files.
  Only create documentation files if explicitly requested by the User.
- Only use emojis if the user explicitly requests it. Avoid writing emojis to files unless asked.
```

**设计启示**：
- 明确前置条件（必须先读取）
- 明确禁止行为（不要主动创建文档）
- 防止过度行为（不要加 emoji）

### 3.4 代码搜索工具 (Grep)

**参数设计**：
```typescript
{
  pattern: string,              // 正则表达式
  path?: string,                // 搜索目录
  glob?: string,                // 文件过滤 (*.js)
  type?: string,                // 文件类型 (js, py)
  output_mode?: "content" | "files_with_matches" | "count",
  "-A"?: number,                // 后文行数
  "-B"?: number,                // 前文行数
  "-C"?: number,                // 上下文行数
  "-i"?: boolean,               // 忽略大小写
  "-n"?: boolean,               // 显示行号
  multiline?: boolean,          // 多行匹配
  head_limit?: number,          // 限制结果数
  offset?: number               // 跳过前N个结果
}
```

**描述设计特点**：
- 明确说明是基于 ripgrep（不是 grep）
- 提供正则语法提示（大括号需转义）
- 说明默认输出模式
- 提供使用示例

### 3.5 任务管理工具 (TodoWrite)

**描述设计特点**（精华提取）：

```
## When to Use This Tool
1. Complex multistep tasks - When a task requires 3 or more distinct steps
2. Non-trivial and complex tasks - Tasks that require careful planning
3. User explicitly requests todo list
4. User provides multiple tasks (numbered or comma-separated)
5. After receiving new instructions - Immediately capture as todos
6. After completing a task - Mark complete and add follow-up tasks
7. When starting a new task - Mark as in_progress (only ONE at a time)

## When NOT to Use This Tool
1. Single, straightforward task
2. Trivial task with no organizational benefit
3. Task can be completed in less than 3 trivial steps
4. Purely conversational or informational request
```

**示例驱动的描述**：
- 提供正面示例（何时使用）
- 提供反面示例（何时不用）
- 每个示例都有 `<reasoning>` 解释为什么

**参数设计**：
```typescript
{
  todos: Array<{
    content: string,        // 任务内容
    status: "pending" | "in_progress" | "completed" | "cancelled"
  }>
}
```

### 3.6 子任务委派工具 (Task)

**设计要点**：
- 支持多种子 Agent 类型
- 可以后台运行
- 支持恢复（resume）之前的 Agent

**子 Agent 类型**：
| 类型 | 用途 | 可用工具 |
|------|------|----------|
| `explore` | 代码探索 | grep, glob, read, bash |
| `general` | 通用任务 | 所有工具（禁用 todo） |
| `plan` | 方案设计 | 只读工具 |

### 3.7 补丁应用工具 (ApplyPatch)

**设计要点**：
- 专为 GPT 系列模型设计的编辑格式
- 支持文件创建、删除、更新、重命名
- 类似 diff 但更简化

**格式规范**：
```
*** Begin Patch
*** Add File: hello.txt
+Hello world
*** Update File: src/app.py
*** Move to: src/main.py
@@ def greet():
-print("Hi")
+print("Hello, world!")
*** Delete File: obsolete.txt
*** End Patch
```

### 3.8 Skill 加载工具

**设计要点**：
- 动态生成工具描述（基于可用 skill 列表）
- 输出 XML 格式的技能内容
- 提供技能目录下的文件列表

```typescript
// 动态描述生成
description = [
  "Load a specialized skill...",
  "<available_skills>",
  ...skills.map(skill => `<skill><name>${skill.name}</name>...</skill>`),
  "</available_skills>"
].join("\n")
```

---

## 四、System Prompt 设计

### 4.1 Prompt 分层架构

```
┌─────────────────────────────────────┐
│     基础指令 (codex_header.txt)      │  ← 所有模型共享
├─────────────────────────────────────┤
│   模型特定 Prompt (anthropic.txt)    │  ← 按模型选择
├─────────────────────────────────────┤
│        环境信息 (动态生成)            │  ← 运行时注入
├─────────────────────────────────────┤
│      Agent 特定 Prompt               │  ← 按 Agent 类型
└─────────────────────────────────────┘
```

### 4.2 模型特定 Prompt 选择逻辑

```typescript
export function provider(model: Provider.Model) {
  if (model.api.id.includes("gpt-5")) return [PROMPT_CODEX]
  if (model.api.id.includes("gpt-") || model.api.id.includes("o1") || model.api.id.includes("o3"))
    return [PROMPT_BEAST]
  if (model.api.id.includes("gemini-")) return [PROMPT_GEMINI]
  if (model.api.id.includes("claude")) return [PROMPT_ANTHROPIC]
  return [PROMPT_ANTHROPIC_WITHOUT_TODO]  // 默认（不带 Todo 功能）
}
```

### 4.3 GPT 系列 Prompt 设计特点 (beast.txt)

**核心指导原则**：

1. **强调自主完成**
```
You are opencode, an agent - please keep going until the user's query is
completely resolved, before ending your turn and yielding back to the user.
```

2. **强制网络研究**
```
THE PROBLEM CAN NOT BE SOLVED WITHOUT EXTENSIVE INTERNET RESEARCH.
You must use the webfetch tool to recursively gather all information...
```

3. **禁止中途停止**
```
NEVER end your turn without having truly and completely solved the problem,
and when you say you are going to make a tool call, make sure you ACTUALLY
make the tool call, instead of ending your turn.
```

4. **标准化工作流程**
```
# Workflow
1. Fetch any URL's provided by the user
2. Understand the problem deeply
3. Investigate the codebase
4. Research the problem on the internet
5. Develop a clear, step-by-step plan
6. Implement the fix incrementally
7. Debug as needed
8. Test frequently
9. Iterate until the root cause is fixed
10. Reflect and validate comprehensively
```

5. **沟通风格指导**
```
Always communicate clearly and concisely in a casual, friendly yet professional tone.
<examples>
"Let me fetch the URL you provided to gather more information."
"Ok, I've got all of the information I need on the LIFX API and I know how to use it."
"Whelp - I see we have some problems. Let's fix those up."
</examples>
```

### 4.4 环境信息注入

```typescript
export async function environment(model: Provider.Model) {
  return [
    `You are powered by the model named ${model.api.id}.`,
    `<env>`,
    `  Working directory: ${Instance.directory}`,
    `  Is directory a git repo: ${project.vcs === "git" ? "yes" : "no"}`,
    `  Platform: ${process.platform}`,
    `  Today's date: ${new Date().toDateString()}`,
    `</env>`,
  ].join("\n")
}
```

### 4.5 工具使用指引设计模式

**模式一：明确的使用/不使用场景**
```
## When to Use This Tool
1. ...
2. ...

## When NOT to Use This Tool
1. ...
2. ...
```

**模式二：示例驱动 + 推理解释**
```xml
<example>
User: Help me rename the function getCwd to getCurrentWorkingDirectory
Assistant: Let me search for all occurrences...

<reasoning>
The assistant used the todo list because:
1. First, the assistant searched to understand the scope
2. Upon finding multiple occurrences, it determined this was complex
</reasoning>
</example>
```

**模式三：参数说明带默认值**
```
- output_mode: "content" shows matching lines, "files_with_matches" shows
  file paths (default), "count" shows match counts
```

**模式四：工具替代指引**
```
Avoid using Bash with `find`, `grep`, `cat` commands. Instead:
- File search: Use Glob (NOT find or ls)
- Content search: Use Grep (NOT grep or rg)
- Read files: Use Read (NOT cat/head/tail)
```

---

## 五、Agent 系统设计

### 5.1 Agent 类型

| Agent | 模式 | 描述 | 权限特点 |
|-------|------|------|----------|
| **build** | primary | 默认 Agent，执行所有工具 | 允许所有操作 |
| **plan** | primary | 计划模式，禁用编辑 | 仅允许读取和规划 |
| **general** | subagent | 通用并行执行 | 禁用 Todo 工具 |
| **explore** | subagent | 代码探索 | 仅搜索和读取 |
| **compaction** | primary (hidden) | 消息压缩 | 拒绝所有操作 |
| **title** | primary (hidden) | 标题生成 | 拒绝所有操作 |
| **summary** | primary (hidden) | 摘要生成 | 拒绝所有操作 |

### 5.2 Agent 定义结构

```typescript
interface AgentInfo {
  name: string,
  description?: string,
  mode: "subagent" | "primary" | "all",
  native?: boolean,          // 是否内置
  hidden?: boolean,          // 是否对用户隐藏
  topP?: number,
  temperature?: number,
  color?: string,
  permission: PermissionRuleset,
  model?: { modelID: string, providerID: string },
  variant?: string,
  prompt?: string,           // Agent 特定 prompt
  options: Record<string, any>,
  steps?: number             // 最大步数限制
}
```

### 5.3 权限系统设计

```typescript
// 权限类型
type Action = "allow" | "ask" | "deny"

// 权限范围示例
{
  "*": "allow",              // 全局默认
  "doom_loop": "ask",        // 死循环检测
  "external_directory": {    // 外部目录访问
    "*": "ask",
    "/tmp/*": "allow"
  },
  "question": "deny",        // 禁止向用户提问
  "read": {                  // 文件读取
    "*": "allow",
    "*.env": "ask",          // 敏感文件需确认
    "*.env.example": "allow"
  },
  "edit": {                  // 文件编辑
    "*": "deny",
    "src/**": "allow"
  }
}
```

### 5.4 Explore Agent 专用 Prompt

```typescript
const PROMPT_EXPLORE = `
You are a code exploration agent. Your job is to quickly find and understand
code in a codebase.

Guidelines:
- Start with broad searches, then narrow down
- Use glob for file patterns, grep for content
- Read key files to understand architecture
- Report findings concisely with file:line references
- Specify thoroughness level: quick/medium/very thorough
`
```

---

## 六、设计最佳实践总结

### 6.1 工具描述设计原则

1. **开头说明工具用途**（一句话）
2. **Usage 部分列举使用场景和限制**
3. **提供正面和反面示例**
4. **参数说明包含默认值和类型**
5. **说明与其他工具的关系**（优先使用哪个）

### 6.2 参数设计原则

1. **必选参数最小化**
2. **提供合理默认值**
3. **使用 enum 限制可选值**
4. **参数名语义清晰**
5. **支持增量/分页获取**（大数据量场景）

### 6.3 返回值设计原则

1. **主输出 `output` 面向用户可读**
2. **元数据 `metadata` 面向程序处理**
3. **提供 `title` 用于 UI 展示**
4. **错误信息具体且可操作**

### 6.4 System Prompt 设计原则

1. **按模型能力差异化**
2. **明确角色定位和目标**
3. **提供标准化工作流程**
4. **给出沟通风格示例**
5. **注入运行时环境信息**
6. **强调禁止行为（用大写强调）**

### 6.5 Agent 设计原则

1. **单一职责**：每个 Agent 专注一类任务
2. **最小权限**：只开放必要的工具和目录
3. **可组合**：通过 Task 工具组合使用
4. **可配置**：支持温度、步数等参数调整
5. **可扩展**：支持用户自定义 Agent

---

## 七、工具清单速查

| 工具 | 核心参数 | 返回值 | 不可或缺的原因 |
|------|----------|--------|----------------|
| `read` | file_path, offset, limit | 带行号的文件内容 | 编辑前的必要前置 |
| `write` | file_path, content | 成功/失败 | 创建新文件 |
| `edit` | filePath, oldString, newString | diff + diagnostics | 安全的代码修改 |
| `glob` | pattern, path | 文件列表 | 快速定位文件 |
| `grep` | pattern, path, glob | 匹配内容/文件 | 代码内容搜索 |
| `bash` | command, timeout | 命令输出 | 执行系统命令 |
| `webfetch` | url, prompt | 网页摘要 | 获取外部知识 |
| `websearch` | query | 搜索结果 | 发现相关资源 |
| `todowrite` | todos[] | 成功/失败 | 任务进度可视化 |
| `task` | prompt, subagent_type | Agent 输出 | 复杂任务分解 |
| `question` | question | 用户回答 | 需要人工确认时 |
| `skill` | name | 技能内容 | 领域知识注入 |
| `apply_patch` | patch | diff + diagnostics | GPT 模型的编辑方式 |
| `codesearch` | query, max_tokens | 代码示例/文档 | 获取最新 API 用法 |

---

## 八、参考资源

- 工具实现目录: `packages/opencode/src/tool/`
- System Prompt 目录: `packages/opencode/src/session/prompt/`
- Agent 定义: `packages/opencode/src/agent/agent.ts`
- 权限系统: `packages/opencode/src/permission/next.ts`
- 配置系统: `packages/opencode/src/config/config.ts`

---

*本文档基于 OpenCode 项目源码分析生成，用于 Agent 系统和内置工具设计参考。*
