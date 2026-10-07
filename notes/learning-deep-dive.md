# Grok Bot 0.18 重构项目深入学习文档

> 研究对象：`jasonchen505/grok-bot-0.18-reconstructed`（fork 自 `b-nnett/grok-bot-0.18-reconstructed`）
> 研读时间：2026-10-07；文档覆盖后端重构（`source/`）、确定性构建工具链（`scripts/`）、前端重构（`frontend/`）、上游背景
> 姿态：**只读学习**。该仓库无 LICENSE，上游已归档；第三方评审一致定性为 read-and-learn only，**不要把它的代码复制进自己的项目**。

## 一、一句话定位

这是一个**非官方的、面向源码的逆向工程项目**：作者从 Anysphere 发布的 Grok Bot 0.18.0 macOS 编译产物（`app.asar`）出发，恢复出可读的 TypeScript 实现（Electron 主进程、host、coordinator、协议边界），前端因官方只发了无 source map 的压缩 bundle 而保留 shipped renderer、只打最小 UI 补丁；并用一套确定性工具链把"恢复源码 + 被校验的官方产物"重新拼成一个可运行的 macOS 应用，顺手加了三个实验功能：推理路由器（Cursor/Claude Code/Codex/OpenRouter 四选一）、本地 Docker sandbox、本地用量统计。

## 二、上游背景（已查证）

- **产品归属**：Grok Bot 是 **Anysphere（Cursor 背后公司）** 的桌面 agent 应用，不是 xAI 的 Grok 聊天机器人（两者常被混淆）。证据：README 自称"不是 Anysphere 官方 monorepo"；官方安装器下载自 `downloads.cursor.com/grokbot/stable`；上游 bundle ID 为 `com.anysphere.sand`；NOTICE.md 明确"not affiliated with or endorsed by Anysphere, Cursor, xAI, or SpaceX"。
- **版本**：0.18.0，官方构建约 2026-08-13（据 sibling 仓库 grok-bot-0.18-original 标注）；Electron 42.1.0；`app.asar` SHA-256 `66654081…1de58`，DMG SHA-256 `a253ccd8…d203eb`。
- **作者与归档**：b-nnett 于 2026-08-23 创建（"周末研究项目"），2026-08-28 主动归档（创建仅 5 天后），现只读。未找到公开的归档原因说明，也未见 DMCA 施压证据；issue #9 里作者曾表态"no plans to maintain or monitor this repo"。3521 stars / 3647 forks，2026-08-28 曾登 GitHub trending（当日约 3406 stars）。
- **License**：**全仓无 LICENSE 文件**。NOTICE.md 明确："No upstream source-code license is asserted or granted here"，"complete an independent rights review before public redistribution"。PROVENANCE.md 补充：不要把重构材料当作官方源码或官方构建呈现。
- **社区反响**：YouTube 有讲解视频，Notion 有长文《The Free Grok Bot Alternative》，均强调"无 license、法律上不是开源"；tinyhumansai/tinyhivemind 的 survey 定性为 read-and-learn only、不可复制代码；rueclaw 评审结论 "Study it; do not deploy it as a normal app"（无可用许可、依赖逆向行为、npm audit 27 个生产告警）。衍生生态：`grok-bot-0.18-original`（未改动 runtime 归档 + 分析文档）、`youming-ai/opengrokbot`（明确不重新分发安装器）、`zzw09773/grok-bot-0.18-linux`（Linux fork）、被 awesome-grok-bot 收录为 "Study-only, archived"。
- **Issue 现状**（上游 open 7 个，不是 API 数字 14）：#11 路由 provider 的 secrets 到不了 Mac 端 coordinator（bug）；#9 路由路径下发工具却无 executor、模型空转烧 token（作者不维护，reporter 标注 "for the record"）；#2 Windows 重构支持（有贡献者想接手）；#3 用 GitHub Releases/Actions 发安装包；#10 一份阿拉伯语七阶段主重构计划；#7 AIDC 衍生项目许可澄清；#17 SlowMist 安全团队的私下漏洞联系请求。另有 7 个 open PR（路由修复、RPC 契约恢复、Docker gateway、Intel Mac、Linux port）、5 个 closed PR。

