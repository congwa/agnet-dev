# 17 · 番外：Grok Build 全项目速览——第四个样本

> **一句话导读**：xAI 在一次"把用户家目录传上云"的数据事故后两天内开源了自家 coding agent「Grok Build」——155 万行 Rust、81 个 crate，架构上处处能看到本系列前 16 章讨论过的决策点被第四次回答：状态用 actor 无锁、并行工具用完成序回灌、压缩切在语义锚点、hook 走外部进程、扩展生态整个**寄生在 Claude Code 的文件约定上**。但它同时也是"开源 ≠ 开放"的标本：不收外部 PR、system prompt 在二进制里 XOR 混淆、遥测端点全部构建期注入——源码全给你看，控制权一点不给。
>
> 调研时间：**2026-08-06**。仓库 `xai-org/grok-build`，读的是当日 HEAD（commit `a5589e9`，2026-08-05 从内部 monorepo 同步，`SOURCE_REV = 4d6d1137…`）。调研方式：五路并行读源码 + 对关键结论抽样复核，文中所有 file:line 为当日实测。事故经过部分来自媒体报道，**属第三方来源**，已在文中标注。
>
> **本章定位**：番外，也是 Grok Build 的总入口。第 01–15 章各在章末有「第四个样本：Grok Build」小节承接本章、按各自维度展开，16 章总表已扩入 Grok 列；本章负责全项目的整体视图、事故背景与总判断。三个主样本的既有结论不受影响。

---

## 0. 背景：它为什么开源

先交代这个仓库的来历，因为它直接解释了后面好几处代码形态。

**时间线**（综合 DevOps.com、The Decoder 报道，第三方来源、未经官方逐条确认）：

- 2026-07 上旬：研究员用 mitmproxy 抓包发现，Grok Build 在一个 12GB 的测试仓库上执行任务，实际任务只需约 192KB 数据，却向 xAI 的 Google Cloud 上传了 **5.1GB**（73 个数据块，约为必需量的 27,800 倍）。用户报告在家目录运行时，SSH 密钥、密码管理器数据库、个人文档和照片被上传；**隐私开关不影响上传行为**。
- 2026-07-13：xAI 关闭上传服务器（无安全公告）。
- 2026-07-14：GitHub 上创建 `xai-org/grok-build` 仓库（GitHub API 实测 `created_at: 2026-07-14T20:04:23Z`）。
- 2026-07-15：正式开源，Apache-2.0。Musk 称已上传数据将被 "completely and utterly deleted"；受影响用户数未披露，也未提供数据已删除的验证途径。

**开源的性质**：这是**源码透明化，不是社区项目**。`CONTRIBUTING.md:3-4` 原话："This repository does **not** accept external pull requests or unsolicited patches."，且明确不提供 CLA。仓库是内部 monorepo 的周期性只读镜像（README.md:31-35），与 Codex 同款模式。截至调研日 24k+ star。

这个背景带来两个阅读视角：一，代码里能找到**事故整改的直接痕迹**（第 6 节）；二，整个开源姿态是"你可以审计我，但别想改我"——这条张力贯穿全篇。

---

## 1. 规模与架构：TUI 比引擎还大

### 1.1 规模盘点

| 指标 | 数值 | 口径 |
|---|---|---|
| workspace member | 81 个 crate | 根 Cargo.toml members 数组 |
| .rs 总行数 | 1,547,925 行（2,513 个文件） | 含测试（仅 tests/ 目录就有 14.6 万行） |
| 最大 crate | `xai-grok-pager`（TUI）48.6 万行 | **TUI 比 agent core（xai-grok-shell，37.6 万行）还大** |
| 最大单文件 | `xai-grok-shell/src/agent/config.rs` 12,717 行 | — |

一个命名陷阱：`crates/codegen/` 下 64 个 crate **全是手写代码**（占仓库 95% 行数），目录名为何叫 codegen 仓内无解释，推测是 monorepo 路径残留。真正入库的生成物只有 14 行的加密 prompt 文件（见第 4 节）。

对照：Pi 11 个 npm 包、Codex 130 个 crate、Grok Build 81 个 crate——规模上是 Codex 的直接同类。

### 1.2 分层：介于 Pi 和 Codex 之间的第三种形态

