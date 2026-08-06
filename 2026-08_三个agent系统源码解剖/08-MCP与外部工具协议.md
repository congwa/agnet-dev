# 08 · MCP 与外部工具协议

> **一句话导读**：MCP 把「agent 连外部系统」从 N×M 的定制适配变成 1×N 的标准协议，但它把工具 schema 变成了**每轮请求都要重发的常驻税**——Codex 用 OAuth、审批模式、按需启动、`defer_loading` 一整套工程手段把这笔税压下去，LangChain 用一个 adapter 包把 MCP 工具降级成普通 `BaseTool`（"MCP 只是众多工具来源之一"），而 Pi 在 README 里写了两个字：不做。
>
> **本章涉及源码**
> - Pi：`packages/coding-agent/README.md`（Philosophy 章节）、`src/core/skills.ts`、`src/core/extensions/types.ts`、`examples/extensions/kimi-deferred-tools.ts`、`examples/extensions/dynamic-tools.ts`
> - Codex：`codex-rs/config/src/mcp_types.rs`、`codex-rs/codex-mcp/src/{tools.rs,connection_manager.rs,connection_manager/tool_catalog.rs,connection_manager/startup.rs,tool_catalog_cache.rs}`、`codex-rs/rmcp-client/src/`、`codex-rs/core/src/mcp_tool_call.rs`、`codex-rs/core/src/tools/{spec_plan.rs,handlers/mcp.rs,handlers/mcp_resource.rs}`、`codex-rs/core/src/session/turn.rs`、`codex-rs/cli/src/mcp_cmd.rs`、`codex-rs/mcp-server/`、`codex-rs/docs/codex_mcp_interface.md`
> - LangChain v1：`langchain/agents/middleware/provider_tool_search.py`、`langchain/agents/middleware/tool_selection.py`；外部包 `langchain-mcp-adapters`（`langchain_mcp_adapters/tools.py`）

---

## 0. 先搞清楚：这是个什么东西

假设你在造一个 agent，希望它能查 GitHub issue、读 Postgres、控制浏览器。最朴素的做法是给它写三个工具函数：`github_list_issues`、`pg_query`、`browser_click`。每个函数你都要自己定义 schema、自己处理鉴权、自己维护 API 变更。

现在换个人来造 agent，他也要接 GitHub。他得把这套东西**再写一遍**。有 N 个 agent、M 个外部系统，就要写 N×M 份适配代码。这就是 MCP（Model Context Protocol）想解决的问题：定义一套标准协议，让 GitHub 官方写**一次** server，所有 agent 都能连——N×M 降到 N+M。

具体怎么运作？MCP 是 JSON-RPC 2.0 之上的一层约定，双方角色是 **client**（你的 agent harness）和 **server**（提供能力的进程/服务）。握手完成后 client 问 server：`tools/list`，server 回一个数组，每个元素是 `{ name, description, inputSchema }`——注意 `inputSchema` 就是 JSON Schema，形状和上一章讲的 function calling schema 几乎一样。client 把这些 schema 塞进发给模型的 `tools` 数组；模型决定调用时，client 反过来给 server 发 `tools/call`，拿到 `CallToolResult` 再回灌进上下文。

除了 tools，协议还定义了另外两类原语：**resources**（server 暴露的可读数据，用 URI 寻址，`resources/list` + `resources/read`）和 **prompts**（预置的提示词模板）。还有个反向通道叫 **elicitation**：server 可以在执行中反过来问 client 一个问题（"你确定要删这个分支吗？"），由 client 转给用户。

传输层有两种：

- **stdio** —— client 把 server 当子进程 `spawn` 起来，JSON-RPC 消息走 stdin/stdout，一行一条。适合本地工具，天然继承你的文件系统权限，没有网络鉴权问题。
- **Streamable HTTP** —— server 是一个 HTTP 端点，client 发 POST，可以用 SSE 流式回消息。适合远程 SaaS，代价是要处理 bearer token / OAuth。

心智模型可以简化成一句：**MCP 就是"工具定义的包管理器 + 远程过程调用"**。它没有发明任何新的模型交互方式，只是把"harness 从哪儿拿到工具列表"这件事标准化了。

## 1. 为什么这件事很难

**（一）工具声明是每轮都要付的常驻税。**
这是 MCP 最被诟病的一点，也是本章的主线。一个 MCP server 注册几十个工具是常态：Playwright MCP 有 21 个工具、Chrome DevTools MCP 有 26 个。这些工具的 `name + description + inputSchema` 全量序列化后进 `tools` 数组，而 `tools` 数组是**每一轮请求都要重发的**。装三个 server 就可能是几万 token 的固定开销——还没开始干活，上下文已经去掉了 10%。更麻烦的是这笔开销和"你这一轮用不用得上它"完全无关。

**（二）server 是有生命周期的外部进程/服务，不是纯函数。**
stdio server 要 `spawn`、要等握手、要在会话结束时 kill，握手可能超时（server 在 `npm install` 一个依赖），可能崩溃后需要重连。HTTP server 要处理 token 过期、429、网络分区。而这些启动开销发生在**用户敲下回车之后、模型第一次请求之前**——每多等一秒，用户都在盯着光标。

**（三）鉴权的密钥不能落在配置文件里。**
远程 MCP server 要么用 bearer token，要么走 OAuth。token 写进 `config.toml` 会进 git、会被 agent 自己读到。OAuth 则需要一整套浏览器回调、refresh、并发刷新去重、凭据存储（keyring 还是文件？）的基础设施——这跟 agent 本身一点关系都没有，但你不做就接不了任何企业级 server。

**（四）审批粒度要从"整个 server"细到"单个工具"。**
MCP server 是第三方代码。给它 blanket 授权等于给第三方任意执行权。但每次调用都弹窗又会把 agent 变成点击游戏。协议里有个 `annotations.readOnlyHint` 可以声明"我是只读的"，但那是 **server 自己声称的**，harness 该不该信？

**（五）错误要能传播回模型而不是炸掉整个 run。**
server 返回 `isError: true` 时，正确处理是包成一条工具消息喂回去让模型自己改参数；但传输层断连、schema 反序列化失败这类错误则完全不同——把它们混在一起处理，要么该重试的直接崩了，要么该崩的被静默吞掉。

## 2. Pi 的做法

### 2.1 一句话概括

**不做**。README 的 Philosophy 章节明确写着 "No MCP"，替代路径是「写 CLI 工具 + 写 README/SKILL.md，让模型用 `bash` 调」——把工具文档从"schema 常驻"换成"文件按需 read"。

### 2.2 机制拆解

**明确的拒绝声明。** 这不是"还没做"，是设计决策，写在 README 里并附了论证链接。