## 三、项目结构总览

```
source/                    # 可读 TypeScript 重构：Electron 主进程 / 后端 / 协议（约 1724 文件）
  electron-main/           # 桌面主进程：账户鉴权、box 连接器、coordinator 启动器、MCP 管理、secrets、IPC 适配器
  electron-preload/        # 渲染进程 preload 桥（window.desktop + window.coordinatorPort）
  node-agent-coordinator/  # 独立 Node 子进程：转录路由、inference router、gateway 客户端、MCP 桥
  host/                    # 跑在 sandbox/box 里的 agent host（gateway server、runner、MCP 认证、工具执行）
  box-exec-daemon/         # box 内执行守护进程
  local-exec-daemon/       # 本地执行守护进程
  shared/                  # 跨进程共享类型、RPC 契约、observability、inference 类型
  packages/                # proto 定义等
frontend/                  # 可读 React 渲染器重构 + 设计工作区
  production/              # 可发运渲染器根（ProductionRenderer.tsx 约 3800 行）
  recovered/               # 按功能面划分的恢复实现（22 个功能面）
  dev/                     # DevShell：脱离真实 desktop bridge 的预览壳 + DOM 检查器
  manifests/               # 证据锚点清单（ui-evidence-anchors.json 等）
scripts/                   # bootstrap / 编译 / renderer 补丁 / 打包 / 签名 / 校验
patches/                   # 第三方补丁
manifests/reconstruction/  # 重构期的小清单（构建直接消费）
tests/                     # node --test 回归测试（publication、router、updater guard 等）
research-archives/         # Git LFS 保存的原版 macOS/Windows 安装器（SHA-256 固定）
docs/                      # ARCHITECTURE.md / PUBLISHING.md
```

设计总纲（README + ARCHITECTURE.md）：成品 app 是 **hybrid**——runtime 从 `source/` 编译，renderer 保留 checksum-pinned 的 shipped 产物 + 一处确定性最小补丁（Router 设置 UI），bundle ID 换掉、ad-hoc 签名，机器上已装的官方 app 永不覆盖。构建对上游 0.18.0 是**外部 pinned 输入**：bootstrap 下载 DMG → 验 SHA-256 → 提取 `app.asar` → 水合到被 gitignore 的 `src/app/dist`。

## 四、后端架构详解（`source/`）

### 4.1 进程模型

```
polished shipped renderer
        │  desktop preload / RPC（window.desktop, window.coordinatorPort）
        ▼
   Electron main
        ├── settings、secrets、auth、插件生命周期
        ├── 远程 box connector（BrokeredHostConnector → GrokBotService ConnectRPC ensureSandBox）
        └── 本地 Docker connector（grok-bot-local-vm）
                        │
                        ▼
              coordinator + host（Node 子进程，由主进程 fork）
                        │
                 inference router
              ┌─────────┼─────────┐
           Cursor   Claude Code   Codex / OpenRouter
                        │
                  Grok Bot MCP tools
```

- **Coordinator**（`node-agent-coordinator/main.ts`）：由 Electron 主进程 fork 的独立 Node 子进程，经 MessagePort 三通道（control/data/main-data）通信。它是渲染器与 box 网关之间的代理：renderer-port-server 收渲染器请求 → inference-router 先做本地路由 → 未处理则经 `CoordinatorGatewayClient` 转发到 host gateway，SSE 事件流再转回渲染器。还负责 webauthn 签名器、MCP OAuth 转发、client-side-tool-v2 中继、local-exec-daemon 监管。
- **Host**（`host/gateway-server.ts`）：跑在 box（远程 sandbox 或本地容器）里的 HTTP 服务器。`POST /api/<method>` 走 `SAND_GATEWAY_COMMANDS` 命令表；`GET /events` 是 SSE 事件流（可选 gzip、15s 心跳）；Bearer token 鉴权用 `timingSafeEqual`；拒绝浏览器 origin；local-exec/webauthn 双桥通道。`host/runner/` 是 agent turn 执行器（沙盒动作审计、auto-review、subagent、shell 观察等）。