第 01 章讲过 Codex 的铁律：TUI 对 core 的引用数为 0，只认协议。Grok Build 不一样：

```
   Pi                Grok Build                    Codex
 TUI 直接调用      TUI 直接链接 core，但          TUI 零 core 依赖，
 Agent 对象        通信走进程内 ACP channel       只依赖协议 crate
    │                    │                            │
 无协议边界        协议形状的边界 + 直接链接        编译期可验证的边界
```

证据：`xai-grok-pager/Cargo.toml:99` 直接依赖 `xai-grok-shell`，但 agent 跑在同进程独立线程上，TUI 经 `xai-acp-lib` 的内存 channel 与之收发 ACP 消息（`pager/src/acp/spawn.rs:1-4`："Simplified to only support GrokShell (in-process) mode. Subprocess and remote modes can be added later"）。协议本体不是自研，**外采 Zed 的 `agent-client-protocol` 0.10.4**（根 Cargo.toml:97），并且内外统一：进程内 TUI、`grok agent stdio` 的 IDE 嵌入、headless 全走同一套 ACP 消息。ACP 在这里已经从"编辑器协议"膨胀成整个产品的内部 RPC 面——`xai-grok-shell/src/extensions/` 下有 150+ 个 `x.ai/*` 扩展方法。

**一个引擎四种形态**（全部入口在 `xai-grok-shell/src/agent/app.rs`）：TUI（进程内线程）、编辑器嵌入（`run_stdio_agent`，:277）、headless（`run_headless`，:409）、常驻 leader 服务（`run_leader`，:974，socket + WebSocket bridge 多客户端接入）。反直觉的一处：**headless 强制要求 grok.com 登录会话，唯一传输是 relay WebSocket，没有本地 fallback**（app.rs:421-424）——API-key 用户被指去 stdio 模式。CI 场景反而是绑云最深的形态。

还有一处姿态：入口二进制启用了 obfstr/cryptify **编译期字符串+控制流混淆**（xai-grok-pager-bin/Cargo.toml:66-68 "Binary hardening"）——开源项目给自己的发行二进制上混淆，全系列仅此一家。

---

## 2. 逐维度速览：Grok Build 在三家坐标系里的位置

先给全景表，后面几节展开最有信息量的维度。

| 维度（对应章） | Grok Build 的答案 | 最接近谁 |
|---|---|---|
| 循环形状 [02] | 三层嵌套手写 loop + `tokio::select!` 命令 actor | Codex |
| 状态同步 [14] | **actor + mpsc channel，无锁**；读者拿深拷贝 | 谁都不像（第四种） |
| 并行工具回灌 [05/14] | **FuturesUnordered 完成序** + call_id 配对 + 写边界修复 | 与 Codex 相反 |
| 上下文策略 [03] | system prompt 仅 4.6KB，重量下放 user 前缀和工具描述 | Pi（更极端） |
| 项目约定文件 [03] | **没有 GROK.md**，原生认 CLAUDE.md/AGENTS.md/.cursor | 寄生策略（独有） |
| 压缩 [04] | 客户端 full-replace，切点=最后一条真实 user 消息 + prefire 预压缩 | 第四种切点哲学 |
| 记忆 [09] | markdown + SQLite 混合检索 + "dream" 睡眠整理，默认关 | Codex（但遗忘机制不同） |
| 持久化与分支 [10] | 双 JSONL（可替换投影 + 只追加事件日志），fork=目录复制 | Pi 与 Codex 的混合 |
| 钩子 [06] | 外部进程 + HTTP webhook，**配置格式兼容 Claude Code hooks.json** | Codex 同族 |
| 插件/skills [07] | git 仓库市场、无签名；skills 是 Claude Code 格式超集 | 寄生策略 |
| MCP [08] | rmcp 官方 SDK，stdio/HTTP/SSE + OAuth；自身不当 MCP server | Codex |
| 子 agent [13] | `task` 工具 + ACP 子会话，默认后台，worktree 隔离，深度 1 | Codex |
| Code Mode [05] | 没有；位置被 **Rhai 脚本编排子 agent** 占了 | 独有 |
| 模型抽象 [12] | fork async-openai，单 client 三协议方言（含 Anthropic /v1/messages） | 谁都不像 |
| 沙箱 [11] | Landlock/Seatbelt 两套（Windows 无内核沙箱）+ 规则引擎审批 | Codex（少一套） |
| 遥测 [15] | 默认关 + 端点构建期注入，但 remote settings 可远程打开 | 与 Codex 互为镜像 |

