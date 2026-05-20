# OpenCode 仓库分析与 Agent 工作流

## 项目性质

**OpenCode** 是一个开源的 AI 编程助手，运行在终端（TUI）、Web 和桌面（Electron）三种界面中。由 [anomalyco](https://github.com/anomalyco/opencode) 开发，MIT 协议。

## 核心功能

1. **AI 对话代理** — 内置 `build`（有文件读写权限）和 `plan`（只读）两种 agent，连接 LLM 进行代码生成、编辑、解释
2. **工具系统** — 文件操作、命令执行、Web 搜索、LSP 分析、Git 集成等
3. **MCP/ACP 支持** — Model Context Protocol 和 Agent Client Protocol
4. **插件系统** — `packages/plugin` 提供 SDK 扩展
5. **多云 LLM** — 支持 OpenAI、Anthropic、Google、AWS Bedrock、Azure、Groq、Mistral、DeepSeek 等 20+ 提供商

## 技术栈

| 技术 | 用途 |
|---|---|
| **Bun** | 运行时 + 包管理 + 编译单文件二进制 |
| **TypeScript** | 全部源码 |
| **Effect v4** | 核心架构（类型安全的 DI、错误处理、并发、Schema 校验） |
| **SolidJS + OpenTUI** | TUI / Web / Desktop 三端 UI |
| **Drizzle ORM** | SQLite 本地数据库 + PlanetScale 云端 |
| **SST Ion** | 基础设施即代码（Cloudflare Workers、R2、PlanetScale） |

## Monorepo 结构

```
packages/
├── opencode/      ← 主 CLI + TUI + 核心业务逻辑 + HTTP API 服务器
├── core/          ← 共享抽象层（文件系统、配置、认证、插件加载等 Effect 服务）
├── llm/           ← LLM 抽象层（4 轴路由：协议/端点/认证/帧解析）
├── sdk/js         ← JavaScript/TypeScript 客户端 SDK（从 OpenAPI 生成）
├── app/           ← Web 应用（SolidJS + Vite）
├── desktop/       ← Electron 桌面应用
├── ui/            ← 共享 UI 组件（聊天、Diff、Markdown、文件树、主题、i18n）
├── plugin/        ← 插件 SDK
├── slack/         ← Slack 集成
├── enterprise/    ← 企业版（团队管理、Stripe 计费、多用户）
├── console/       ← 云平台（PlanetScale DB、认证、用量统计）
├── function/      ← Cloudflare Workers（API、GitHub App 认证）
├── http-recorder/ ← HTTP 录放测试工具（LLM 的 cassette 测试）
└── web/           ← 官网 + 文档（Astro + Starlight）
```

## Agent 通过 LLM 结果修改本地文件的完整流程

### 整体架构

```
用户输入 → Session Loop → LLM Stream(含Tool Call) → AI SDK自动执行Tool → 写文件 → 结果回传LLM → 循环
```

关键模块：`packages/opencode/src/session/prompt.ts`(主循环) → `processor.ts`(事件处理) → `tools.ts`(工具调度) → `tool/*.ts`(具体工具) → `core/filesystem.ts`(实际写盘)

---

### 第一步：主循环（`session/prompt.ts`）

`runLoop()`（第1240行）是一个 `while(true)` 循环：

1. 从 SQLite 加载消息记录
2. 创建空白的 assistant 消息占位
3. 调用 `SessionProcessor.create()` 获得 `Handle`
4. 调用 `SessionTools.resolve()` 把所有 Tool 定义转为 AI SDK 格式
5. 调用 `handle.process(streamInput)` —— **核心：发起 LLM 流式请求**
6. 检查结束原因：
   - `"stop"` → 退出循环
   - `"continue"` → 继续下一轮（此时 tool 执行结果已加入消息历史）
   - `"compact"` → 压缩上下文后继续

---

### 第二步：Tool 注册与调度（`session/tools.ts`）

`SessionTools.resolve()`（第24行）将 opencode 内部的 `Tool.Info` 定义包装成 AI SDK 的 `tool()` 对象：

```typescript
// tools.ts:81
tools[item.id] = tool({
  inputSchema: jsonSchema(schema),
  execute(args, options) {
    return run.promise(
      Effect.gen(function* () {
        const ctx = context(args, options)  // 构建 Tool.Context
        const result = yield* item.execute(args, ctx)  // 执行真正的工具
        return output
      }),
    )
  },
})
```

**核心机制**：AI SDK 的 `streamText()` 内置了 tool call 的执行能力。当 SDK 从 LLM 响应中解析出 tool-call 时，它会**自动调用**注册的 `execute` 函数，等结果返回后再发出 tool-result 事件。

---

### 第三步：事件流处理（`session/processor.ts`）

`handleEvent`（第304行）监听 LLM 流中的事件：

| 事件 | 行为 |
|------|------|
| `tool-call` | 记录到 DB，设为 "running" 状态（第376行） |
| `tool-result` | 记录输出，标记 "completed"（第451行） |
| `tool-error` | 标记 "error"（第503行） |
| `step-start` | 快照文件状态，用于 diff 追踪（第527行） |
| `step-finish` | 生成文件 patch diff，记录 token 用量（第554行） |
| `text-delta` | 累积流式文本（第618行） |

**关键点**：实际的 tool 执行是在 AI SDK 内部完成的，processor 只是在旁监听并持久化结果。

---

### 第四步：具体文件编辑工具

#### 4.1 Write Tool（`tool/write.ts`）

完整创建一个新文件或覆写已有文件：

1. **解析路径**（第41行）— 相对路径转绝对
2. **检查外部目录权限**（第44行）— `assertExternalDirectoryEffect()`
3. **读取已有文件**（第47行）— BOM-aware 读取
4. **计算 diff**（第53行）— 用 `diff` 库的 `createTwoFilesPatch()`
5. **请求许可**（第54行）— `ctx.ask()` 展示 diff 给用户确认
6. **写入文件**（第64行）— `fs.writeWithDirs(filepath, content)`
7. **格式化**（第65行）— 如果配置了 Prettier 等
8. **通知 LSP**（第75行）— 触发诊断
9. **返回结果** — diff 统计、LSP 错误等

#### 4.2 Edit Tool（`tool/edit.ts`）

对已有文件做 find-and-replace 修改：

1. **获取文件锁**（第88行）— 用 semaphore 防止并发编辑同一文件
2. **判断场景**：
   - `oldString` 为空 → 创建/覆写文件（第90-116行）
   - `oldString` 非空 → 查找替换（第118-168行）
3. **多策略替换**（第674-711行）— 按顺序尝试一系列匹配算法：
   - `SimpleReplacer` — 精确匹配
   - `LineTrimmedReplacer` — 忽略行首尾空格
   - `BlockAnchorReplacer` — 用首尾行做锚点 + Levenshtein 相似度
   - `WhitespaceNormalizedReplacer` — 规范化所有空白
   - `IndentationFlexibleReplacer` — 消除公共缩进
   - `EscapeNormalizedReplacer` — 处理转义序列
   - `TrimmedBoundaryReplacer` — 修剪边界匹配
   - `ContextAwareReplacer` — 首尾行锚点 + 50%中间行相似度
   - `MultiOccurrenceReplacer` — 全部精确出现位置
4. **写入+格式化+LSP通知**

#### 4.3 ApplyPatch Tool（`tool/apply_patch.ts`）

应用 unified diff patch：

1. **解析 patch**（第41行）— 提取 hunk
2. **分类每个 hunk**（第72-191行）— "add" / "update" / "delete" / "move"
3. **请求许可**（第206行）— 展示完整 diff
4. **逐项执行**（第220行）：
   - **add** → `afs.writeWithDirs(path, content)`
   - **update** → `afs.writeWithDirs(path, content)`
   - **move** → `afs.writeWithDirs(newPath, content) + afs.remove(oldPath)`
   - **delete** → `afs.remove(path)`
5. **发布事件**（第252行）— `File.Event.Edited` + `FileWatcher.Event.Updated`
6. **LSP 诊断**（第266行）

---

### 第五步：实际写入磁盘（`core/filesystem.ts`）

`writeWithDirs()`（第97行）是最终落盘函数：

```typescript
writeWithDirs = Effect.fn("FileSystem.writeWithDirs")(function* (path, content, mode?) {
  const write = typeof content === "string" 
    ? fs.writeFileString(path, content) 
    : fs.writeFile(path, content)

  yield* write.pipe(
    Effect.catchIf(
      (e) => e.reason._tag === "NotFound",  // 父目录不存在
      () => Effect.gen(function* () {
        yield* fs.makeDirectory(dirname(path), { recursive: true })
        yield* write  // 重试
      }),
    ),
  )
})
```

底层使用 Effect 的 `NodeFileSystem`。如果父目录不存在，自动递归创建。

---

### 第六步：上下文维护

- **消息持久化** — 每次 tool call 和结果都写入 SQLite（`PartTable`）
- **历史重建** — `toModelMessagesEffect()`（`message-v2.ts:630`）把 DB 记录转为 AI SDK 的 `ModelMessage[]`
- **快照追踪** — `snapshot/index.ts` 在每步前后用内部 git 仓库记录文件快照，用于 undo 和 diff 展示
- **压缩** — 当上下文超限时，`compaction.ts` 汇总对话并替换为精简版

---

### 完整数据流

```
用户输入 → runLoop()
  → streamText() 调用 LLM
  → LLM 返回 tool-call（如 "edit" 工具，参数：文件路径、oldString、newString）
  → AI SDK 自动调用 EditTool.execute()
    → EditTool:
      1. 读取文件
      2. 多算法匹配 oldString
      3. ctx.ask() 展示 diff 给用户
      4. fs.writeWithDirs() 写文件
      5. LSP 通知 + 诊断
      6. 返回 { output: "替换成功，+2/-1 行" }
  → AI SDK 发出 tool-result 事件
  → processor 记录到 DB
  → 循环继续，tool call+result 进入下一轮 LLM 请求的上下文
  → LLM 继续推理（更多 tool call 或最终文本回复）
  → 直到 finish_reason="stop"
```

### 关键文件索引

| 文件 | 用途 |
|------|------|
| `packages/opencode/src/session/prompt.ts` | 主会话循环 (`runLoop`, line 1240) |
| `packages/opencode/src/session/processor.ts` | LLM 响应流式事件处理 |
| `packages/opencode/src/session/llm.ts` | LLM 流创建 (AI SDK 或原生) |
| `packages/opencode/src/session/llm/ai-sdk.ts` | AI SDK 事件 → LLMEvent 适配层 |
| `packages/opencode/src/session/tools.ts` | Tool 定义包装为 AI SDK 格式 |
| `packages/opencode/src/session/message-v2.ts` | 消息/part schema、历史转换 |
| `packages/opencode/src/session/session.ts` | 会话 CRUD、DB 操作 |
| `packages/opencode/src/session/run-state.ts` | 会话执行状态管理 |
| `packages/opencode/src/tool/tool.ts` | Tool 定义框架、校验、截断 |
| `packages/opencode/src/tool/registry.ts` | Tool 注册表 - 构建所有 tool 定义 |
| `packages/opencode/src/tool/write.ts` | 写文件工具 |
| `packages/opencode/src/tool/edit.ts` | 编辑文件工具（多算法 find-and-replace） |
| `packages/opencode/src/tool/apply_patch.ts` | 应用 unified diff patch 工具 |
| `packages/opencode/src/tool/external-directory.ts` | 外部目录权限检查 |
| `packages/opencode/src/agent/agent.ts` | Agent 定义、权限、配置 |
| `packages/core/src/filesystem.ts` | `writeWithDirs()` 实际文件 I/O (line 97) |
| `packages/opencode/src/util/bom.ts` | BOM-aware 文件读写工具 |
| `packages/opencode/src/snapshot/index.ts` | 基于 git 的文件变更追踪 |
| `packages/opencode/src/permission/index.ts` | 权限系统 (ask/allow/deny) |
| `packages/opencode/src/session/compaction.ts` | 上下文溢出处理 |

## CI/CD 工作流

- **包管理/构建**: Bun + Turborepo
- **类型检查**: `tsgo`（Go 版 tsc，速度更快）
- **Lint**: `oxlint`（Rust 版 linter）
- **测试**: 每包独立 `bun test`，禁止根目录运行；LLM 测试使用 HTTP cassette 回放
- **发布**: GitHub Actions 自动构建跨平台 CLI 二进制（macOS/Linux/Windows），签名、发布到 npm/Homebrew/Scoop/AUR/Docker/Electron