### 4.2 Inference Router（核心实验功能）

文件：`source/node-agent-coordinator/inference-router.ts`；设置持久化 key 为 `settings.router-provider.v1`，四档 `cursor | claude-code | codex | openrouter`，默认 `cursor`（`frontend/.../settings/overlay/router.ts`）。

- **拦截点**：所选 provider 非 `cursor` 时拦截 `sendPrompt`，走 `runRoutedProviderText`（`host/extensions/inference/provider-session.ts`）；`cursor` 时直接透传原路径。
- **本地转录**：按 agentId 维护 `inference-router-transcript.json`（schemaVersion 2，每 agent 最多保留 200 条，0600 权限原子写）。三层转录合并返回：远端转录 + 本地路由转录拼接后再 slice limit（`getAgentTranscriptTail` 等）。
- **流式伪装**：流式时 post `transcript` 事件伪装成远端转录；另用 250ms pulse 主动维持 composing 状态，防止远程 roster 刷新抹掉本地 activity。Reactions 也落在本地 store。
- **串行化**：不同 provider 按 agentId 走 per-agent promise chain，避免并发 turn 互相覆盖本地状态。
- **三路 provider 实现**：
  - `codexExecutor`：用用户自己的 `~/.codex/auth.json` 的 ChatGPT OAuth token，经 `codex-direct-responses.ts` 走自有 HTTP transport；强制文件 600 权限、拒绝 symlink（防提权）。
  - `claudeExecutor`：`@anthropic-ai/claude-agent-sdk` 的 `query()`，`pathToClaudeCodeExecutable` 指向本地 claude CLI；MCP 工具经 `createRoutedMcpBridge` 起一个 127.0.0.1 随机端口的本地 MCP HTTP 服务器（secret URL 路径、JSON-RPC，工具标注 readOnlyHint 等 annotation）。
  - `openRouterExecutor`：`ai` SDK 的 `createOpenAI` 兼容端点，默认模型 `openai/gpt-5.2`；`OPENROUTER_API_KEY` 优先取环境变量，否则从 box secrets store 读。codex/openrouter 的工具走 `ai` SDK ToolSet，`executeTool` 回调 dispatchRemote 到主进程执行。
- **已知坑**（上游 issue #11）：路由 provider 的 secrets 在某些路径下到不了 Mac 端 coordinator——secret 传递链（box secrets store → provider-session）是脆弱点。

### 4.3 Box 连接器：远程 vs 本地 Docker

- **远程**（`electron-main/box/`）：`BrokeredHostConnector` 经 `GrokBotService` ConnectRPC 调 `ensureSandBox` 申请 box，返回 gatewayUrl/gatewayToken/networkToken/vncUrl；支持 box-blocked 限流（retry-after，最大 15 分钟）、`x-automation-failure-hint` 错误标记；`EnvDescriptorHostConnector` 支持环境变量直连。
- **本地 Docker**（`local-docker-host-connector.ts`）：镜像 `public.ecr.aws/k0i0n2g5/cursorenvironments/universal:sand-box-latest`，容器名 `grok-bot-local-vm`，端口 1337/1339/1340/6080/6081/8790 **只绑 127.0.0.1**；host bundle 按 sha256 内容寻址只读挂载进容器；`~/.codex` / `~/.claude` 只读挂载透传用户已有的登录；容器用 label 做 schema 版本与所有权校验，拒绝操作非自有同名容器。
- **切换**：`createSettingsRoutedHostConnector` 按 `settings.getBoxRuntime()` 在 local-docker 与远程之间切换；Settings → Router 页有 "Use local Docker VM" 开关，远程仍是默认。

### 4.4 Settings / Secrets / Auth