---

## 3. 循环与状态：第四种状态架构

第 14 章的根问题——"允不允许两处代码同时改状态"——Grok Build 的答案仍是"不允许"，但实现路径和 Pi（单线程 throw）、Codex（Mutex）都不同：**把状态关进一个 actor，用消息队列天然串行化**。

```
  SessionActor (tokio::select! 命令循环)          ChatStateActor（独立 task）
  ┌──────────────────────────────┐   mpsc 命令   ┌────────────────────────────┐
  │ prompt 队列 / 运行任务槽(单个) │ ────────────► │ state: ChatState           │
  │ 插话 buffer / 定时器          │               │   conversation: Vec<Item>  │
  └──────────────────────────────┘   mpsc 事件   │ persistence: Box<dyn ..>   │
                                   ◄──────────── │ ── 独占所有权，&mut self ── │
                                                 │    "no locks needed"       │
                                                 └─────────────┬──────────────┘
                                                               │ write-behind
                                                               ▼
                        ~/.grok/sessions/<cwd>/<sid>/
                        ├── chat_history.jsonl   ← 可整体替换的模型侧投影
                        └── updates.jsonl        ← 只追加的 UI 事件日志
                            （rewind 不删日志，追加 RewindMarker）
```

- **无锁是设计宣言**：`xai-chat-state/src/lib.rs:12` 的 ASCII 架构图里直接写着 "State (no locks needed)"；persistence trait 用 `&mut self`，注释明言 "no locks, no atomics, no shared state"。代价是读者视图：`GetConversation` 是**全量深拷贝**（actor/mod.rs:305-311，还打了 "cloning full conversation" 的 debug log 自认成本）——没有 Codex 的 Arc COW，比 Pi 的 `.slice()` 浅拷贝更贵，靠对话 pruning 控上界。
- **并行工具回灌与 Codex 恰好相反**：`FuturesUnordered`（tool_calls.rs:611），结果**按完成顺序**进历史，不保证请求序。正确性不靠序，靠三件套：call_id 配对（协议层天然无序安全）、写边界修复不变量（`repair_dangling_tool_calls` + `dedup_duplicate_tool_results`，只在三个写边界跑）、turn 末 reconciliation。第 14 章说 Codex 的 `FuturesOrdered` 是"正确性藏在类型名里"——Grok 直接放弃了顺序保证，把不变量做成显式修复函数。**两种哲学：一个防患于未然，一个宽进严出。**
- **竞态防护粒度是文件路径级**：同一批并行调用里，非只读操作命中同一 `file_path` 参数时共享一把 per-path `tokio::sync::Mutex` 按模型发出顺序串行（tool_dispatch.rs:50-59），其余全并发。
- **插话是双范式并存**：默认 Pi 式排队（`InterjectionBuffer`，"An interjection never cancels the turn"，interjection.rs:332，且在同 turn 的循环边界就排干、比 Pi 更激进）；另留 `send_now` 的 Codex 式 cancel-and-send 硬中断（commands.rs:233）。第 02 章的"排队 vs 打断"之争，Grok 的答案是"都要，让队列策略函数裁决"。
- **崩溃语义**：内存 actor 是运行时真相，落盘默认 Buffered 不 fsync（`AppendDurability`，jsonl/mod.rs:22-25）——比 Pi 的同步 append 弱、比纯内存强；JSONL 有 torn-tail 自愈（append 前查最后字节非 `\n` 就先封死残行，storage/jsonl/mod.rs:257-330，注释里写着这个 bug 曾经 "bricked session resume"）。
- **防傻转圈是显式机制**：连续相同 (tool, args) 签名 8 次注入 nudge、16 次硬停，`run true` 式空转 4 次即停（turn.rs:2724-2728，常量间还有编译期断言）；硬停返回专用 `TurnOutcome::StationarityEnded`，注释明确"与 Completed 区分，防止恢复逻辑重开循环"。四家里唯一把"模型转圈"当一等失败模式处理的。
- **时间旅行三机制并存**：会话内 rewind（每 prompt 都是 checkpoint，含文件快照回滚）、fork（**目录复制**型分支，非 Pi 的消息树、非 Codex 的字节区间引用）、跨压缩 rewind（重放 `updates.jsonl` 事件日志重建）——平时走投影快路径，只在跨压缩时动用 Pi 式重放。

