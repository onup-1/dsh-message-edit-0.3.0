# DSH Message Edit（0.3.0 · 非官方移植版）

> **这不是原作者发布的版本。**
>
> 上游是 [Moeblack/dsh-message-edit](https://github.com/Moeblack/dsh-message-edit)（MIT，0.2.3）。上游按 DSH `0.1.0-rc.6` 的内部 API 编写，在 DSH `0.2.2-rc.5` 上**完全不可用**（GET 一律 409、POST 一律 400）。
>
> 这里是我按自己的使用需要移植到 DSH 0.2.x 的版本：只改了构建产物（`index.mjs` / `client.js`）与 `package.json`，**没有改上游的 `src/`**（本仓库因此也不含源码）。版权仍归原作者，MIT 许可与署名见 [LICENSE](LICENSE) 与 [NOTICE.md](NOTICE.md)。
>
> 上游若发布官方 0.2.x 版本，请优先用官方版本。

`dsh-message-edit` 为 DeepSeek Harness 补充基于事件溯源的「消息编辑与重生成」能力。插件不改写历史事件，也不修改 DSH 引擎内部；每次编辑、重生成或重试都会从目标回合之前创建一个新会话版本，原会话始终保留并可随时切回。

## 安装（本移植版）

```
plugin_manager install_bundle  target=<本目录绝对路径>
```

上游的 `dsh plugin --profile web add dsh-message-edit` 装的是 npm 上的 0.2.3，**在 DSH 0.2.x 上不可用**。

## 0.3.0 相对上游 0.2.3 的改动

| 上游（rc.6 API） | 本移植版（0.2.x API） |
| --- | --- |
| `Session.events`、`SessionPersistence.inspect()` / `readFrom()` | `ctx.sessionQuery.readSession(id)` 返回的 `{ events, inheritedEventCount }` |
| `header.seedLength` | `Session.inheritedEventCount` / 快照的 `inheritedEventCount` |
| seed 直接以事件前缀开始 | seed 末尾补 `session/end-seed { inherited: true }` 标记 |
| 插件自定义事件 | `message-edit/version` 带 `ignorable: true`（否则读路径拒绝解释） |
| 合成 `assistant/message` 只写 `{turn, step, message}` | 补 `stream: []`，且不能带 `sourceEventSeqs` |
| `ctx.sessions.open(id)`（0.2.x 没有） | `ctx.get('uiWorkspace').openSession(id)`，回退 `sessions.retain(id, { source: 'mainView' })` |
| `inject` 里写了不存在的 `conversation` 服务 | 改成 `["slots", "sessions"]`（写错会让插件静默永不生效） |
| `require('@deepseek-ai/dsh-client-runtime/client')` | 自带一个极小的 `createSnapshotStore`，只 require `react` / `react/jsx-runtime` |
| 分支会话在侧边栏各占一行 | 版本分支自动归档（`PARK_VERSION_BRANCHES`，默认开） |

另外修了一个上游在 rc.6 上也存在的静默 bug：用户输入会先以 `agent/inbox/spliced` 排队，边界取 `turn.startSeq - 1` 时这条排队事件被留在继承前缀里，导致**改了提问却仍执行原任务**。移植版会向前找到那条排队事件并把边界移到它之前。

## 功能

- **编辑消息**：可编辑已落定的用户文本、`assistant.reasoning` 思考块与 `assistant.response` 回复文本。
- **重生成**：从最后一条已落定助手回复所属回合之前分支，使用原用户输入重新生成。
- **重试任意回合**：在 Timeline 中选择任意历史回合重新执行。
- **级联策略**：
  - `truncate`（默认）：只重新执行目标输入，删除该点之后的旧后续。
  - `preserve`：保留后续用户输入，并在新分支中依次重新执行；助手输出与工具链全部重新生成。
- **版本切换**：会话标题栏的 `←` 撤销当前原子效果，`→` 重施加最新直接子效果；Timeline 展示完整已知分支树、操作时间、编辑前后内容与当前版本。
- **Timeline 标签页**：注册到 `conversation.view`，`order: 15`，位于 Trajectory（10）与 Prompt Studio（20）之间。

## 设计

### 时间可组合性

插件把**完整回合**作为效果原子。目标回合的 `turn/start`、模型请求、工具调用、工具结果与 `turn/end` 不会被局部复制后拼接；新版本从该回合之前的闭合边界分支：

1. 用户消息编辑、Reroll 与 Retry：回退整个目标回合，再把目标用户输入作为新回合交给 Agent。
2. 助手块编辑：回退整个目标回合，以原用户输入和编辑后的助手内容构造一个新的完整闭合回合；原工具链不进入新版本。选择 `preserve` 时，后续用户输入再依次交给 Agent，产生新的完整工具链。
3. 每个版本都追加一个不可拆分的 `message-edit/version` 效果对：`effect` 记录正向效果，`inverse` 记录恢复目标。父版本链自动导出组合逆；恢复不是删除事件，而是沿逆链切换到仍然存在的版本。
4. 消息历史变换彼此不交换，因此撤销遵循 LIFO：一次只撤销当前原子效果并保留更早效果；各后继分支始终保留，可从父版本重新施加。

### 分支与 Agent 接线

旧实现先用短生命周期 Session 暂存分支、落盘、移除 live Session，再用 `agents.resume()` 重建 Agent。这个过程存在两个分离的生命周期边界：暂存日志已经持久化后，Agent 仍可能创建失败。现实现只使用 `AgentRegistry.create()` 已公开的 `seed + meta` 事务缝：

1. 在来源 Agent 的 runMaintenance() 内，从已闭合边界取得不可变 seed；第一回合之前使用空 seed。
2. 用本地等价的纯事件构造器把版本效果对与可选手工助手回合加入 seed，再调用 `ctx.agents.create({ seed, meta })`。Session 在 Agent 构造前一次性验证完整 seed；任何一步失败都会由 AgentFactory 的结构性逆撤销，外部观察者看不到半成品 Session，Agent 的回合计数也直接从完整历史初始化。
3. 发布后调用 `ctx.sessions.flush()`，在 HTTP 操作成功前建立耐久性屏障。
4. Workspace 挂接与 child Agent 生命周期分别返回原子逆；操作失败时按相反顺序组合恢复。随后通过 `child.agent.followup()` 排入需要重新执行的用户输入。

此路径不接触 `ReactLoopAgent`、AgentLoop 私有方法或 apiproxy 的收窄 fork RPC；分支 seed 仍由同一 Session 公共事件契约验证。

### 空间可组合性

- Host 只依赖公开的 `sessions`、`agents`、`sessionPersistence`、`sessionQuery`、`workspaceRegistry` 与 `webServer` 服务。
- Browser 只通过 `slots`、`conversation`、`connection` 与 runtime `sessions` 服务组合。
- Timeline 与标题栏共享一个按 `sessionId` 建立的值级 Snapshot source；控制器反应式订阅当前 Session 的闭合回合值与 Session 列表中的谱系值，Session 身份替换时重新绑定，不缓存旧 Session 对象。
- 新版本导航等待 runtime Session 列表发布对应 ID 后再执行 `ctx.uiWorkspace.openSession()`（回退到 `sessions.retain(id, { source: 'mainView' })`），依赖可用性变化直接驱动导航。

## 数据模型

每个插件版本在自己的非继承后缀中包含一个 `message-edit/version` 事件：

```ts
interface MessageEditVersionEvent {
  schemaVersion: 2
  effect: {
    id: string
    operation: 'edit' | 'reroll' | 'retry'
    cascade: 'truncate' | 'preserve'
    targetTurn: number
    targetEventSeq: number
    targetBlockIndex?: number
    blockKind?: 'user' | 'assistant.reasoning' | 'assistant.response'
    before?: string
    after?: string
  }
  inverse: {
    kind: 'restore-version'
    sessionId: string
  }
}
```

会话头的 `parentSession` 构成版本树，且必须与事件中的 `inverse.sessionId` 一致；`Session.inheritedEventCount`（0.2.x；旧版为会话头 `seedLength`）区分当前版本自己的元数据与从祖先继承的同名事件。Timeline 通过 `ctx.sessionQuery.traceSession()` 和 `readSession()` 生成完整值级投影，并由原子逆链导出 `undoStack` 与直接 `redoSessionIds`。旧版平面事件仍可读取，并在投影时规范化为同一效果对。

## UI

- `conversation.view`
  - `id: message-edit-timeline`
  - `order: 15`
  - `label: Timeline`
- `conversation.session.header.actions`
  - `id: message-edit-controls`
  - 直接父效果撤销、直接子效果重施加、效果链计数、最后回复重生成

组件使用 CSS Modules 与 `--dsw-*` 语义 token，不引入 UI 库。所有产品文案为中文，代码注释为英文。

## 构建

```bash
npm install
npm run build
```

构建基于 npm 发布的 `@deepseek-ai/*@0.1.0-rc.6` 类型与本地工具链（typescript、tsdown、lightningcss），不再依赖 dsh 源码树。构建生成：

- `index.mjs`：Host 插件
- `client.js`：Browser 插件
- `client.js.map`：Browser source map

## 安装

```bash
dsh plugin --profile web add dsh-message-edit
```

或本地开发：

```bash
dsh plugin --profile web add -w link:/path/to/dsh-message-edit
```

`dsh plugin` 是 pnpm 转发器：`add` 后会自动识别 `dsh.bundle` 声明并把插件收编进 profile 的 `dsh.profile.bundles`，重启 dsh 即生效。本地开发建议用 `link:`（符号链接），改动源码重构建后重启即更新。

## HTTP 接口

- `GET /message-edit?sessionId=<id>`：读取可编辑消息、可重试回合与完整版本树。
- `POST /message-edit`：执行 `edit`、`reroll` 或 `retry`，返回已发布的新 Session ID。

## 范围边界

- 不原地改写 Session 事件；历史是 append-only、deep-frozen。
- 不联动恢复或修改工作区文件、命令外部效果与既有产物。
- 不修改 DSH 引擎、apiproxy 或官方 UI 包。

## 0.3.0：移植到 DSH 0.2.2-rc.5

0.2.3 是按 `@deepseek-ai/*@0.1.0-rc.6` 编译的，在 0.2.x 上完全不可用（`GET` 恒 409「当前版本不在版本树中。」，`POST` 恒 400「events is not iterable」）。0.3.0 是同一功能的状态机在 0.2.x 公共 API 上的重新接线，行为与事件格式保持一致。

Host 侧：

- 读日志改用 `ctx.sessionQuery.readSession(id)` 返回的 `{ events, inheritedEventCount }`（`Session.events` 与 `SessionPersistence.inspect/readFrom` 在 0.2.x 已删除），`seedLength` 换成 `Session.inheritedEventCount`。
- 分支 seed 由 `inheritedSeed()` 构造：继承前缀 + 位于切点的 `session/end-seed { inherited: true }` 标记；`message-edit/version` 事件带 `ignorable: true`（0.2.x 的读路径拒绝解释白名单外的插件事件，除非显式忽略）。
- `agents.create({ seed, inheritedEventCount, meta: { parentSession, isSeeded, cwd, agentPreset } })`：`inheritedEventCount` 是顶层字段，不再放进 `meta`。
- 切点修正 `queuedInputBoundary()`：0.2.x 的用户输入先以 `agent/inbox/spliced { target: 'next-turn' }` 排队、回合开始时才 drain 成 `user/message`。若切点落在排队事件之后，该排队输入会被继承并优先 drain，导致「编辑了提问却仍在跑原任务」。现在把切点退到排队事件之前，编辑后的文本才真正成为模型看到的输入。
- 手工助手回合的 `assistant/message` 补齐 0.2.x 的结算字段：`{ turn, step, message, stream: [] }`，且不能带 `sourceEventSeqs`。

Browser 侧：

- `createSnapshotStore` 改为 bundle 内自带的等价实现（`getSnapshot/subscribe/set/update`），不再依赖已不存在的 `@deepseek-ai/dsh-client-runtime/client`。
- 槽位注册改用 0.2.x 的 `ctx.slots.inject(name, () => ctx.slots.register(...))`，`label` 改为函数。
- 版本切换改用 `ctx.uiWorkspace.openSession(sessionId)`（与侧边栏点击同一导航路径），不再调用已不存在的 `sessions.open()`。
- `inject` 只声明真实存在且确实取用的服务：`["slots", "sessions"]`。Cordis 的 `inject` 是「必需服务」声明，只要列了 0.2.x 不存在的服务名（例如 `conversation`，0.2.x 里叫 `uiConversation`），插件就永远停在未生效状态、`apply()` 不被调用，Timeline 标签与 `←/→` 按钮全部不渲染，而且控制台没有任何报错——这是移植初期「装了却看不见」的真正原因。
- 版本分支默认**不进入会话列表**：每次编辑/重生成/重试都会 fork 出一个新 Session，DSH 侧边栏按 Session 逐个列出，于是同一个对话会被「切成两份」。现在创建分支后立即调用 `WorkspaceRegistry.archiveSession`，把分支归档（默认筛选下不显示），侧边栏里这份对话仍然只有一行，分支通过 Timeline 的 `←`/`→` 往返切换。归档与 DSH 的「归档会话」是同一机制，可随时右键「取消归档」把它变回独立会话；把 `index.mjs` 顶部的 `PARK_VERSION_BRANCHES` 改为 `false` 可整体关掉该行为。

已在 DSH 0.2.2-rc.5 上实测通过：`GET` 投影、编辑用户文本（新分支重跑编辑后的提问）、编辑助手文本（不触发模型调用）、重生成、重试任意回合、版本树与 `←/→` 切换；编辑第 50 轮后，原分支第 51 轮之后的内容完整保留。GUI 侧已人工确认：会话右侧出现 **Timeline** 标签、标题栏出现 `←` / `→` /「重生成最后回复」按钮，编辑后能看到改写过的回复。

级联语义补充：`truncate`（默认）只重跑目标输入；`preserve` 会把目标之后的**用户提问**按顺序排队重跑，而「用户提问」的判定是 `user/message` 事件的 `data.source.kind === "user"`。由子代理 `send_message` 或其他内部来源注入的 user 消息不算用户提问，因此不会在 `preserve` 下被重放。