- `SandSettingsStore`（settings.json）：inference provider、box runtime 等。
- `user-secrets.json`：经 Electron safeStorage（keychain/keyring）按账户槽加密存储；OpenRouter key 从环境变量或 box secrets store 取；codex 凭据走用户自己的 `~/.codex/auth.json`，claude-code 走本地 CLI 登录——**路由器设计上不要求用户再给一份 API key**，复用已有本地登录是它成立的关键。
- `window.desktop`（preload 桥）暴露面：附件管理、mcp（list/install/authenticate/工具开关）、cursorAccount（登录/用量/头像/PR review 偏好）、experiments、foreverBox（本地 box 重建、egress tunnel、webauthn proxy）、secrets（list/reveal/upsert/remove）、agent（pinned agents、sidebar、default model、**getInferenceRouter/setInferenceRouter**、boxRuntime）、update、主题/时区/windowControls、telemetry 上报通道、onboarding；另有 `window.coordinatorPort`（coordinator MessagePort broker）与 VNC preload。

### 4.5 安全纵深与防御性设计（工程观察）

- gateway auth 用 `timingSafeEqual`；codex `auth.json` 拒绝符号链接且要求 600；secret 文件 600 + 原子 rename 写；本地 MCP bridge 用随机 UUID 路径。
- 代码里大量注释解释"为什么"：1200ms composing 延迟是为了让虚拟化转录行可见；250ms pulse 防止远程 roster 刷新抹掉本地状态。
- 遥测密度高：transport stage recorder、send echo 记录、Sentry scrub 生成文件、RPC 全链路 traceparent 透传——这也是重构者选择默认关闭 telemetry 的原因之一。

## 五、确定性构建工具链（`scripts/`）

流水线：`bootstrap → build → package → verify`，另有 `publication:check` 证明 clean-history 导出无损。

1. **bootstrap**（`scripts/bootstrap-runtime.mjs`）：优先用已缓存 runtime；否则优先 Git LFS 归档 DMG（SHA-256 不匹配就报错并提示先 `git lfs pull`），再无则从官方 URL 下载（partial 文件 + SHA-256 校验后改名）。挂载 DMG 后缓存 runtime，并水合 checksum-pinned 的 source payload。定位明确：pinned 应用只提供 Electron shell、ABI 匹配的原生依赖和有文档的 build fallback。
2. **build**（`scripts/build.mjs → clean-build.mjs`）：构建 "fidelity hybrid ASAR"——clean-source 产出 + 显式 fallback 组合。`runtimeComposition` 声明每层 runtime 的模式（`clean-source` / `artifact-fallback` / `checksum-pinned-artifact-runtime`），附 `dist/runtime-composition-audit.json` 审计文件并计入 build manifest（schema v2，逐文件 SHA-256）。**fail-closed**：未达到 clean 闭包就标记 blocked。
3. **package**（`scripts/package-macos.mjs`，仅 darwin）：复制 pinned runtime 为新 bundle 后：`xattr -cr` 清除原厂商 quarantine 元数据（注释说明留着会让 Gatekeeper 拒绝 ad-hoc 签名）；`plutil` 删除 `ElectronAsarIntegrity`、替换 bundle ID 为 `com.anysphere.sand.reconstructed`、显示名 `Grok Bot 0.18 Reconstructed`；URL scheme 只保留 `sand`、移除继承的 `grokbot` 等别名；替换 asar 后 ad-hoc 签名并 `codesign --verify --deep --strict`（内建两次签名重试，macOS 偶发拒绝替换嵌套 framework 签名）。
4. **verify**（`scripts/verify.mjs`）：逐项校验打包产物：ASAR 必备条目、renderer runtime manifest 的 app-icon SHA-256 一致、renderer 源码 map 禁止存在、clean 产出必须含 `// Deterministic clean-source` 标记、artifact fallback 必须等于"不可变证据 + 确定性 transform"、runtime composition 与审计 JSON 一致、每 clean 输出字节/SHA-256 与 manifest 相符、checksum-pinned renderer 文件清单完整且无未声明文件。硬门槛：**clean 产出 ≥1000 个 `// src/` 证据标记**。
5. **renderer 补丁**（`scripts/lib/router-renderer-patch.mjs`）：不重建 renderer，对 shipped renderer 做最小文本注入。Settings registry chunk 与 panel chunk 各要求恰好 1 个候选（anchors `replaceExactlyOnce`，缺失或歧义即抛错）。三处改动：设置 tab registry 注入 `{id:"router", label:"Router"}`；`x==="router"` 时渲染 `RRouterPanel`；Usage 页注入 `RRouterUsage`。组件源码内嵌（四档 provider 选择器 + Docker VM 开关 + 用量统计 + 密钥保存），样式直接复用 shipped 的 `sand-xxx` 类名。每次打补丁写 `dist/renderer-router-extension.json` 存证（chunks 前后 SHA-256、bytes、schemaVersion）。
6. **publication:check**（`scripts/verify-publication-tree.mjs` + `docs/PUBLISHING.md`）：`git archive HEAD` → 解包 → `git init` → `add --all` → `write-tree`，要求新索引的 Git tree 与 `HEAD^{tree}` **逐字相同**，证明 clean-history 导出无损（无遗漏/多余文件）；另硬检查一个"被 gitignore 排除但仍要保留"的文件确实存在。PUBLISHING 还要求导出后从 fresh clone 完整跑完 `npm ci / bootstrap / check / package / verify`，并检查 LFS 归档含两个安装包。
7. **熔断机制**：`build-asar.mjs` 的 `reconstructedUpdaterGuard` 在 electron-main 源码头部强制前置 `SAND_DISABLE_UPDATES=1`、`SAND_DISABLE_SENTRY=1`、`SAND_DISABLE_TELEMETRY=1`；codebase-telemetry 的 csnaps 二进制无可恢复的 shipped carrier 时，打包审计要求 host 代码走 "availability-gated-no-op" 分支（无 carrier、无 spawn、no-op API）。
8. **CI**（`.github/workflows/check.yml`）：push/PR 在 ubuntu 跑 `npm run check` + `frontend:build` + `publication:check`，node 26.5.0，权限 `contents: read`；tests 全是 `node --test`，其中 publication-bootstrap/publication-packaging、reconstructed-updater-guard、router-settings 都是上述机制的专项断言。