---

## 4. 上下文、压缩与记忆：最小 prompt 和"做梦"

**System prompt 只有 4.6KB**（templates/prompt.md，实测 4,638 字节 45 行），比 Pi 的 2.3KB 大一倍、比 Codex 的 11–21KB 小一个量级。环境信息（git 状态、项目布局）全部下放到**首条 user 消息前缀**，用户请求包 `<user_query>` 标签；工具列表也不进 system prompt。per-model 分化只有两档：换个身份名（`system_prompt_label`，五级解析，默认 "Grok"），或整个换成 **concise 两句话版**（切换模型时就地改写会话里的 System item，model_switch.rs:83-95）。

**但这个 prompt 在二进制里是 XOR 混淆的**。明文模板就在仓库里，运行时却用 `prompt_encrypted.rs` 的加密字节（文件头注释："XOR-encrypted prompt templates (key = position-dependent seed)"），解密到 `Zeroizing<String>` 用完清零——防的是对发行二进制 `strings` 提取，不防读源码。开源仓库里加密自己已公开的 prompt，姿态耐人寻味。

**没有 GROK.md**。项目约定文件识别名单（compat.rs:401-415）：`Agents.md / Claude.md / CLAUDE.md / CLAUDE.local.md / AGENT.md / AGENTS.md`，外加 `.claude/CLAUDE.md`——**自家品牌反而没有专属文件名**。rules 目录认 `.grok/rules` + `.claude/rules` + `.cursor/rules`。注入是全文 verbatim 无截断，且压缩后 AGENTS.md **原文重注入**、不依赖摘要模型转述（assemble.rs:73-79）。

**压缩是第四种切点哲学**。第 04 章记录了三家：Pi 白名单切点向新挪、LangChain 向旧回溯、Codex 服务端。Grok Build 是客户端 full-replace，切点**钉死在语义锚点——最后一条真实 user 消息**（compaction_utils.rs:581-582），之后的消息原样保留，之前的全部换成 9 节结构化摘要；放不下再走 "verbatim → fitted → lossy" 降级梯子。两个亮点：

- **prefire 两段式预压缩**：用量到阈值−10% 就在后台把前 95% 历史先摘要成 NOTE₁ 缓存（带前缀指纹，rewind 即失效），真到 85% 触发时只需"NOTE₁ + 5% 尾巴"重写——把用户可感知的压缩延迟藏进后台（two_pass.rs:1-13）。四家里唯一对压缩延迟做工程优化的。
- 摘要 prompt 里有一段独有的防御："If the prior conversation contains a note about files at /tmp/compaction/segment_*.md … those files are an out-of-band memory channel for a FUTURE work agent, not for you."——防摘要模型自己去读压缩残档。
- 顺带一条：压缩摘要 prompt 模板里出现了 `grok-4.20` 这个未发布型号名（chat 侧专用压缩模型配置，intra_compaction/config.rs:208）。

**记忆叫 "dream"，默认关**（`--experimental-memory` 门控）。布局和 Claude Code 的 auto-memory 神似——连 `MEMORY.md` 的文件名都一样，按 workspace 用 blake3(cwd) 分目录。写入三条路：会话结束的零成本元数据摘要（不调 LLM）、压缩前的 LLM flush turn（写入前做 embedding 余弦去重，阈值 0.92）、以及**睡眠整理**：距上次 ≥4 小时且新增 ≥3 个会话日志时，把日志合并进 MEMORY.md，DREAM_SYSTEM_PROMPT 原话要求 "Resolve contradictions — if a recent session disproves an older fact, keep only the current truth"、"Convert relative dates to absolute dates"，没东西可记就回 `NO_REPLY`。遗忘不用 Codex 的引用计数，用"整理即遗忘"组合：dream 消化后删源日志 + 30 天 GC + 检索时间衰减 + 写入去重。读取双路：首轮自动以用户 query 做 FTS5+向量混合检索注入 top-6，加 `memory_search`/`memory_get` 工具供模型主动查——正好是第 09 章"分层召回"的完整实现。