`packages/coding-agent/README.md:498`

```markdown
**No MCP.** Build CLI tools with READMEs (see [Skills](#skills)), or build an extension
that adds MCP support. [Why?](https://mariozechner.at/posts/2025-11-02-what-if-you-dont-need-mcp/)
```

在仓库里 grep `mcp`，整个 `packages/coding-agent/src/` 只有一处出现，还是 `tool-result-images.ts` 里一句注释提到"MCP bridges 可能返回图片"。**没有任何 MCP 协议实现**。`examples/extensions/` 下 70 多个示例扩展里也没有 MCP 示例——README 说的"build an extension that adds MCP support"是留给用户的路，不是官方提供的路。

**替代方案第一层：CLI + README。** 作者那篇博客的核心论证是量化的：Playwright MCP 的 21 个工具占 13.7k token（Claude 上下文的 6.8%），Chrome DevTools MCP 的 26 个工具占 18.0k token（9.0%）；而他自己写的 Node.js 版浏览器控制脚本 + 一个 README，只有 **225 token**。差了约 60 倍。

关键不在"CLI 比 MCP 快"，而在**文档的加载时机**：MCP 的 schema 是 `tools` 数组的一部分，每轮请求都重发；CLI 工具的 README 是磁盘上一个文件，模型**只在需要时 `read` 一次**，之后它作为一条普通消息留在历史里，不会随轮次翻倍。作者的原话是："When I start a session where the agent needs to interact with a browser, I just tell it to read that file in full and that's all it needs."

附带的好处也很实在：CLI 输出可以重定向到文件、可以管道串联（`tool-a | jq | tool-b` 一条命令完成三步，中间结果根本不进上下文），而 MCP 的每次 `tools/call` 结果都必须走一遍上下文。