## 六、前端重构（`frontend/`）

### 6.1 三子树分工

- **production/**：可发运渲染器根。`ProductionRenderer.tsx`（约 3800 行）装配全部 UI 状态与协调器客户端；`main.tsx` 挂到 `#root` 并在 `__reconstructed_health` 端点上报健康信息。注释中的 `@evidence` 只标注字节/符号出处，是溯源注释而非导入。
- **recovered/**：按功能面划分的恢复实现，是 ProductionRenderer 直接消费的单元；`contracts/`（如 `desktop-bridge.ts`）定义与桌面宿主/协调器的端口契约；`ui/` 是 sand-kit 基础控件。
- **dev/**：`DevShell.tsx` 开发壳，脱离真实 desktop bridge 用 fixture 预览 settings、plugins、hidden-chats、org-chart、conversation 等入口，内置 DOM 检查器（dom-inspector）核对恢复边界。

`frontend/README.md` 明确关系：可编辑渲染器由 Vite 构建并被 macOS 包默认选中；`npm run bootstrap` 从被忽略的 `src/app/dist` 水合 checksum 锁定的 shipped renderer 做保真度比对。

### 6.2 recovered/features/ 功能面（22 个一级目录）

conversation（对话工作区：composer、转录、侧边栏、消息卡片、工具结果）、agent-info（settings、头像编辑、channels、group-members、async-tasks、shared-room）、computer（shell 视图、rebuild、teach-recording 演示教学录制、update）、settings（设置弹层：general / router / usage / beta）、account（会话菜单、登录状态、登出）、automations（routines 定时任务）、roster（代理名单：选择态、隐私遮挡、重连通知）、access（访问遮盖层：首屏 gate、电脑重建状态）、plugins（插件弹层 auth 适配）、hidden-chats、onboarding、org-chart、permissions（本地工具权限 dock）、feedback / about / deep-links、update（强制更新视图 + 更新药丸）、terminal、window-chrome（根 shell 状态、全局快捷键、通知、状态徽章）、root-resilience（协调器连接状态）、error-boundary、runtime-theme-token-installer。

### 6.3 Router 设置 UI 的集成链路

Router 是 Settings 弹层的四个 section 之一。`settings/overlay/view.tsx` 用 `SETTINGS_SECTIONS` 显式注册 `[general, router, usage, beta]`（注释注明该注册表"从主渲染器 binding wDn 恢复"）。`ProductionRenderer.tsx` 维护 `overlay` 状态（`"hidden-chats" | "settings" | "plugins" | "about" | "feedback" | "confirm-logout" | null`）和 `settingsSection`，点击设置即把 `SettingsDesktopSurface` **懒加载**为 overlay 挂到 shell（带 `SettingsOverlayErrorBoundary` 和 Suspense 降级）——对应 `renderer-bootstrap.json` 里 `lazyBoundaries` 的 `overlay:settings` 条目。`section === "router"` 时渲染 `RouterSettingsPanel`，读写经 `bridge.agent.clientPersistence`，失败时经 surface-notice 发布错误；命令面板里也有 "Settings: Router" 条目（`settings:router`）直达该节。provider 选择还会影响 usage 节显隐：非 cursor provider 时强制显示 Usage & Billing。

### 6.4 证据 manifest 机制（"重构正确性锚点"）

`manifests/` 是"资产与已审核语义边界的标识"，每个重构模块必须能指向 shipped bundle 里的字节或字符串证据，而非凭空重写：

- **`ui-evidence-anchors.json`**：48 条目，把重构代码的 clean path 锚定到 shipped 产物 `index-UbX-y3il.js` 的具体证据：selector（如 `.sand-command-palette`）、visible-string、dom-signature、asset，外加行号/字节偏移，有的还指回 `component-names.json` 或 `evidence.ts` 注册表。
- **`renderer-bootstrap.json`**：挂载证据（`document.getElementById("root")` 的字节针）、runtime 获取（`window.desktop` / `window.coordinatorPort` 的 needles）、provider 顺序、根子节点符号、路由机制（`kind: "store-handle"`，browser/hash router 均为 null——该应用不用 react-router，用 store handle 做路由）、`lazyBoundaries`（5 个懒加载边界：computer、hidden-chats、org-chart、plugins、settings）与期望的 shipped routes。
- **`semantic-symbols.json` / `conversation-evidence.json`**：把压缩符号（如 `R5e`、`Q`）映射回恢复出的名字（如 `attachmentBasename`、`HiddenChatsDialog`），带行号、置信度、判定依据；`evidence.ts`（生产侧）把每个可见字符串做成可执行数据，让测试能验证所有可见文案在 shipped 产物中确实存在。
- `renderer-runtime-assets.json`、`component-names.json`：资源与组件名清单。

## 七、工程观察：这个项目的"方法论"值得学

1. **Evidence-only 规则**（PROVENANCE.md 原文精神）：immutable release 就是产品规格；恢复源码只能表达"至少有一个可检查产物锚点（产物代码、source-path 标记、source map、shipped 字符串/资源/CSS、renderer DOM 签名、IPC/RPC 契约、shipped runtime 的可重复观察）"支持的行为。renderer 尤其严格：不许发明屏幕/控件填证据缺口，证据不足就记录不确定性。**typecheck/build 通过不算 provenance 证明；推测行为是 release-blocking 缺陷。**
2. **Fail-closed 的构建**：clean 闭包达不到就 blocked；verify 有 1000 个证据标记硬门槛；renderer 补丁 anchors 要求 replaceExactlyOnce，缺失或歧义直接抛错。这种"宁可不构建也不猜"的姿态，是逆向工程项目里最难得的纪律。
3. **Hybrid 务实主义**：承认"完整重建前端 polish 是另一个数量级的逆向工程，周末做不完"，于是 runtime 全重构、renderer 保留官方产物 + 最小可审计补丁。这是对投入产出比的诚实判断。
4. **可复现性设计**：pinned 输入（SHA-256）、逐文件 manifest、audit JSON、publication:check 的 tree 逐字比对——整套是把"可复现构建"（reproducible builds）思想用到了逆向场景。
5. **防御性注释文化**：代码里大量解释"为什么"（composing 延迟、pulse、容器 label 校验），读起来像一份取证报告而不是普通源码。

## 八、批判性思考

1. **Router 的本质是"寄生"而非"替代"**：三个非 Cursor provider 的转录、reactions、composing 全靠本地伪装层（250ms pulse、伪装 transcript 事件）去"骗"渲染器，让它以为还是远端转录。这意味着任何 shipped renderer 的行为变更都会直接击穿伪装层——而 renderer 恰恰是作者声明"不重构"的那部分。脆弱性集中在最不被控制的一层。
2. **Secret 链是单点**：issue #11（secrets 到不了 coordinator）说明"复用本地登录免 API key"的设计虽然优雅，但 secret 在主进程 → coordinator → provider-session → box 之间的传递链没有统一抽象，每条 provider 都是特事特办（codex 读文件、claude 读 CLI、openrouter 读环境变量/box store）。
3. **"Evidence-only" 与"新增功能"的张力**：Router、Docker sandbox、用量统计都是作者**新写**的功能，不在 evidence-only 规则覆盖范围内（规则管的是"恢复"，不管"扩展"）。但新功能深度寄生在恢复出的边界上（preload 桥、coordinator 协议），而这些边界本身只是"证据支持的行为"的近似——两层近似叠加，正确性保证是乘法衰减的。
4. **法律灰度**：NOTICE 写得越严谨（"no license asserted or granted"、"independent rights review"），越说明作者清楚这是在版权灰度地带行走。3647 个 fork 里有多少人真的只读不抄，是未知数。SlowMist 来私下报漏洞也侧面说明：这类"可运行的逆向产物"天然是攻击面研究目标。
5. **归档的诚实**：作者 5 天后归档、不维护、不回应 issue/PR（#9 的 reporter 只能写 "for the record"）。对比很多逆向项目"假装会维护"，这种明确的"一次性研究产物"定位反而更负责任——但也意味着 issue 里的 bug（#11 secrets、#9 空转烧 token）永远不会修，**拿来跑等于用已知有坑的版本**。
6. **可学 vs 可用**：这个项目最大的学习价值不在"怎么跑 Grok Bot"，而在三个可迁移的方法：(a) 对编译产物的证据锚定式逆向（manifest + 字节针 + 符号映射）；(b) fail-closed 的可复现构建流水线；(c) 用最小补丁寄生官方 UI 的集成手法（replaceExactlyOnce anchors + 存证 JSON）。这三样搬到任何 Electron 应用逆向/二次开发场景都成立。

## 九、与之前研究项目的关联

- **与 humanize（agent flow 编排）的关系**：humanize 是"把 plan 变成可执行 run"的编排层；这个项目里的 coordinator + host + box 恰好是一个**真实存在的** agent 执行 substrate——coordinator 做转录路由、host 在 sandbox 里跑 runner、box 提供隔离。读完 humanize 的 flow/run/epic 抽象再看这里，能看到同一套思想在生产级产品里的具体形状（per-agent 串行、turn 执行器、工具审计）。
- **与 LeJudge 的关系**：LeJudge 强调"judge 只读词不读数、算术全在代码里"的信任边界设计；这里 preload 桥（`window.desktop`）的"窄可信桥"和 gateway 的 Bearer + timingSafeEqual + 拒绝浏览器 origin 是同一类思想：**把不可信输入挡在可验证的薄边界之外**。
- **与 Agora 的关系**：Agora 用 Git 做集体研究的 append-only 记忆；这里的 `dist/renderer-router-extension.json`（每次补丁的 chunks 前后 hash 存证）和 build manifest（逐文件 SHA-256）是**单机版的同一思想**：任何变更都留下可审计的证据链。

---

*文档位置：本 fork `notes/learning` 分支 `notes/learning-deep-dive.md`；`main` 分支未动。*
*研读方式：本地 shallow clone 全量阅读 + 4 路并行研读（后端/工具链/前端/上游背景）+ 关键事实逐条源码核验。*