---

## 5. 工具与扩展：寄生在 Claude Code 生态上

**工具系统最坦率的一点写在 README 里**（:135-139）：tool 实现是从 **openai/codex 和 sst/opencode 移植**的（"in-tree source ports"）。不止移植——内置了**三套方言工具集**：主命名空间 `grok_build`（run_terminal_cmd/read_file/search_replace/…约 30 个）、`codex::apply_patch/…`（配 21KB 的 apply_patch prompt，加密常量名就叫 `CODEX_PROMPT_ENC`）、`opencode::bash/read/edit/…`。换个 agent_type 就能让模型说 Codex 方言或 OpenCode 方言。另有实验性 hashline 工具集（`行号:hash→内容` 锚点编辑，锚点过期返回新锚点让模型重试，批量原子生效）。

MCP 工具用 **`search_tool`（BM25 检索）+ `use_tool`（转发）二段式**延迟加载——与 Claude Code 的 ToolSearch 同构。

**没有 Code Mode，它的位置被 Rhai workflow 占了**：模型可提交 Rhai 脚本，但 host 函数只有 `agent()/parallel()/phase()/log()`——脚本编排的是**子 agent**而不是工具（预算默认 128 个 agent、上限 1024，支持断点续跑）。对照 Codex Code Mode（模型写 TS 直接调工具省 round trip），Grok 的赌注是"可编程的多智能体编排"，工具调用仍由每个子 agent 传统方式发。

**钩子是外部进程范式**（第 06 章三分法里与 Codex 同族）：Command（JSON 进 stdin、exit code 2 = deny）+ HTTP webhook 两种 handler，15 个事件变体（14 个规范事件），失败一律 fail-open。能力天花板同 Codex 一档：`PreToolUse` 只能 allow/deny **不能改写 tool input**，`UserPromptSubmit` 连注入上下文都不行，模型调用没有任何包裹点——第 06 章"只有洋葱范式能重试/替换模型调用"的结论第四次成立。最强的是 `Stop` 门：`decision:"block"` + reason 可强迫模型继续干活（Claude Code stop-hook 同款语义）。

**兼容矩阵是战略级的**。hooks.json 结构、事件名（连 Cursor 风格 `beforeShellExecution` 别名都收）、`.claude/skills/`、`.cursor/skills/`、`~/.claude/agents/`、`.claude-plugin/plugin.json`、`.mcp.json`、仓库级 CLAUDE.md 全家桶——全部原生识别，pager 还有 `/import-claude` 命令扫 `~/.claude.json` 一键搬家。第 07 章讲过生态分发的冷启动难题，Grok Build 的答案是**不冷启动，直接寄生**：Claude Code 用户的所有配置资产零成本迁移。skills frontmatter 是 Claude Code 的超集（多了 `when-to-use`/`compatibility` 等字段）。

插件市场 = git 仓库（官方源 `xai-org/plugin-marketplace`），**无签名无审核**，唯一供应链加固是默认关闭的 commit-sha 钉扎；执行面信任靠目录级授信（项目级插件默认只列元数据，hooks/MCP/脚本被阻止直到授信）。

**子 agent**走 `task` 工具 + ACP 子会话：内置 general-purpose/explore/plan 三型，用户自定义 agent 是 `.md` + frontmatter（也认 `~/.claude/agents/`），**默认后台运行**（`run_in_background` 默认 true，这和另外三家都不同）、嵌套深度默认 1、`isolation: worktree` 可选。上下文引导三种：全新会话（默认）、Forked（父会话史规范化成一条 `<background_context>`，最多 3 个完整 turn 逐字 + 更早的只留统计）、Resumed（继承已完成 peer 的 transcript）。**无 agent 互发消息**——父模型对运行中子 agent 只有 poll/wait/kill。

---

## 6. 沙箱与数据边界：事故在代码里留下的疤

### 6.1 沙箱：两套内核沙箱 + 细到病态的审批分类器