**替代方案第二层：Skills。** Skill 是符合 [Agent Skills 标准](https://agentskills.io)的目录，包含一个 `SKILL.md`（frontmatter 里有 `name` / `description`）和任意脚本、参考文档。Pi 启动时扫描所有 skill 位置，**只把 name/description/路径注入系统提示**：

`packages/coding-agent/src/core/skills.ts:335`

```typescript
export function formatSkillsForPrompt(skills: Skill[]): string {
	const visibleSkills = skills.filter((s) => !s.disableModelInvocation);
	if (visibleSkills.length === 0) return "";

	const lines = [
		"\n\nThe following skills provide specialized instructions for specific tasks.",
		"Use the read tool to load a skill's file when the task matches its description.",
		"When a skill file references a relative path, resolve it against the skill directory ...",
		"",
		"<available_skills>",
	];
	for (const skill of visibleSkills) {
		lines.push("  <skill>");
		lines.push(`    <name>${escapeXml(skill.name)}</name>`);
		lines.push(`    <description>${escapeXml(skill.description)}</description>`);
		lines.push(`    <location>${escapeXml(skill.filePath)}</location>`);
		lines.push("  </skill>");
	}
	lines.push("</available_skills>");
	return lines.join("\n");
}
```

三行 XML 一个 skill，常驻成本大约几十 token。`docs/skills.md` 把这个模式直接命名了：**"This is progressive disclosure: only descriptions are always in context, full instructions load on-demand."** 注意这段代码只在 `read` 工具可用时才被调用（`system-prompt.ts:65` 的 `customPromptHasRead && skills.length > 0`）——因为整个机制的前提就是模型能读文件。

同一份文档也诚实地写了这个方案的软肋："models don't always do this; use prompting or `/skill:name` to force it"。**没有 schema 强制，就没有确定性**。这是拿可靠性换上下文。

**逃生舱：扩展可以自己接 MCP。** `ExtensionAPI` 提供了完整的动态工具能力：

`packages/coding-agent/src/core/extensions/types.ts:1331`

```typescript
	/** Get the list of currently active tool names. */
	getActiveTools(): string[];

	/** Get all configured tools with parameter schema, prompt guidelines, and source metadata. */
	getAllTools(): ToolInfo[];

	/** Set the active tools by name. */
	setActiveTools(toolNames: string[]): void;
```

配合 `registerTool()`（`types.ts:1246`，可在 `session_start` 之后任意时刻调用，见 `examples/extensions/dynamic-tools.ts`），写一个 MCP client 扩展在技术上完全可行：连 server、`tools/list`、把每个 MCP tool 转成 `pi.registerTool({ name, description, parameters, execute })`。

更有意思的是 `examples/extensions/kimi-deferred-tools.ts`——它演示了**在 Pi 里手搓 tool search**：

```typescript
	pi.registerTool({
		name: "tool_search",
		label: "Tool Search",
		description: "Find and activate tools for a capability.",
		promptSnippet: "Search for additional tools when the active tools cannot perform the task",
		parameters: Type.Object({ query: Type.String({ description: "Capability to search for" }) }),
		async execute(_toolCallId, params) {
			if (!params.query.toLowerCase().includes("calc")) { /* ... 未命中 ... */ }
			const active = pi.getActiveTools();
			const added = active.includes("Calculator") ? [] : ["Calculator"];
			if (added.length > 0) pi.setActiveTools([...active, ...added]);
			return { content: [{ type: "text", text: "Success. Found 1 matching tool(s)" }], details: { matches: ["Calculator"], added } };
		},
	});

	pi.on("session_start", () => {
		pi.setActiveTools(["tool_search"]);
	});
```

会话开始时激活列表里**只有 `tool_search` 一个工具**，模型发现干不了活时调用它，扩展在运行时把 `Calculator` 加进激活列表。这说明 Pi 核心是有"按需暴露工具"的原语的——只是官方没有拿它去接 MCP。

### 2.3 代价与适用边界

- **没有结构化 schema，模型只能靠文档猜参数。** MCP 工具的 `inputSchema` 有双重作用：给模型看的说明书 + 很多 provider 用它做 constrained decoding，保证生成的参数一定是合法 JSON。CLI 工具两样都没有。模型拼错 flag、漏参数、把 `--json` 写成 `--format json`，只能靠命令报错后重试——多烧一轮往返，而且失败模式是长尾的。
- **没有细粒度审批的挂钩。** MCP 的每个 tool 有名字有注解，harness 可以按工具名配审批策略（Codex 做的正是这个）。CLI 工具统统是 `bash` 的一次调用，你的权限门看到的只是一个字符串 `node /path/browser.js click "#submit"`。要做细粒度控制，得自己在 `beforeToolCall` 里解析命令行——回到了 N×M。
- **生态是零。** 官方 GitHub MCP、Sentry MCP、Linear MCP 这些现成的东西，在 Pi 里一个都用不了，除非你自己给每一个写 CLI 包装。作者自己在博客结尾承认了这一点：*"With great power comes great responsibility though. You will have to come up with a structure for how you build and maintain those tools yourself."*
- **论证本身有个盲区**：博客通篇没有讨论 MCP 在哪些场景仍然更优（标准化的鉴权、server 侧的会话状态、elicitation 这种反向通道），也没有讨论"tool search / 按需加载"这类正在被 harness 采纳的缓解手段。它比较的是"全量加载的 MCP" vs "按需加载的 README"，而不是"按需加载的 MCP" vs "按需加载的 README"。第 5 节会回到这一点。
- **适用边界**：单人 / 小团队、工具是自己写的、可以用 `--help` 和 README 讲清楚、且你愿意为可靠性下降买单。反过来，需要接十几个第三方 SaaS、需要审计每次外部调用、需要 OAuth 的场景，这套方案的边际成本会迅速追上并超过 MCP。

## 3. Codex 的做法

### 3.1 一句话概括

全面拥抱：完整的 client 实现（stdio + Streamable HTTP、bearer token + OAuth + `codex mcp login`）、tools/resources 双原语、四档审批模式、每 server 的工具白/黑名单；同时 Codex **自己也能当 MCP server**（`codex mcp-server`）。对于第 1 节的"常驻税"问题，它的工程回应是三层：**工具过滤 → 按需启动 → `defer_loading` 转 tool search**。

### 3.2 机制拆解

**配置面。** 配置根键是 `[mcp_servers.<name>]`（`config/src/config_toml.rs:260`），单个 server 的字段相当丰富：

`codex-rs/config/src/mcp_types.rs:161`

```rust
pub struct McpServerConfig {
    #[serde(flatten)]
    pub transport: McpServerTransportConfig,
    /// Authentication flow to use when no configured authorization resolves.
    pub auth: McpServerAuth,
    /// When `false`, Codex skips initializing this MCP server.
    pub enabled: bool,
    /// When `true`, `codex exec` exits with an error if this MCP server fails to initialize.
    pub required: bool,
    /// When `true`, every tool from this server is advertised as safe for parallel tool calls.
    pub supports_parallel_tool_calls: bool,
    /// Model-facing surfaces from which this server's tools must be omitted.
    pub omit_tools_from: Option<Vec<ToolExposureSurface>>,
    /// Startup timeout in seconds for initializing MCP server & initially listing tools.
    pub startup_timeout_sec: Option<Duration>,
    /// Default timeout for MCP tool calls initiated via this server.
    pub tool_timeout_sec: Option<Duration>,
    /// Approval mode for tools in this server unless a tool override exists.
    pub default_tools_approval_mode: Option<AppToolApproval>,
    /// Explicit allow-list of tools exposed from this server. When set, only these tools will be registered.
    pub enabled_tools: Option<Vec<String>>,
    /// Explicit deny-list of tools. These tools will be removed after applying `enabled_tools`.
    pub disabled_tools: Option<Vec<String>>,
    pub scopes: Option<Vec<String>>,
    pub oauth: Option<McpServerOAuthConfig>,
    pub oauth_resource: Option<String>,
    /// Per-tool approval settings keyed by tool name.
    pub tools: HashMap<String, McpServerToolConfig>,
    // ...
}
```

几个值得注意的点：`required` 决定失败是软降级还是硬中止（`codex exec` 直接退出）；`supports_parallel_tool_calls` 是**用户对 server 的信任声明**，因为协议里没有可靠的并发安全标记；`tools: HashMap<String, McpServerToolConfig>` 让审批模式可以精确到单个工具名。默认超时在 `codex-mcp/src/rmcp_client.rs:91`：启动 30 秒、工具调用 300 秒。

传输层就是协议规定的两种（`mcp_types.rs:463`）：`Stdio { command, args, env, env_vars, cwd }` 和 `StreamableHttp { url, bearer_token_env_var, http_headers, env_http_headers }`。注意 bearer token 存的是**环境变量名**而不是值——注释写得很清楚："The actual secret value must be provided via the environment."

**第一层降本：工具过滤。** `enabled_tools` / `disabled_tools` 的语义在代码里定义得毫不含糊：

`codex-rs/codex-mcp/src/tools.rs:63`

```rust
/// A tool is allowed to be used if both are true:
/// 1. enabled is None (no allowlist is set) or the tool is explicitly enabled.
/// 2. The tool is not explicitly disabled.
#[derive(Default, Clone)]
pub(crate) struct ToolFilter {
    pub(crate) enabled: Option<HashSet<String>>,
    pub(crate) disabled: HashSet<String>,
}

impl ToolFilter {
    pub(crate) fn allows(&self, tool_name: &str) -> bool {
        if let Some(enabled) = &self.enabled
            && !enabled.contains(tool_name)
        { return false; }
        !self.disabled.contains(tool_name)
    }
}
```

白名单先过、黑名单后过。这是最朴素的手段——**装了 26 个工具的 server，你只留 3 个**。有效，但需要用户手工配置。

同一个文件里还有一个容易被忽略但很重要的问题：**命名冲突和长度限制**。`normalize_tools_for_model_with_prefix()`（`tools.rs:113`）要保证每个模型可见的工具名唯一且 ≤ 64 字节——两个 server 都叫 `search` 怎么办、server 名太长怎么办，它用 sanitize + 必要时 SHA1 哈希来解决，同时把**原始的 `server_name` / `tool.name` 保留在 `ToolInfo` 上**用于回传给 server。这是任何 MCP client 都躲不掉的一层翻译。

**第二层降本：按需启动。** 这是 Codex 对"常驻税"最有意思的回应。每一轮开始时，Codex 先扫一遍用户输入，判断这一轮**到底需要哪些 server**：

`codex-rs/core/src/session/turn.rs:628`

```rust
async fn required_mcp_servers_for_input(
    sess: &Arc<Session>,
    turn_context: &TurnContext,
    user_input: &[UserInput],
) -> (Vec<String>, Vec<crate::plugins::PluginCapabilitySummary>) {
    // ... 1) 用户显式 @ 提到的 plugin 带来的 server
    let mentioned_plugins = collect_explicit_plugin_mentions(user_input, loaded_plugins.capability_summaries());
    let mut required_servers = mentioned_plugins.iter()
        .flat_map(|plugin| plugin.mcp_server_names.iter().cloned())
        .collect::<HashSet<_>>();

    // ... 2) 输入里形如 `mcp://<server>` 的路径引用
    required_servers.extend(paths.filter_map(|path| {
        path.strip_prefix("mcp://").filter(|server| !server.is_empty()).map(str::to_string)
    }));

    // ... 3) 被提到的 skill 在 frontmatter 里声明的 mcp 依赖
    for skill in mentioned_skills {
        if let Some(dependencies) = skill.dependencies {
            required_servers.extend(dependencies.tools.into_iter()
                .filter(|tool| tool.r#type.eq_ignore_ascii_case("mcp"))
                .map(|tool| tool.value));
        }
    }
    (required_servers.into_iter().collect(), mentioned_plugins)
}
```

三个来源：显式 plugin 提及、`mcp://server` 路径引用、skill 声明的 MCP 依赖。这份名单一路传到 `capture_step_context_with_required_mcp_servers()`（`session/mod.rs:3076`），最终决定"这一轮愿意为哪些 server 的启动**阻塞等待**"：

`codex-rs/codex-mcp/src/connection_manager/tool_catalog.rs:186`

```rust
    let required = self.required_servers.binary_search(server_name).is_ok();
    let has_cached_tools = view.connection.client.has_cached_tools();
    let must_wait_for_startup = required
        || self.is_selected_plugin_mcp_server(server_name)
        || required_servers.iter().any(|required| required == server_name)
        || (server_name == CODEX_APPS_MCP_SERVER_NAME && !has_cached_tools);
    if !must_wait_for_startup && has_cached_tools {
        return;                        // 用缓存的工具列表，完全不等
    }
    if !must_wait_for_startup {
        // 只给 1 秒宽限期（OPTIONAL_MCP_STARTUP_GRACE），超时就把这个 server 从本轮目录里摘掉
        if tokio::time::timeout_at(startup_deadline, view.connection.client.client()).await.is_err() {
            trace!(server_name = %server_name, "omitting pending optional MCP server");
        }
        return;
    }
    let _ = view.connection.client.client().await;   // 必需的，无限等到 startup_timeout_sec
```

三档策略：**必需的 server 死等**（超时上限是它自己的 `startup_timeout_sec`）；**可选的 server 有缓存就直接用缓存**（`McpToolCatalogCache`，进程内 LRU，容量 32、TTL 30 分钟，见 `tool_catalog_cache.rs:28`）；**既不必需又没缓存的，只给 1 秒宽限**，还没起来就这一轮不要它了。

这套设计同时解决了两个问题：第一次启动的延迟被压到 1 秒上限，而工具目录缓存让第二次之后连这 1 秒都不用等。代价是**行为不确定**——同一句话在冷启动和热启动时，模型看到的工具集可能不一样。

**第三层降本：`defer_loading` 与 tool search。** 这是最彻底的一层。Codex 的工具注册表里每个工具有一个 `ToolExposure`（`tools/src/tool_executor.rs:51`，六态：`Direct` / `Deferred` / `DeferredModelOnly` / `DirectModelOnly` / `CodeModeOnly` / `Hidden`）。MCP 工具的曝光度由这段逻辑决定：

`codex-rs/core/src/tools/spec_plan.rs:222`

```rust
        exposures = if search_tool_enabled(turn_context)
            && exposures.contains(ToolExposures::DEFERRED)
            && (effective_tool_mode(turn_context) != ToolMode::CodeModeOnly
                || exposures.contains(ToolExposures::CODE_MODE))
        {
            exposures.difference(ToolExposures::DIRECT)      // 开了 tool search：撤掉 DIRECT
        } else {
            exposures.difference(ToolExposures::DEFERRED)    // 没开：撤掉 DEFERRED
        };

        tool.exposure = match (
            exposures.contains(ToolExposures::DIRECT),
            exposures.contains(ToolExposures::DEFERRED),
            exposures.contains(ToolExposures::CODE_MODE),
        ) {
            (false, false, false) => ToolExposure::Hidden,
            (false, false, true) => ToolExposure::CodeModeOnly,
            (true, false, false) => ToolExposure::DirectModelOnly,
            (true, false, true) => ToolExposure::Direct,
            (false, true, false) => ToolExposure::DeferredModelOnly,
            (false, true, true) => ToolExposure::Deferred,
            (true, true, _) => unreachable!("direct and deferred exposure are mutually exclusive"),
        };
```

`DIRECT` 和 `DEFERRED` 被显式设计成互斥（最后那行 `unreachable!` 是断言）。**只要 tool search 开着，MCP 工具就整体从初始工具列表里消失**，改由模型主动搜索发现。落到线上就是 `ResponsesApiTool` 上的 `defer_loading: Some(true)` 字段（`tools/src/responses_api.rs:162`），由 provider 侧按需展开完整 schema。

还有一个兜底的硬限制：agent plugin 来源的 MCP 工具，序列化后超过 `MAX_SERIALIZED_MCP_TOOL_BYTES = 8_000` 字节，就把 `parameters` 整个换成一个 `additionalProperties: true` 的空对象（`responses_api.rs:136`）。宁可让模型丢失参数细节，也不允许单个工具吃掉 8KB。

`omit_tools_from` 配置项正好对应这三个 surface（`Direct` / `Deferred` / `CodeMode`），让用户可以说"这个 server 的工具不要进初始列表，但允许被搜到"。

**审批：四档模式 + 注解。** 审批模式定义在 `mcp_types.rs:25`：

```rust
pub enum AppToolApproval {
    #[default]
    Auto,
    Prompt,
    Writes,
    Approve,
}
```

它们的语义在这个函数里一目了然：

`codex-rs/core/src/mcp_tool_call.rs:2179`

```rust
fn requires_mcp_tool_approval_for_mode(
    annotations: Option<&ToolAnnotations>,
    approval_mode: AppToolApproval,
) -> bool {
    match approval_mode {
        AppToolApproval::Auto => requires_mcp_tool_approval(annotations),
        AppToolApproval::Prompt => true,
        AppToolApproval::Writes => !annotations
            .and_then(|annotations| annotations.read_only_hint)
            .unwrap_or(false),
        AppToolApproval::Approve => false,
    }
}
```

`Prompt` 每次都问、`Approve` 从不问、`Writes` 信任 server 声明的 `read_only_hint`（缺失时按"要审批"处理，fail-closed）、`Auto` 交给内建启发式。这个 `unwrap_or(false)` 的方向选得很对：注解是 server 自己写的，缺失时默认"不是只读"。

审批决策还能记忆：`McpToolApprovalPolicy` 带一个 `allow_persistent` 字段（`mcp_tool_call.rs:1018`），server 级审批可以持久化到配置，而"临时选中的 plugin"的审批用 `for_selected_plugin()` 构造，`allow_persistent: false`——**临时授权不会变成永久授权**。

同一条路径上还串了 hooks（`run_permission_request_hooks`）和一个可选的 "guardian" 自动审查者（`ApprovalsReviewer::AutoReview`），可以让另一个模型来判断这次调用该不该放行。

**鉴权：OAuth 是一等公民。** `rmcp-client/src/oauth/` 下有完整的一套：`refresh_lock.rs`（并发刷新去重）、`refresh_transaction.rs`、`store_lock.rs`、`resolved_store.rs`（keyring 或文件，由 `mcp_oauth_credentials_store_mode` 决定）。CLI 侧是 `codex mcp login <name>`：

`codex-rs/cli/src/mcp_cmd.rs:486`

```rust
async fn run_login(config: &Config, login_args: LoginArgs) -> Result<()> {
    // ...
    let (url, http_headers, env_http_headers) = match &server.transport {
        McpServerTransportConfig::StreamableHttp { url, http_headers, env_http_headers, .. } =>
            (url.clone(), http_headers.clone(), env_http_headers.clone()),
        _ => bail!("OAuth login is only supported for streamable HTTP servers."),
    };
    // 用户没显式给 scopes 时，先去 server 的元数据端点发现支持的 scopes
    let discovered_scopes = if explicit_scopes.is_none() && server.scopes.is_none() {
        discover_supported_scopes(&server.transport, Arc::clone(&http_client),
            OAuthDiscoveryTimeout::LOCAL, StreamableHttpRedirectMode::Legacy).await
    } else { None };
    // ...
    perform_oauth_login_retry_without_scopes(/* ... */).await?;
    println!("Successfully logged in to MCP server '{name}'.");
    Ok(())
}
```

注意函数名里的 `retry_without_scopes`——有些 server 不接受 `scope` 参数，第一次失败后要不带 scope 重试。这类兼容性补丁是"全面拥抱一个新协议"的真实成本。

错误信息也做了针对性处理，`connection_manager/startup.rs:76` 的 `mcp_init_error_display()` 里有三条专门的分支：GitHub Copilot MCP 不支持 OAuth（直接告诉用户去建 personal access token 并给出 `config.toml` 片段）、"Auth required" 时提示跑 `codex mcp login <name>`、超时时提示调大 `startup_timeout_sec` 并把当前值填进去。

**resources：三个独立工具，且只在有 server 时注册。**

`codex-rs/core/src/tools/spec_plan.rs:951`

```rust
fn add_mcp_resource_tools(context: &CoreToolPlanContext<'_>, registry: &mut ToolRegistry) {
    if context.mcp.has_servers() {
        registry.add(ListMcpResourcesHandler);
        registry.add(ListMcpResourceTemplatesHandler);
        registry.add(ReadMcpResourceHandler);
    }
}
```

一行 `if` 省掉三个工具的 schema——没配 MCP server 的用户不为 MCP 付任何 token。`list_mcp_resources` 的参数是 `{ server?, cursor? }`：不给 `server` 就聚合列出所有 server 的资源，给了才能用 `cursor` 翻页（`mcp_resource.rs:88` 明确拒绝"给 cursor 但不给 server"）。

**反向：Codex 自己当 MCP server。** `codex mcp-server` 把 Codex 引擎暴露成一个 MCP server，让别的 agent 能把 Codex 当工具用。它注册的模型可见工具只有两个：`codex`（启动一个会话）和 `codex-reply`（继续会话），定义在 `mcp-server/src/codex_tool_config.rs:106` 和 `:225`：

`codex-rs/mcp-server/src/codex_tool_config.rs:117`

```rust
    Tool::new(
        "codex",
        "Run a Codex session. Accepts configuration parameters matching the Codex Config struct.",
        input_schema,
    )
    .with_title("Codex")
    .with_raw_output_schema(codex_tool_output_schema())
```

调用 `codex` 会返回一个标准 `CallToolResult`，同时把 `threadId` 镜像进 `structuredContent`，客户端拿这个 id 去调 `codex-reply` 继续对话。执行过程中的审批走 MCP 的反向请求（`applyPatchApproval` / `execCommandApproval`，见 `mcp-server/src/{patch_approval,exec_approval}.rs`），客户端回 `{ decision: "allow" | "deny" }`。

`codex-rs/docs/codex_mcp_interface.md:5` 第一句就是：

```markdown
- Status: experimental and subject to change without notice
```

这跟 `codex app-server` 的关系需要说清楚，而且**文档和代码在这里对不上**。`codex_mcp_interface.md` 声称 mcp-server 暴露 `thread/*` `turn/*` `account/*` `config/*` 一整套 v2 RPC，类型来自 `app-server-protocol/src/protocol/{common,v1,v2}.rs`。但 `mcp-server/Cargo.toml` 里**根本没有依赖 `codex-app-server-protocol` 或 `codex-app-server`**（它直接依赖 `codex-core` + `rmcp`），`message_processor.rs` 的方法分发里也 grep 不到任何 `thread/` 或 `turn/` 前缀——未识别的方法统一走 `handle_unsupported_request()`（`message_processor.rs:518`）。

实际形态是：`mcp-server` 是一个**独立的、直接架在 `codex-core` 上的 MCP server**，只实现标准 MCP 方法（`initialize` / `tools/list` / `tools/call` / `resources/*` / `prompts/*` …）加上 `codex` / `codex-reply` 两个工具；`app-server` 才是那套 `thread/*` `turn/*` v2 RPC 的宿主，走自己的传输（stdio / websocket / unix socket + 健康探针），供 VS Code 扩展这类富客户端使用。文档很可能是照着 app-server 的能力清单写的，或者描述的是尚未落地的计划。**"experimental" 这个标签在这里是名副其实的。**

### 3.3 代价与适用边界

- **复杂度极高。** `codex-mcp` 一个 crate 就有 30 个源文件，`rmcp-client` 还有 20 多个，光 OAuth 就单独占一个子目录（含并发刷新锁、事务、存储锁）。这是"要接企业级远程 server"必须付的价，但它把 harness 的核心复杂度提高了整整一个量级。
- **行为不确定性。** 1 秒宽限期 + LRU 工具缓存 + tool search，意味着同一句 prompt 在不同时刻可能面对不同的工具集。调试时很难复现。
- **`supports_parallel_tool_calls` 把安全判断推给了用户。** 默认关，但一旦开了，这个 server 的**所有**工具都被当作并发安全。协议没有提供可靠的信号（`read_only_hint` 是 server 自称的），所以只能靠人。
- **审批模式再细也是"调用前"的粒度。** 你可以决定某个工具要不要弹窗，但看不到它内部真正做了什么——`github__create_issue` 弹窗时你看到的是参数，不是它会发出的 HTTP 请求。
- **适用边界**：需要接多个第三方 server、有企业鉴权要求、需要审计每次外部调用的场景。个人用两三个本地 stdio server 的话，这套机制的绝大部分你都用不上。

## 4. LangChain v1 的做法

### 4.1 一句话概括

**中立管道**：核心库里一行 MCP 代码都没有，MCP 支持放在独立的 `langchain-mcp-adapters` 包里，作用是把 MCP tool 转成普通的 `BaseTool`——转完之后 `create_agent` 看不出它和一个 `@tool` 装饰的本地函数有任何区别。

### 4.2 机制拆解

**核心库确实是干净的。** 在 `/tmp/langchain/libs/langchain_v1/` 和 `libs/core/` 里 grep `mcp`，只能命中测试文件和几处 provider 消息格式转换里的注释——**没有任何 MCP 协议实现**。这跟 Pi 的"不做"是两种不同的"没有"：Pi 是产品层面拒绝，LangChain 是框架层面**不认为这是框架的职责**。工具从哪儿来（本地函数、MCP、provider 托管的服务端工具）对 `create_agent` 来说无所谓，它只认 `BaseTool`。

**适配器的全部工作就是一次类型转换。** `langchain-mcp-adapters` 的核心函数：

`langchain_mcp_adapters/tools.py:357`

```python
def convert_mcp_tool_to_langchain_tool(
    session: ClientSession | None,
    tool: MCPTool,
    *,
    connection: Connection | None = None,
    server_name: str | None = None,
    tool_name_prefix: bool = False,
    handle_tool_errors: bool = True,
    # ...
) -> BaseTool:
    async def call_tool(runtime=None, **arguments):
        # ... 走 interceptor 链，最终 session.call_tool(tool_name, tool_args)
        return _convert_call_tool_result(call_tool_result)

    lc_tool_name = tool.name
    if tool_name_prefix and server_name:
        lc_tool_name = f"{server_name}_{tool.name}"

    error_handler = _handle_mcp_tool_error if handle_tool_errors else False
    return StructuredTool(
        name=lc_tool_name,
        description=tool.description or "",
        args_schema=tool.inputSchema,      # MCP 的 inputSchema 直接就是 args_schema
        coroutine=call_tool,
        response_format="content_and_artifact",
        metadata=metadata,
        handle_tool_error=error_handler,
    )
```

最关键的一行是 `args_schema=tool.inputSchema`——MCP 的 inputSchema 本来就是 JSON Schema，LangChain 的 `StructuredTool` 也接受 JSON Schema，**中间没有任何翻译**。`response_format="content_and_artifact"` 让 MCP 的 `structuredContent` 能以 artifact 形式带出来而不占模型上下文。

`tool_name_prefix` 是它对命名冲突的处理——比 Codex 的 sanitize + SHA1 简陋得多，纯粹靠 `f"{server_name}_{tool.name}"`，也不做长度检查（很多 provider 有 64 字符上限）。

错误处理的默认值和它的 docstring 值得引用：

```
handle_tool_errors: If `True` (default), an MCP tool execution error
    (`CallToolResult(isError=True)`) is returned to the model as a
    `ToolMessage` with `status="error"` so the agent can self-correct
    instead of crashing the run. If `False`, a `ToolException` is raised
    instead (legacy behavior). Transport/session failures and
    content-conversion errors (e.g. unsupported audio content) always
    raise regardless of this setting; only MCP execution errors
    (`isError=True`) are governed by it.
```

这正是第 1 节张力（五）的标准答案：**server 说"我执行失败了"回灌给模型让它自己改，传输层断了就抛异常终止**。两类错误严格分开。

**多 server 客户端。** `MultiServerMCPClient` 用一个 dict 配置多个 server，`get_tools()` 一次性拿到扁平的工具列表：

```python
client = MultiServerMCPClient({
    "math":    {"command": "python", "args": ["/path/to/math_server.py"], "transport": "stdio"},
    "weather": {"url": "http://localhost:8000/mcp", "transport": "http"},
})
tools = await client.get_tools()
agent = create_agent("openai:gpt-4.1", tools)
```

支持 `stdio` / `http`（streamable HTTP）/ `sse` / `websocket` 四种传输。注意 `create_agent(model, tools)` 这行——**MCP 工具和本地工具混在同一个 list 里，没有任何区别对待**。这就是"中立管道"的完整含义。

**对"常驻税"的回应：middleware 层，与 MCP 无关。** LangChain 没有在 MCP 层做任何按需加载，因为它根本不知道哪个工具来自 MCP。缓解手段放在通用的 middleware 层，两条路：

`langchain/agents/middleware/provider_tool_search.py:47`

```python
_SERVER_TOOL_SEARCH_TOOLS: dict[str, _ServerToolSearchSpec] = {
    "anthropic": {
        "type": "tool_search_tool_bm25_20251119",
        "name": "tool_search_tool_bm25",
    },
    "openai": {"type": "tool_search"},
}
```

`ProviderToolSearchMiddleware` 把指定工具打上 `extras["defer_loading"] = True`，再往 `tools` 里追加 provider 原生的 tool search 工具——和 Codex 用的是**同一个 provider 侧机制**，只是 Codex 自己实现了 BM25 索引和 fallback，LangChain 直接委托给 provider。它的 docstring 把限制写得很直白：

```
!!! warning

    This relies on provider-native tool search and only takes effect for
    supported providers. If a tool is deferred but the model's provider
    cannot be identified or does not support tool search, the model call
    raises `ValueError`.
```

另一条路是 `LLMToolSelectorMiddleware`（`tool_selection.py:123`）：在主模型调用前，用一个小模型从 N 个工具里挑出 ≤ `max_tools` 个，然后 `request.override(tools=...)`。它还带了 `transformers = (InternalCallTransformer,)`，把筛选那次调用的 token 从 `run.messages` 里摘掉，免得"省 token"这件事本身在烧 token。这条路不依赖 provider 支持，代价是每轮多一次模型调用。

### 4.3 代价与适用边界

- **鉴权、生命周期、审批全归你。** `MultiServerMCPClient` 的 HTTP 配置只有 `url` 和 `headers`——**没有 OAuth**。要接需要 OAuth 的 server，你得自己跑完流程、把 token 塞进 header、自己管刷新。stdio server 的进程生命周期也要自己在 `async with` 里管好，没有 Codex 那种"1 秒宽限、失败降级、缓存目录"的托底。
- **没有审批层。** MCP 工具转成 `BaseTool` 之后就是普通工具，要审批得自己叠 `HumanInTheLoopMiddleware`，而那是按工具名配的，不认识"MCP server"这个概念。
- **命名冲突处理简陋。** `f"{server_name}_{tool.name}"` 加不加还得你手动开 `tool_name_prefix`，没有长度截断和哈希兜底。接多个 server 时踩到 provider 的工具名长度限制是迟早的事。
- **但这个定位是对的。** LangChain 是框架，用户造的 agent 可能压根不用 MCP，把协议实现塞进核心库只会让所有人为少数人的需求付版本依赖的代价。放在独立包里、依赖官方 `mcp` SDK、只做一层 30 行的类型转换——这是框架该有的克制。
- **适用边界**：你在造一个应用型 agent，MCP 只是若干工具来源之一，且你有能力自己处理鉴权和进程管理。

## 5. 三方横向对比

| 维度 | Pi | Codex | LangChain v1 |
|---|---|---|---|
| 态度 | 明确拒绝（README Philosophy 一条） | 全面拥抱，client + server 双向 | 中立管道，核心库零 MCP 代码 |
| 协议实现位置 | 无（可由扩展自建） | `codex-mcp` + `rmcp-client` 两个 crate，50+ 源文件 | 独立包 `langchain-mcp-adapters`，约 600 行 |
| 传输 | — | stdio / Streamable HTTP | stdio / http / sse / websocket |
| 鉴权 | — | bearer token（环境变量名）+ 完整 OAuth（`codex mcp login`、keyring/文件存储、并发刷新锁、scope 发现、无 scope 重试） | 只有静态 headers，OAuth 自己做 |
| 工具过滤 | — | `enabled_tools`（白）/ `disabled_tools`（黑），白先黑后 | 无，`get_tools()` 全量 |
| 常驻税缓解 | 换掉了问题：CLI + README + Skills，文档按需 `read` | 三层：工具过滤 → 按需启动（1 秒宽限 + LRU 目录缓存）→ `defer_loading` 转 tool search | 通用 middleware：`ProviderToolSearchMiddleware` / `LLMToolSelectorMiddleware`，不感知 MCP |
| server 生命周期 | — | 必需死等 / 可选 1 秒宽限 / 有缓存直接用；`required=true` 时 `codex exec` 硬失败 | 用户自己 `async with` |
| 审批粒度 | 无（只能在 `beforeToolCall` 里解析 bash 命令行） | server 级 + 工具级四档（`Auto`/`Prompt`/`Writes`/`Approve`），临时 plugin 授权不可持久化 | 无（可叠 `HumanInTheLoopMiddleware`，但不认识 server 概念） |
| 命名冲突 | — | sanitize + 必要时 SHA1，保证 ≤ 64 字节且唯一；原始名另存 | 可选 `f"{server}_{tool}"` 前缀，无长度处理 |
| 错误传播 | — | `FunctionCallError::RespondToModel` vs `::Fatal` | `isError=true` → `ToolMessage(status="error")`；传输/转换错误始终抛 |
| resources 支持 | — | `list_mcp_resources` / `list_mcp_resource_templates` / `read_mcp_resource`，仅在有 server 时注册 | `load_mcp_prompt` 有，resources 支持见文档（本章未逐行核对） |
| 反向做 MCP server | — | `codex mcp-server`：独立架在 `codex-core` 上，只暴露 `codex` / `codex-reply` 两个工具 + 标准 MCP 方法，审批走反向请求；标 experimental | — |
| 单工具 schema 上限 | — | 8000 字节（超限则把 `parameters` 换成空对象） | 无 |

## 6. 可以拿走的工程经验

1. **把"工具目录"和"工具启动"解耦，并给可选依赖设一个极短的宽限期。** Codex 的 `OPTIONAL_MCP_STARTUP_GRACE = 1 秒` 是个很好的默认：必需的死等、可选的等 1 秒、有缓存的直接用缓存。适用条件是你能区分"这一轮必需"和"锦上添花"——Codex 靠的是扫用户输入里的 `@plugin` 提及、`mcp://` 路径和 skill 的依赖声明。做不到这个区分的话，这套机制会退化成"随机丢工具"。
2. **信任声明要 fail-closed，而且要区分"谁声明的"。** `AppToolApproval::Writes` 读 server 自称的 `read_only_hint`，但 `.unwrap_or(false)`——缺失时按"需要审批"处理。同理，`supports_parallel_tool_calls` 是**用户**在配置里声明的而不是 server 自称的，因为并发安全性没法被第三方可信地断言。
3. **临时授权不要变成永久授权。** `McpToolApprovalPolicy::for_selected_plugin()` 硬编码 `allow_persistent: false`。用户为了完成一个任务临时启用某个 plugin 时按下的"允许"，不应该被写进全局配置。这是一行代码的事，但漏掉它就是安全事故。
4. **错误分两类，且在类型系统里分开。** "工具执行失败"（`isError: true`、命令 exit code 非 0）要回灌给模型自我修复；"传输失败 / 反序列化失败"要往上抛终止 run。LangChain 的 adapter 把这条规则直接写进了 `handle_tool_errors` 的 docstring，Codex 用 `FunctionCallError::RespondToModel` vs `::Fatal` 两个变体在类型层面强制。用一个 `except Exception` 包住一切是最常见的错误。
5. **在协议边界上做名字翻译，并保留原始身份。** MCP 的工具名来自第三方，可能冲突、可能超长、可能含非法字符。Codex 的 `ToolInfo` 同时持有 `server_name` / `tool.name`（回传给 server 用）和 `callable_namespace` / `callable_name`（给模型看，sanitize + 去重 + ≤64 字节）。任何"把外部标识符暴露给模型"的场景都该这么做。
6. **给单个工具的 schema 设字节上限，超限降级而不是报错。** Codex 的 8KB 上限触发时把 `parameters` 换成 `additionalProperties: true` 的空对象——模型丢失参数细节但工具仍然可调用。比"某个 server 的一个畸形工具让整轮请求超长"好得多。
7. **框架该把协议实现放在核心库之外。** LangChain 把 MCP 放进独立包只做一层类型转换，核心库连 `mcp` 都不 import。判断标准是：这个协议是**用户的需求**还是**框架的抽象**？如果只是众多工具来源之一，它就该是插件。

## 7. 本章存疑

- 关于"MCP 的上下文开销"，一个公允的结论是：**问题真实存在，但两边的方案代价不对称**。真实性的最好证据不是那篇博客的 60 倍数字，而是 Codex 自己的代码——一个全面拥抱 MCP 的实现，同时建了三层机制（工具白黑名单、按需启动、`defer_loading` 转 tool search）专门来削减这笔开销，还给单工具加了 8KB 硬上限。没有人会为不存在的问题写这么多代码。但 Pi 的替代方案也在付真实的代价：没有结构化 schema 意味着模型只能靠 `--help` 和 README 猜参数（拼错就多烧一轮往返，且没有 constrained decoding 兜底），没有工具身份意味着做不了按工具的审批和审计，没有协议意味着第三方生态完全用不上。另外那篇博客比较的是"全量加载的 MCP" vs "按需加载的 README"——如果把 tool search / `defer_loading` 也算进来，差距会显著缩小，但代价会转移到 provider 支持度和一次额外的搜索往返上。**两条路都能走通，选哪条取决于你的工具是自己写的还是别人写的。**
- ⚠️ 未确认：`langchain-mcp-adapters` 对 MCP resources 的支持程度。README 提到 `load_mcp_prompt`，`tools.py` 里能看到 `ResourceLink` / `EmbeddedResource` 的 content 类型处理，但是否有独立的 resources 读取 API 本章未逐行核对（该包不在本次分析的 monorepo 内，只通过 GitHub raw 抓了 `tools.py` 和 README）。
- ⚠️ 未确认：Codex 的 tool search 在 MCP 工具上的实际召回质量。`apply_mcp_tool_exposure_policy` 会在 tool search 开启时把 MCP 工具整体从 DIRECT 转成 DEFERRED，这意味着**模型不主动搜就永远看不到它们**。仓库里没有相关的 benchmark，也没看到"搜不到时的兜底"逻辑。
- **`codex_mcp_interface.md` 与代码不一致，已在 3.2 节写明。** 文档说 `codex mcp-server` 暴露 `thread/*` `turn/*` 等 v2 RPC 且类型来自 `app-server-protocol`，但 `mcp-server/Cargo.toml` 不依赖那个 crate，`message_processor.rs` 也没有对应的方法分发。⚠️ 未确认：这是文档滞后于代码重构，还是描述的是计划中的能力。读这一块时以代码为准。
- Pi 的博客论证里没有提到 elicitation（MCP 的 server→client 反向通道）。CLI 工具在这一维度上没有等价物——一个 CLI 脚本没法在执行到一半时问用户一个问题并等回答（除非它自己去抢 TTY）。这个能力差距在本章的对比表里没有体现，因为 Pi 侧根本没有对应项。

## 8. 第四个样本：Grok Build

> 调研时间 2026-08-06，`xai-org/grok-build` commit `a5589e9`；全项目背景见 [17 番外](./17-番外-GrokBuild全项目速览.md)。本节只看 MCP 维度。

**立场：全面拥抱，作为 client；对外暴露自己时不用 MCP，用 ACP。** 在本章"全面拥抱 / adapter 接入 / 明确拒绝"的三分坐标上，Grok 落在 Codex 那一格，但底座选择相反——Codex 自研协议实现，Grok 用官方 rmcp SDK 2.1，还把它隔离在专用 crate 里（`xai-grok-mcp/Cargo.toml:6` 描述原话："Quarantines rmcp + reqwest 0.13 (rmcp 2.1 requires reqwest >= 0.13.2 while the rest of the workspace uses reqwest 0.12) and owns the MCP credential store and OAuth flow orchestrator."）——为了一个依赖的 reqwest 版本冲突单独开检疫区，工程洁癖可见一斑。

**transport 与修补**：stdio / Streamable HTTP / SSE 三种配置级（`servers.rs:856-857, 4356-4366`）。值得注意的是它对上游 SDK 的两处自修：stdio 用自实现的 `SafeTokioChildProcess` 修 rmcp 的 Drop-spawn 问题（`servers.rs:2043-2231`）；HTTP 侧自带指数退避包装修 rmcp SSE 零退避重连（`mcp_http_client.rs`）。用官方 SDK 不等于免维护——协议实现的坑最终还是自己填。

**企业侧三件套齐全**（对照第 1 节列的"接企业 server 的基础设施税"）：OAuth 完整（浏览器流 + 跨进程去重，`src/oauth.rs`），凭据落 `$GROK_HOME/mcp_credentials.json`；配置源含**仓库级 `.mcp.json`（Claude Code 兼容格式）**与 config.toml；工具命名 `server__tool` 双下划线拼接（`servers.rs:42-46`）——注意这里**没有** Codex 那套 sanitize + SHA1 + ≤64 字节的长度处理（未见对应代码，长名/冲突行为未确认）。

**上下文开销的答案与 Codex 同构**：`search_tool`（BM25 检索 MCP 工具）+ `use_tool`（按检索到的 schema 转发）二段式延迟加载（`implementations/search_tool/mod.rs:1`、`use_tool/mod.rs:1-27`），输出侧 `MCP_MAX_OUTPUT_BYTES = 20_000` **按字节**截断（`util/mcp_truncate.rs:34-42`，注释点名"Some agents use MAX_MCP_OUTPUT_TOKENS; we bound by bytes"）。第 7 节说"没有人会为不存在的问题写这么多代码"——第四个样本又写了一遍。

**子 agent 的 MCP 继承是四家里做得最显式的**：spawn 时快照 `SharedMcpPool`，`Arc<McpClient>` 共享传输、工具 map 独立（`servers.rs:928-935`）；agent 定义可声明 `mcp_inheritance: All|None|Named|Except`（`xai-grok-agent/src/config.rs:935-941`）——把"子 agent 能看到哪些外部工具"做成了声明式配置，而不是隐式继承。

**自身不当 MCP server**：CLI 子命令全集里没有 `mcp serve`（`pager/src/app/cli.rs:9-146`），对外暴露走 ACP（`grok agent stdio`）和 WebSocket。这和 Codex 的 `codex mcp-server` 形成对照——Grok 把"被嵌入"这件事整个押在了 ACP 上。两个特殊变体：SDK 进程内 MCP **反向桥**（SDK 宿主进程里 `create_sdk_mcp_server` 定义的 server，agent 经 ACP 扩展方法 `x.ai/mcp/sdk_call` 反向调用，再适配成 rmcp transport，半双工、不支持 server→client 的 sampling/notifications，`acp_transport.rs:1-20`）；以及 `computer-hub-mcp-adapter`——把 MCP server 桥进 xAI 自家的远程工具路由基础设施 Computer Hub（local-shadows-remote 的 `CompoundResolver`，`xai-computer-hub-core/src/lib.rs:1-29`），对应 `grok workspace` 把本地工作区暴露给云端。

本节未确认：SSE 配置最终走独立传输还是统一降级到 Streamable HTTP（refresh 路径把 Http/Sse 同等映射 `HttpConfig`，初始建连路径未逐行核对）；MCP 工具名超长/冲突时的处理；`annotations.readOnlyHint` 是否参与审批决策未追查。