隔离用第三方 crate `nono`（Linux Landlock + macOS Seatbelt，`xai-grok-sandbox/src/lib.rs:14-17`）；**Windows 没有内核沙箱**（feature 门控 `unix`）——对照 Codex 的三套，少了 Windows 那套。进程级网络开放（agent 要调 LLM API），子进程网络仅 Linux 上经 seccomp 封锁。五个内置 profile（默认 `workspace`：全盘可读、workspace 可写、不限网）。

审批层约 2.6 万行（`xai-grok-workspace/src/permission/`）：bash 命令按分号/管道**逐段拆分**逐段判定，无法解析的脚本 fail-closed 转人工；auto 模式分类器能拦 `cat x >> ~/.bashrc`、`git diff --output=~/.bashrc`、`go build -o ~/.bashrc` 这类花式写 dotfile/ssh 的组合（auto_mode/mod.rs:2077-2137 测试锁定），环境注入前缀 `DYLD_`/`GIT_CONFIG` 被拦。最有意思的防提权：**hook 源文件的内核级 write-deny**——沙箱启动前对 Grok 全局 hook 文件做只读保护并复核 symlink/hardlink/dev+ino 身份，检测到可重定向**直接拒绝启动沙箱**（hook_write_deny.rs:36-47，fail-closed）——与 Codex 的 WritableRoot 防 `.git/hooks` 同题异构，且更硬。

### 6.2 数据边界：残留代码找到了，禁用方式是"掏空函数体"

媒体说"上传功能残留代码仍存在但已禁用"。实测残留形态比报道更具体——**内容上传函数全部被编译期存根化**（不是注释、不是 flag、不是服务端开关）：

```rust
// upload/trace.rs:948-959 —— 上传对话消息:函数还在,只剩 skip
upload_turn_messages(...) → skip_artifact(..., "turn_messages.json",
                                          "chat_content_upload_disabled")
// upload/trace.rs:990-993 —— 打包聊天历史:无论传入多少消息,产出空文件
let jsonl = { let _ = messages; Vec::new() };
// upload/trace.rs:439-445 —— 上传完整 prompt 文本:同样只剩 skip
skip_artifact(..., "full_prompt.txt", "prompt_content_upload_disabled")
```

要恢复这些路径必须改源码重编译——在"禁用可信度"的光谱上，这比配置开关硬得多。整改三件套齐全：

1. **内容路径中和**（上表）；仓库变更序列化模块整个被掏空只剩类型 re-export。
2. **新增 workspace 分类器**（`xai-file-utils/src/workspace_classifier.rs`）：`$HOME` 本身、`~/Library`、Desktop/Downloads/Documents、`.ssh`/`.claude`/`.grok` 目录明确排除出上传范围，只有 git 仓库或"项目目录"才可归类上传——直接对应"把家目录整个传了"的事故。
3. **ZDR/opt-out 时主动 purge 本地待传队列**（auth/model.rs:184-189）。

**仍然活着的**是元数据/工件上传（trace upload，四重门控：ZDR → feature 开关 → 凭证 → 目录分类器）：`metadata.json` 含 `repo_root` 路径和 strip 凭证后的 `remote_url`、工具定义、权限决策、用户 prompt 里的图片等。

### 6.3 遥测：与 Codex 互为镜像的两种治理

对照第 15 章的 Codex 六套机制，Grok Build 的形态几乎处处相反：

| | Codex | Grok Build |
|---|---|---|
| 默认 | metrics **默认开**，无需登录 | 全部**默认关**（`TelemetryMode::Disabled`） |
| 端点 | **硬编码在源码**（ab.chatgpt.com + client key） | **构建期 `option_env!` 注入**，OSS 仓库里查不到实际端点（`internal_defaults()` 全 None）；明文的只有 Mixpanel host 和默认 proxy `cli-chat-proxy.grok.com` |
| 仓库信息外泄面 | 模型请求 header 带 cwd 绝对路径 + remote URL，**关不掉** | header 只带 id 类字段;仓库路径/remote URL 在 GCS trace metadata 里，**默认关** |
| 设备标识 | `x-codex-installation-id`（随机持久 UUID） | `x-grok-agent-id` = **硬件派生设备指纹**（macOS 硬件序列号 / Linux machine-id 派生 UUIDv5） |
| 文档披露 | 仓库内 0 命中 | 用户指南专章披露配置项、env、ZDR（docs/user-guide/05, 24） |
| 治理暗门 | 用户配置可覆盖企业设置 | **`remote_settings` 可服务端远程打开遥测和 trace 上传**（config.rs:2340-2367），用户本地不改配置也会被打开（仍受 ZDR/凭证门） |

两边各有一个"最该被挑战的点"：Codex 是"默认开 + 零披露"，Grok 是"默认关，但服务端握着远程开关"——刚出过数据事故的公司在代码里保留 remote kill-switch 的**反向开关**，这个设计值得每个企业买家在采购评审里问一句。出站脱敏倒是做成了集中库（`xai-grok-secrets`，Sentry/Mixpanel 发送前统一调 `redact_secrets`）——第 15 章说"脱敏标准必须集中而不是散装"，这里是正面样本。

---

## 7. 三条总判断

**1. 生态策略：不建生态，寄生生态。**
CLAUDE.md、.claude/skills、~/.claude/agents、hooks.json、.mcp.json、.claude-plugin——Claude Code 用户的全部配置资产在 Grok Build 里开箱即用，还有 `/import-claude` 一键搬家；工具实现直接移植 codex/opencode 并保留方言工具集。第 07 章的分发冷启动难题，它的答案是把竞品的文件约定当成事实标准来实现。这是后发者的理性策略，也意味着 **Claude Code 的配置格式正在成为 coding agent 领域的 POSIX**——四个样本里已有两个（Grok 全面兼容、Codex 部分概念同构）向它收敛。

**2. "开源"和"开放"是两件事，这个仓库是最好的教材。**
源码 100% 可读可自建、内容上传被编译期物理禁用、遥测默认关——透明度是真的。同时:不收 PR、prompt XOR 混淆、发行二进制加控制流混淆、遥测端点构建期注入（你自己 build 的版本发不出遥测，也**验证不了官方二进制发了什么**）、headless 模式强制绑 grok.com、服务端 remote settings 可远程改行为。它开源的是"审计权"，保留的是全部"控制权"。评估任何"开源 agent"时，这份清单可以直接当 checklist。

**3. 事故是最好的架构文档。**
workspace 分类器排除 `$HOME`/`.ssh`、内容上传函数存根化、ZDR purge、出站统一脱敏库、hook 文件内核级 write-deny——这些代码几乎每一行都能对应到事故报道里的一个具体伤口。第 15 章说"数据边界的审计必须按内容做，而不是按代码模块做"，Grok Build 用 5.1GB 的教训把这句话变成了 `workspace_classifier.rs`。反过来看 Codex：还没出过事故，所以 `x-codex-turn-metadata` 里的仓库路径至今随每次请求外发、关不掉。**边界设计的成熟度往往不是远见的函数，是事故的函数。**

---

## 8. 未确认与边界说明

- **事故经过全部来自媒体报道**（DevOps.com、The Decoder、SQ Magazine），未经 xAI 官方逐条确认；仓库是整改后快照、无 git 历史，**无法从源码验证历史版本的上传行为范围**。
- `option_env!` 注入的遥测端点/token/DSN 在官方发行二进制里的实际值，仓库内不可知；官方二进制实际外发行为需抓包验证，本次未做。
- `network_policy.rs` 的网站级出网策略是**未接线的类型建模**，当前不生效（源码注释自认）。
- lsp 工具是否存在降级到 codebase-graph 的远端配置路径、子 agent 并发数硬上限、SSE 配置是否独立传输，三处未逐行确认。
- 五路调查覆盖了主要 crate，但 81 个 member 未逐一排查（如 xai-grok-announcements、xai-gix-status 等外围 crate 未读）。
- 报道称代码规模 "north of a million lines"、另一报道称 844,530 行——与实测 1,547,925 行（含测试）均有出入，口径差异未考证。

**参考来源**（事故报道，第三方）：
[DevOps.com](https://devops.com/xai-open-sources-grok-build-coding-agent-after-cloud-upload-exposes-ssh-keys-repos/) ·
[The Decoder](https://the-decoder.com/xai-open-sources-grok-build-on-github-after-massive-data-breach/) ·
[SQ Magazine](https://sqmagazine.co.uk/xai-open-sources-grok-build/)
