# CLAUDE.md — ZeroClaw Agent 工程协议

本文文件定义了本仓库中 Claude Code 的默认工作协议。
适用范围：整个仓库。

## 1) 项目快照（优先阅读）

ZeroClaw 是一个以 Rust 为核心的自主代理运行时，针对以下目标进行优化：

- 高性能
- 高效率
- 高稳定性
- 高可扩展性
- 高可持续性
- 高安全性

核心架构采用 trait 驱动和模块化设计。大多数扩展工作应通过实现 trait 并在工厂模块中注册来完成。

关键扩展点：

- `src/providers/traits.rs` (`Provider`) — 模型提供商
- `src/channels/traits.rs` (`Channel`) — 通信渠道
- `src/tools/traits.rs` (`Tool`) — 工具
- `src/memory/traits.rs` (`Memory`) — 内存/记忆
- `src/observability/traits.rs` (`Observer`) — 可观测性
- `src/runtime/traits.rs` (`RuntimeAdapter`) — 运行时适配器
- `src/peripherals/traits.rs` (`Peripheral`) — 硬件外设板（STM32、树莓派 GPIO）

## 2) 深层架构观察（为何需要此协议）

以下代码库现实应驱动每个设计决策：

1. **Trait + 工厂架构是稳定性的支柱**
    - 扩展点设计为显式且可交换的。
    - 大多数功能应通过 trait 实现 + 工厂注册添加，而非跨模块重写。
2. **安全关键面是一等公民且与互联网相邻**
    - `src/gateway/`、`src/security/`、`src/tools/`、`src/runtime/` 具有高爆炸半径。
    - 默认设置已倾向安全优先（配对、绑定安全、限制、密钥处理）；请保持这种方式。
3. **性能和二进制大小是产品目标，而非锦上添花**
    - `Cargo.toml` 发布配置和依赖选择针对大小和确定性进行优化。
    - 便利性依赖和广泛抽象可能会悄悄削弱这些目标。
4. **配置和运行时契约是面向用户的 API**
    - `src/config/schema.rs` 和 CLI 命令实际上是公共接口。
    - 向后兼容性和显式迁移很重要。
5. **项目现在运行在高并发协作模式**
    - CI + 文档治理 + 标签路由是产品交付系统的一部分。
    - PR 吞吐量是设计约束，不仅仅是维护者的不便。

## 3) 工程原则（规范性）

这些原则默认是强制性的。它们不是口号；它们是实施约束。

### 3.1 KISS（保持简单，愚蠢）

**为何重要**：运行时 + 安全行为必须在压力下保持可审计。

要求：

- 优先使用直接的控制流，而非巧妙的元编程。
- 优先使用显式的 match 分支和类型化结构，而非隐藏的动态行为。
- 保持错误路径明显和局部化。

### 3.2 YAGNI（你不会需要它）

**为何重要**：过早的功能会增加攻击面和维护负担。

要求：

- 在没有具体接受用例的情况下，不要添加新的配置键、trait 方法、功能标志或工作流分支。
- 不要引入推测性的"面向未来"的抽象，至少要有一个当前的调用者。
- 保持不支持的路径显式（报错退出），而非添加部分虚假支持。

### 3.3 DRY + 三次法则

**为何重要**：天真的 DRY 可能会在提供商/渠道/工具之间创建脆弱的共享抽象。

要求：

- 当保持清晰度时，重复小型局部逻辑。
- 仅在重复、稳定的模式后提取共享工具（三次法则）。
- 提取时，保留模块边界并避免隐藏耦合。

### 3.4 SRP + ISP（单一职责 + 接口隔离）

**为何重要**：trait 驱动的架构已经编码了子系统边界。

要求：

- 保持每个模块专注于一个关注点。
- 尽可能通过实现现有的窄 trait 来扩展行为。
- 避免混合策略 + 传输 + 存储的"胖接口"和"上帝模块"。

### 3.5 快速失败 + 显式错误

**为何重要**：代理运行时中的静默回退可能创建不安全或昂贵的行为。

要求：

- 优先使用显式的 `bail!`/错误来处理不支持或不安全的状态。
- 永远不要静默扩大权限/能力。
- 当回退是有意和安全的时，记录回退行为。

### 3.6 默认安全 + 最小权限

**为何重要**：网关/工具/运行时可以执行具有现实世界副作用的行为。

要求：

- 访问和暴露边界默认拒绝。
- 永远不要记录秘密、原始令牌或敏感负载。
- 保持网络/文件系统/Shell 范围尽可能窄，除非有明确理由。

### 3.7 确定性 + 可重现性

**为何重要**：可靠的 CI 和低延迟排查取决于确定性行为。

要求：

- 在 CI 敏感路径中优先使用可重现的命令和锁定的依赖行为。
- 保持测试确定性（没有不稳定的时间/网络依赖，除非有防护措施）。
- 确保本地验证命令与 CI 期望一致。

### 3.8 可逆性 + 回滚优先思维

**为何重要**：在高 PR 量下，快速恢复是强制性的。

要求：

- 保持更改易于回滚（小范围，清晰的爆炸半径）。
- 对于有风险的更改，在合并前定义回滚路径。
- 避免阻止安全回滚的混合大型补丁。

## 4) 仓库地图（高层级）

- `src/main.rs` — CLI 入口点和命令路由
- `src/lib.rs` — 模块导出和共享命令枚举
- `src/config/` — schema + 配置加载/合并
- `src/agent/` — 编排循环
- `src/gateway/` — webhook/网关服务器
- `src/security/` — 策略、配对、密钥存储
- `src/memory/` — markdown/sqlite 内存后端 + 嵌入/向量合并
- `src/providers/` — 模型提供商和弹性包装器
- `src/channels/` — Telegram/Discord/Slack 等渠道
- `src/tools/` — 工具执行表面（shell、文件、内存、浏览器）
- `src/peripherals/` — 硬件外设（STM32、树莓派 GPIO）；参见 `docs/hardware-peripherals-design.md`
- `src/runtime/` — 运行时适配器（目前为原生）
- `docs/` — 架构 + 流程文档
- `.github/` — CI、模板、自动化工作流

## 5) 按路径划分的风险等级（审查深度契约）

使用这些等级来决定验证深度和审查严格度。

- **低风险**：仅文档/杂务/测试的更改
- **中风险**：大多数 `src/**` 行为更改，无边界/安全影响
- **高风险**：`src/security/**`、`src/runtime/**`、`src/gateway/**`、`src/tools/**`、`.github/workflows/**`、访问控制边界

不确定时，归类为更高级别。

## 6) 代理工作流程（必需）

1. **写前读**
    - 在编辑之前检查现有模块、工厂连线和相邻测试。
2. **定义范围边界**
    - 每个 PR 一个关注点；避免混合功能+重构+基础设施补丁。
3. **实施最小补丁**
    - 显式应用 KISS/YAGNI/DRY 三次法则。
4. **按风险等级验证**
    - 仅文档：轻量级检查。
    - 代码/有风险更改：完整相关检查和专注场景。
5. **记录影响**
    - 更新文档/PR 说明，包括行为、风险、副作用和回滚。
6. **尊重队列卫生**
    - 如果是堆叠 PR：声明 `Depends on #...`。
    - 如果是替换旧 PR：声明 `Supersedes #...`。

### 6.3 分支 / 提交 / PR 流程（必需）

所有贡献者（人类或代理）必须遵循相同的协作流程：

- 从非 `main` 分支创建和工作。
- 使用清晰、范围的提交消息将更改提交到该分支。
- 打开 PR 到 `main`；不要直接推送到 `main`。
- 等待必需检查和审查结果后再合并。
- 通过 PR 控制合并（根据仓库策略允许使用 squash/rebase/merge）。
- 合并后删除分支是可选的；当有意维护时，允许长期分支。

### 6.4 Worktree 工作流程（多轨道代理工作必需）

使用 Git worktree 安全且可预测地隔离并发代理/人类轨道：

- 每个活动分支/PR 流使用一个 worktree，以避免跨任务污染。
- 保持每个 worktree 在单个分支上；不要在一个 worktree 中混合不相关的编辑。
- 在提交/PR 之前在相应的 worktree 中运行验证命令。
- 按范围清晰命名 worktree（例如：`wt/ci-hardening`、`wt/provider-fix`），并在不再需要时删除过时的 worktree。
- 第 6.3 节的 PR 检查点规则仍然适用于基于 worktree 的开发。

### 6.1 代码命名契约（必需）

除非子系统有更强的现有模式，否则对所有代码更改应用这些命名规则。

- 一致使用 Rust 标准大小写：模块/文件 `snake_case`，类型/trait/枚举 `PascalCase`，函数/变量 `snake_case`，常量/静态 `SCREAMING_SNAKE_CASE`。
- 按领域角色而非实现细节命名类型和模块（例如 `DiscordChannel`、`SecurityPolicy`、`MemoryStore`，而非像 `Manager`/`Helper` 这样的模糊名称）。
- 保持 trait 实现者命名显式和可预测：`<ProviderName>Provider`、`<ChannelName>Channel`、`<ToolName>Tool`、`<BackendName>Memory`。
- 保持工厂注册键稳定、小写和面向用户（例如 `"openai"`、`"discord"`、`"shell"`），并在没有迁移需求时避免别名扩散。
- 按行为/结果命名测试（`<subject>_<expected_behavior>`），并保持 fixture 标识符中立/项目范围。
- 如果在测试/示例中需要类似身份的命名，仅使用 ZeroClaw 原生标签（`ZeroClawAgent`、`zeroclaw_user`、`zeroclaw_node`）。

### 6.2 架构边界契约（必需）

使用这些规则在增长下保持 trait/工厂架构稳定。

- 通过添加 trait 实现 + 工厂连线来扩展能力；避免对隔离功能进行跨模块重写。
- 保持依赖方向向内到契约：具体集成依赖于 trait/config/util 层，而非其他具体集成。
- 避免创建跨子系统耦合（例如提供商代码导入渠道内部，工具代码直接变更网关策略）。
- 保持模块职责单一：编排在 `agent/`，传输在 `channels/`，模型 I/O 在 `providers/`，策略在 `security/`，执行在 `tools/`。
- 仅在重复使用后引入新的共享抽象（三次法则），在当前范围内至少有一个真正的调用者。
- 对于配置/schema 更改，将键视为公共契约：记录默认值、兼容性影响和迁移/回滚路径。

## 7) 更改手册

### 7.1 添加提供商

- 在 `src/providers/` 中实现 `Provider`。
- 在 `src/providers/mod.rs` 工厂中注册。
- 为工厂连线和错误路径添加专注测试。
- 避免提供商特定行为泄漏到共享编排代码中。

### 7.2 添加渠道

- 在 `src/channels/` 中实现 `Channel`。
- 保持 `send`、`listen`、`health_check`、typing 语义一致。
- 用测试覆盖 auth/allowlist/health 行为。

### 7.3 添加工具

- 在 `src/tools/` 中使用严格参数 schema 实现 `Tool`。
- 验证和清理所有输入。
- 返回结构化的 `ToolResult`；避免在运行时路径中出现 panic。

### 5.4 添加外设

- 在 `src/peripherals/` 中实现 `Peripheral`。
- 外设暴露 `tools()` — 每个工具委托给硬件（GPIO、传感器等）。
- 在配置 schema 中注册板类型（如果需要）。
- 参见 `docs/hardware-peripherals-design.md` 了解协议和固件说明。

### 5.5 安全 / 运行时 / 网关更改

- 包括威胁/风险说明和回滚策略。
- 为故障模式和边界添加/更新测试或验证证据。
- 保持可观测性有用但不敏感。
- 对于 `.github/workflows/**` 更改，在 PR 说明中包括 Actions 允许列表影响，并在源更改时更新 `docs/actions-source-policy.md`。

## 8) 验证矩阵

代码更改的默认本地检查：

```bash
cargo fmt --all -- --check
cargo clippy --all-targets -- -D warnings
cargo test
```

首选的本地 PR 前验证路径（推荐，非必需）：

```bash
./dev/ci.sh all
```

注意：

- 当 Docker 可用时，强烈推荐基于本地 Docker 的 CI。
- 如果本地 Docker CI 不可用，贡献者不会被阻止打开 PR；在这种情况下运行最相关的本机检查并记录运行的内容。

按更改类型的其他期望：

- **仅文档/模板**：运行 markdown lint 和相关文档检查。
- **工作流更改**：验证 YAML 语法；在可用时运行工作流 lint/健全性检查。
- **安全/运行时/网关/工具**：包括至少一个边界/故障模式验证。

如果完整检查不切实际，运行最相关的子集并记录跳过的内容和原因。

## 9) 协作和 PR 纪律

- 完全遵循 `.github/pull_request_template.md`（包括副作用/爆炸半径）。
- 保持 PR 描述具体：问题、更改、非目标、风险、回滚。
- 使用约定式提交标题。
- 尽可能偏好小型 PR（`size: XS/S/M`）。
- 欢迎代理辅助的 PR，**但贡献者仍然对理解其代码的作用负责**。

### 9.1 隐私/敏感数据和中性措辞（必需）

将隐私和中立性视为合并门控，而非尽力而为的指南。

- 永远不要在代码、文档、测试、fixtures、快照、日志、示例或提交消息中提交个人或敏感数据。
- 禁止的数据包括（非详尽列表）：真实姓名、个人电子邮件、电话号码、地址、访问令牌、API 密钥、凭据、ID 和私人 URL。
- 使用中立的项目范围占位符（例如：`user_a`、`test_user`、`project_bot`、`example.com`）代替真实身份数据。
- 测试名称/消息/fixtures 必须是非个人化和系统专注的；避免第一人称或特定身份的语言。
- 如果类似身份的上下文不可避免，仅使用 ZeroClaw 范围的角色/标签（例如：`ZeroClawAgent`、`ZeroClawOperator`、`zeroclaw_user`）并避免现实世界的角色。
- 推荐的身份安全命名调色板（在需要类似身份的上下文时使用）：
    - 角色标签：`ZeroClawAgent`、`ZeroClawOperator`、`ZeroClawMaintainer`、`zeroclaw_user`
    - 服务/运行时标签：`zeroclaw_bot`、`zeroclaw_service`、`zeroclaw_runtime`、`zeroclaw_node`
    - 环境标签：`zeroclaw_project`、`zeroclaw_workspace`、`zeroclaw_channel`
- 如果重现外部事件，在提交之前编辑和匿名化所有负载。
- 在推送之前，专门审查 `git diff --cached` 是否有意外敏感字符串和身份泄漏。

### 9.2 被取代 PR 的归属（必需）

当 PR 取代另一贡献者的 PR 并携带实质性的代码或设计决策时，显式保留作者身份。

- 在集成提交消息中，为每个实质上合并的被取代贡献者添加一个 `Co-authored-by: Name <email>` 尾部。
- 使用 GitHub 识别的电子邮件（`<login@users.noreply.github.com>` 或贡献者的验证提交电子邮件），以便正确呈现归属。
- 将尾部放在提交消息末尾的空行之后的各自行上；永远不要将它们编码为转义的 `\\n` 文本。
- 在 PR 正文中，列出被取代的 PR 链接，并简要说明从每个 PR 中合并的内容。
- 如果没有实际合并代码/设计（仅灵感），不要使用 `Co-authored-by`；而是在 PR 说明中给出荣誉。

### 9.3 被取代 PR 的 PR 模板（推荐）

当取代多个 PR 时，使用一致的标题/正文结构以减少审查者的歧义。

- 推荐的标题格式：`feat(<scope>): 统一并取代 #<pr_a>、#<pr_b> [和 #<pr_n>]`
- 如果这只是文档/杂务/元数据，保持相同的取代后缀并使用适当的约定式提交类型。
- 在 PR 正文中，包括以下模板（填写占位符，删除不适用行）：

```md
## 被取代
- #<pr_a> by @<author_a>
- #<pr_b> by @<author_b>
- #<pr_n> by @<author_n>

## 集成范围
- 来自 #<pr_a>：<实质合并的内容>
- 来自 #<pr_b>：<实质合并的内容>
- 来自 #<pr_n>：<实质合并的内容>

## 归属
- 为实质合并的贡献者添加了 Co-authored-by 尾部：是/否
- 如果否，解释原因（例如：没有直接的代码/设计继承）

## 非目标
- <明确列出未携带的内容>

## 风险和回滚
- 风险：<摘要>
- 回滚：<恢复提交/PR 策略>
```

### 9.4 被取代 PR 的提交模板（推荐）

当提交统一或取代先前的 PR 工作时，使用确定性的提交消息布局，以便机器解析和审查者友好。

- 在消息部分之间保持一个空行，在尾部行之前正好有一个空行。
- 将每个尾部放在其各自的行上；不要换行、缩进或编码为转义的 `\n` 文本。
- 为每个实质合并的贡献者添加一个 `Co-authored-by` 尾部，使用 GitHub 识别的电子邮件。
- 如果没有直接继承代码/设计，省略 `Co-authored-by` 并在 PR 正文中解释归属。

```text
feat(<scope>): 统一并取代 #<pr_a>、#<pr_b> [和 #<pr_n>]

<集成结果的单段摘要>

被取代：
- #<pr_a> by @<author_a>
- #<pr_b> by @<author_b>
- #<pr_n> by @<author_n>

集成范围：
- <子系统或功能_a>：来自 #<pr_x>
- <子系统或功能_b>：来自 #<pr_y>

Co-authored-by: <Name A> <login_a@users.noreply.github.com>
Co-authored-by: <Name B> <login_b@users.noreply.github.com>
```

参考文档：

- `CONTRIBUTING.md`
- `docs/pr-workflow.md`
- `docs/reviewer-playbook.md`
- `docs/ci-map.md`
- `docs/actions-source-policy.md`

## 10) 反模式（不要）

- 不要为次要便利性添加重度依赖。
- 不要静默削弱安全策略或访问约束。
- 不要添加推测性的配置/功能标志"以防万一"。
- 不要将大量仅格式化的更改与功能更改混合。
- 不要"顺便"修改不相关的模块。
- 不要在没有明确解释的情况下绕过失败的检查。
- 不要在重构提交中隐藏改变行为的副作用。
- 不要在测试数据、示例、文档或提交中包括个人身份或敏感信息。

## 11) 交接模板（代理 -> 代理 / 维护者）

交接工作时，包括：

1. 更改了什么
2. 未更改什么
3. 运行的验证和结果
4. 剩余风险/未知数
5. 下一步建议操作

## 12) 快速编码护栏

在快速迭代模式下工作时：

- 保持每次迭代可逆（小提交，清晰的回滚）。
- 在实施之前通过代码搜索验证假设。
- 优先使用确定性行为，而非巧妙的捷径。
- 不要在安全敏感路径上"交付并希望"。
- 如果不确定，留下一个带有验证上下文的具体 TODO，而非隐藏的猜测。

---

## 13) 核心模块详解

### 13.1 Agent 模块 (`src/agent/`)

**职责**：代理编排循环，协调 LLM、工具和内存之间的交互。

**核心组件**：

| 文件 | 职责 |
|------|------|
| `agent.rs` | Agent 结构体和构建器，管理配置和状态 |
| `loop_.rs` | 主代理循环，处理消息、工具调用和响应生成 |
| `dispatcher.rs` | 消息分发器，路由用户消息到合适的处理器 |
| `memory_loader.rs` | 内存加载器，检索相关内存上下文 |
| `prompt.rs` | 提示词构建和模板管理 |

**工作流程**：
1. 接收用户消息
2. 加载相关内存上下文
3. 构建 LLM 请求（包含工具 schema）
4. 发送到 Provider
5. 处理响应（文本或工具调用）
6. 执行工具调用
7. 将结果反馈给 LLM
8. 重复直到获得最终答案
9. 存储交互到内存

### 13.2 Providers 模块 (`src/providers/`)

**职责**：LLM 提供商抽象层，统一不同 AI 模型的访问接口。

**支持的提供商**：

| 提供商 | 文件 | 说明 |
|--------|------|------|
| Anthropic | `anthropic.rs` | Claude API 支持 |
| OpenAI | `openai.rs` | GPT 模型支持 |
| Gemini | `gemini.rs` | Google Gemini API |
| Ollama | `ollama.rs` | 本地 Ollama 模型 |
| OpenRouter | `openrouter.rs` | OpenRouter 聚合服务 |
| Compatible | `compatible.rs` | OpenAI 兼容接口 |
| Router | `router.rs` | 提供商路由和负载均衡 |
| Reliable | `reliable.rs` | 弹性包装器，重试和降级 |

**核心 Trait** (`traits.rs`):

```rust
#[async_trait]
pub trait Provider: Send + Sync {
    // 单次聊天（无系统提示）
    async fn simple_chat(&self, message: &str, model: &str, temperature: f64) -> anyhow::Result<String>;

    // 带系统提示的聊天
    async fn chat_with_system(&self, system_prompt: Option<&str>, message: &str, model: &str, temperature: f64) -> anyhow::Result<String>;

    // 多轮对话
    async fn chat_with_history(&self, messages: &[ChatMessage], model: &str, temperature: f64) -> anyhow::Result<String>;

    // 结构化聊天 API（支持工具调用）
    async fn chat(&self, request: ChatRequest<'_>, model: &str, temperature: f64) -> anyhow::Result<ChatResponse>;

    // 是否支持原生工具调用
    fn supports_native_tools(&self) -> bool { false }

    // 预热连接池
    async fn warmup(&self) -> anyhow::Result<()> { Ok(()) }
}
```

**数据结构**：

- `ChatMessage`：对话消息（role: system/user/assistant/tool, content）
- `ChatResponse`：LLM 响应（text, tool_calls）
- `ToolCall`：工具调用请求（id, name, arguments）
- `ConversationMessage`：多轮对话消息枚举

**扩展新提供商**：
1. 实现 `Provider` trait
2. 在 `mod.rs` 工厂中注册
3. 添加配置 schema 支持
4. 编写测试覆盖基本功能

### 13.3 Channels 模块 (`src/channels/`)

**职责**：通信渠道抽象，支持多种消息平台的接入。

**支持的渠道**：

| 渠道 | 文件 | 说明 |
|------|------|------|
| Telegram | `telegram.rs` | Telegram Bot API |
| Discord | `discord.rs` | Discord Bot |
| Slack | `slack.rs` | Slack App |
| WhatsApp | `whatsapp.rs` | WhatsApp Business API |
| Matrix | `matrix.rs` | Matrix 协议 |
| IRC | `irc.rs` | IRC 聊天协议 |
| iMessage | `imessage.rs` | Apple iMessage（macOS） |
| Email | `email_channel.rs` | 邮件渠道 |
| DingTalk | `dingtalk.rs` | 钉钉机器人 |
| Lark | `lark.rs` | 飞书应用 |

**核心 Trait** (`traits.rs`):

```rust
#[async_trait]
pub trait Channel: Send + Sync {
    // 渠道名称
    fn name(&self) -> &str;

    // 发送消息
    async fn send(&self, message: &str, recipient: &str) -> anyhow::Result<()>;

    // 监听消息（长期运行）
    async fn listen(&self, tx: tokio::sync::mpsc::Sender<ChannelMessage>) -> anyhow::Result<()>;

    // 健康检查
    async fn health_check(&self) -> bool { true }

    // 开始输入指示器
    async fn start_typing(&self, recipient: &str) -> anyhow::Result<()> { Ok(()) }

    // 停止输入指示器
    async fn stop_typing(&self, recipient: &str) -> anyhow::Result<()> { Ok(()) }
}
```

**数据结构**：

- `ChannelMessage`：渠道消息（id, sender, content, channel, timestamp）

**扩展新渠道**：
1. 实现 `Channel` trait
2. 在 `mod.rs` 工厂中注册
3. 添加认证和配置支持
4. 实现 send/listen/typing 语义
5. 覆盖 health_check

### 13.4 Tools 模块 (`src/tools/`)

**职责**：工具执行层，为代理提供可调用能力。

**内置工具**：

| 工具 | 文件 | 功能 |
|------|------|------|
| Shell | `shell.rs` | 命令行执行 |
| File Read | `file_read.rs` | 文件读取 |
| File Write | `file_write.rs` | 文件写入 |
| Memory Store | `memory_store.rs` | 存储记忆 |
| Memory Recall | `memory_recall.rs` | 检索记忆 |
| Memory Forget | `memory_forget.rs` | 删除记忆 |
| HTTP Request | `http_request.rs` | HTTP 请求 |
| Browser | `browser.rs` | 浏览器自动化 |
| Browser Open | `browser_open.rs` | 打开浏览器 |
| Screenshot | `screenshot.rs` | 截图 |
| Git Operations | `git_operations.rs` | Git 操作 |
| Schedule | `schedule.rs` | 定时任务 |
| Image Info | `image_info.rs` | 图片信息 |
| Composio | `composio.rs` | Composio 集成 |
| Hardware Board Info | `hardware_board_info.rs` | 硬件板信息 |
| Hardware Memory Read | `hardware_memory_read.rs` | 硬件内存读取 |
| Hardware Memory Map | `hardware_memory_map.rs` | 硬件内存映射 |

**核心 Trait** (`traits.rs`):

```rust
#[async_trait]
pub trait Tool: Send + Sync {
    // 工具名称（用于 LLM 函数调用）
    fn name(&self) -> &str;

    // 人类可读描述
    fn description(&self) -> &str;

    // 参数 JSON schema
    fn parameters_schema(&self) -> serde_json::Value;

    // 执行工具
    async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult>;

    // 获取完整规范
    fn spec(&self) -> ToolSpec { ... }
}
```

**数据结构**：

- `ToolResult`：工具执行结果（success, output, error）
- `ToolSpec`：工具规范（name, description, parameters）

**扩展新工具**：
1. 实现 `Tool` trait
2. 在 `mod.rs` 工厂中注册
3. 定义严格的参数 schema
4. 验证和清理所有输入
5. 返回结构化的 `ToolResult`

### 13.5 Memory 模块 (`src/memory/`)

**职责**：内存/记忆存储和检索，支持多种后端。

**后端实现**：

| 后端 | 文件 | 说明 |
|------|------|------|
| SQLite | `sqlite.rs` | SQLite 数据库存储 |
| Markdown | `markdown.rs` | Markdown 文件存储 |
| None | `none.rs` | 空实现（无内存） |
| Lucid | `lucid.rs` | Lucid 内存后端 |
| Vector | `vector.rs` | 向量存储 |
| Chunker | `chunker.rs` | 文本分块 |
| Embeddings | `embeddings.rs` | 嵌入生成 |
| Response Cache | `response_cache.rs` | 响应缓存 |
| Snapshot | `snapshot.rs` | 内存快照 |
| Hygiene | `hygiene.rs` | 内存清理 |
| Backend | `backend.rs` | 后端抽象 |

**核心 Trait** (`traits.rs`):

```rust
#[async_trait]
pub trait Memory: Send + Sync {
    // 后端名称
    fn name(&self) -> &str;

    // 存储记忆条目
    async fn store(&self, key: &str, content: &str, category: MemoryCategory) -> anyhow::Result<()>;

    // 检索匹配查询的记忆（关键词搜索）
    async fn recall(&self, query: &str, limit: usize) -> anyhow::Result<Vec<MemoryEntry>>;

    // 获取特定记忆
    async fn get(&self, key: &str) -> anyhow::Result<Option<MemoryEntry>>;

    // 列出所有记忆键（可选按类别过滤）
    async fn list(&self, category: Option<&MemoryCategory>) -> anyhow::Result<Vec<MemoryEntry>>;

    // 删除记忆
    async fn forget(&self, key: &str) -> anyhow::Result<bool>;

    // 计数总记忆数
    async fn count(&self) -> anyhow::Result<usize>;

    // 健康检查
    async fn health_check(&self) -> bool;
}
```

**数据结构**：

- `MemoryEntry`：记忆条目（id, key, content, category, timestamp, session_id, score）
- `MemoryCategory`：记忆类别（Core/Daily/Conversation/Custom）

### 13.6 Observability 模块 (`src/observability/`)

**职责**：可观测性系统，记录事件和指标。

**核心 Trait** (`traits.rs`):

```rust
pub trait Observer: Send + Sync + 'static {
    // 记录离散事件
    fn record_event(&self, event: &ObserverEvent);

    // 记录数值指标
    fn record_metric(&self, metric: &ObserverMetric);

    // 刷新缓冲数据
    fn flush(&self) {}

    // 人类可读名称
    fn name(&self) -> &str;

    // 向下转型为 Any
    fn as_any(&self) -> &dyn std::any::Any where Self: Sized { self }
}
```

**事件类型** (`ObserverEvent`)：

- `AgentStart`：代理启动
- `LlmRequest`：LLM 请求（发送前）
- `LlmResponse`：LLM 响应结果
- `AgentEnd`：代理结束
- `ToolCallStart`：工具调用开始
- `ToolCall`：工具调用完成
- `TurnComplete`：回合完成
- `ChannelMessage`：渠道消息
- `HeartbeatTick`：心跳 tick
- `Error`：错误事件

**指标类型** (`ObserverMetric`)：

- `RequestLatency(Duration)`：请求延迟
- `TokensUsed(u64)`：使用的 token 数
- `ActiveSessions(u64)`：活跃会话数
- `QueueDepth(u64)`：队列深度

### 13.7 Security 模块 (`src/security/`)

**职责**：安全策略、沙箱隔离和密钥管理。

**子模块**：

| 子模块 | 文件 | 功能 |
|--------|------|------|
| Policy | `policy.rs` | 安全策略定义和执行 |
| Pairing | `pairing.rs` | 设备配对机制 |
| Audit | `audit.rs` | 审计日志 |
| Detect | `detect.rs` | 平台检测 |
| Bubblewrap | `bubblewrap.rs` | Bubblewrap 沙箱 |
| Docker | `docker.rs` | Docker 沙箱 |
| Firejail | `firejail.rs` | Firejail 沙箱 |
| Landlock | `landlock.rs` | Landlock 沙箱 |

**核心 Trait** (`traits.rs` - Sandbox):

```rust
#[async_trait]
pub trait Sandbox: Send + Sync {
    // 用沙箱保护包装命令
    fn wrap_command(&self, cmd: &mut Command) -> std::io::Result<()>;

    // 检查此沙箱后端在当前平台是否可用
    fn is_available(&self) -> bool;

    // 人类可读的沙箱后端名称
    fn name(&self) -> &str;

    // 沙箱提供内容的描述
    fn description(&self) -> &str;
}
```

### 13.8 Runtime 模块 (`src/runtime/`)

**职责**：运行时适配器，抽象平台差异。

**核心 Trait** (`traits.rs` - RuntimeAdapter):

```rust
pub trait RuntimeAdapter: Send + Sync {
    // 人类可读的运行时名称
    fn name(&self) -> &str;

    // 是否支持 Shell 访问
    fn has_shell_access(&self) -> bool;

    // 是否支持文件系统访问
    fn has_filesystem_access(&self) -> bool;

    // 基础存储路径
    fn storage_path(&self) -> PathBuf;

    // 是否支持长期运行进程
    fn supports_long_running(&self) -> bool;

    // 最大内存预算（字节）
    fn memory_budget(&self) -> u64 { 0 }

    // 构建此运行时的 Shell 命令
    fn build_shell_command(&self, command: &str, workspace_dir: &Path) -> anyhow::Result<tokio::process::Command>;
}
```

**支持的运行时**：
- Native：原生运行时
- Docker（未来）
- Cloudflare Workers（未来）

### 13.9 Peripherals 模块 (`src/peripherals/`)

**职责**：硬件外设抽象，连接物理设备（STM32、树莓派等）。

**核心 Trait** (`traits.rs` - Peripheral):

```rust
#[async_trait]
pub trait Peripheral: Send + Sync {
    // 人类可读的外设名称（如 "nucleo-f401re-0"）
    fn name(&self) -> &str;

    // 板类型标识符（如 "nucleo-f401re"、"rpi-gpio"）
    fn board_type(&self) -> &str;

    // 连接到外设（打开串口、初始化 GPIO 等）
    async fn connect(&mut self) -> anyhow::Result<()>;

    // 断开连接并释放资源
    async fn disconnect(&mut self) -> anyhow::Result<()>;

    // 检查外设是否可达和响应
    async fn health_check(&self) -> bool;

    // 此外设提供的工具（如 gpio_read、gpio_write、sensor_read）
    fn tools(&self) -> Vec<Box<dyn Tool>>;
}
```

**支持的板类型**：
- Nucleo-F401RE（STM32）
- 树莓派 GPIO（本地）
- Arduino Uno
- ESP32

### 13.10 Config 模块 (`src/config/`)

**职责**：配置 schema 和加载/合并逻辑。

**核心组件**：

- `schema.rs`：配置结构定义
- `mod.rs`：配置加载和合并

**配置类型**：
- 基础配置：模型、渠道、工具设置
- 安全配置：策略、沙箱、密钥
- 运行时配置：环境、路径、限制

### 13.11 Gateway 模块 (`src/gateway/`)

**职责**：Webhook/网关服务器，处理外部 HTTP 请求。

**功能**：
- HTTP webhook 端点
- 请求验证和路由
- 与代理集成

### 13.12 其他支持模块

| 模块 | 职责 |
|------|------|
| `cost/` | 成本追踪和计算 |
| `cron/` | 定时任务调度 |
| `daemon/` | 守护进程管理 |
| `doctor/` | 系统诊断 |
| `hardware/` | 硬件发现和检测 |
| `health/` | 健康检查端点 |
| `heartbeat/` | 心跳监控 |
| `identity/` | 身份和认证 |
| `integrations/` | 第三方集成 |
| `migration/` | 数据迁移 |
| `onboard/` | 入向配置 |
| `rag/` | 检索增强生成 |
| `service/` | 服务管理 |
| `skills/` | 技能系统 |
| `skillforge/` | 技能构建 |
| `tunnel/` | 隧道服务 |
| `util/` | 通用工具函数 |

---

## 14) 工厂模式使用指南

ZeroClaw 使用工厂模式来管理可插拔组件的实现注册。

### 工厂注册模式

每个有多个实现的模块都有一个工厂函数：

```rust
// src/providers/mod.rs
pub fn create_provider(config: &ProviderConfig) -> anyhow::Result<Box<dyn Provider>> {
    match config.provider_type.as_str() {
        "openai" => Ok(Box::new(OpenAIProvider::new(config)?)),
        "anthropic" => Ok(Box::new(AnthropicProvider::new(config)?)),
        "gemini" => Ok(Box::new(GeminiProvider::new(config)?)),
        // ...
        _ => bail!("Unknown provider type: {}", config.provider_type),
    }
}
```

### 注册新实现

1. **实现 trait**：为你的组件实现相应的 trait
2. **添加工厂分支**：在工厂函数中添加新的匹配分支
3. **更新配置 schema**：确保配置支持你的新类型
4. **编写测试**：添加工厂连线和基本功能测试

### 工厂注册键约定

- 使用小写、稳定的标识符
- 保持面向用户（如 "openai"、"discord"、"shell"）
- 避免添加别名，除非有迁移需求
- 在添加/删除键时考虑向后兼容性

---

## 15) CLI 命令结构

主入口点：`src/main.rs`

### 命令类别

| 类别 | 枚举 | 说明 |
|------|------|------|
| 服务管理 | `ServiceCommands` | Install/Start/Stop/Status/Uninstall |
| 渠道管理 | `ChannelCommands` | List/Start/Doctor/Add/Remove |
| 技能管理 | `SkillCommands` | List/Install/Remove |
| 迁移 | `MigrateCommands` | Import from OpenClaw |
| 定时任务 | `CronCommands` | List/Add/Once/Remove/Pause/Resume |
| 集成 | `IntegrationCommands` | Info |
| 硬件发现 | `HardwareCommands` | Discover/Introspect/Info |
| 外设管理 | `PeripheralCommands` | List/Add/Flash/SetupUnoQ/FlashNucleo |

### 添加新命令

1. 在 `src/lib.rs` 中定义命令枚举
2. 在 `src/main.rs` 中添加命令处理逻辑
3. 添加必要的配置和验证
4. 编写命令行为测试

---

## 16) 测试策略

### 测试分层

1. **单元测试**：在 `mod.rs` 的 `#[cfg(test)]` 块中
   - 测试 trait 实现
   - 测试数据结构序列化/反序列化
   - 测试工厂连线

2. **集成测试**：在 `tests/` 目录中
   - 端到端工作流测试
   - 多模块交互测试

3. **属性测试**：使用 `proptest` 进行模糊测试（如适用）

### 测试命名约定

- 函数测试：`<subject>_<expected_behavior>`
- 集成测试：描述性名称，说明场景
- 属性测试：`prop_<property>`

### 测试 Fixture 规则

- 使用中立的项目范围标识符
- 避免真实个人数据
- 使用 `zeroclaw_*` 前缀作为命名空间

---

## 17) 性能考量

### 性能目标

- 低延迟：LLM 请求应快速处理
- 高吞吐：支持并发渠道和工具调用
- 小二进制：发布二进制应保持紧凑
- 低内存：高效内存使用，支持资源受限环境

### 性能优化点

1. **异步 I/O**：所有网络和文件操作使用 `tokio`
2. **连接池**：复用 HTTP 连接（Provider 预热）
3. **缓存**：响应缓存和内存缓存
4. **流式**：支持流式响应（如适用）
5. **并行**：独立的工具调用并行执行

---

## 18) 安全边界

### 高风险路径

以下路径具有高安全影响，需要额外审查：

- `src/security/**`：安全策略和沙箱
- `src/runtime/**`：运行时适配和命令执行
- `src/gateway/**`：网络入口点
- `src/tools/**`：工具执行表面
- `.github/workflows/**`：CI/CD 自动化

### 安全审查清单

- [ ] 输入验证：所有用户输入经过验证
- [ ] 输出清理：敏感数据不泄漏
- [ ] 权限检查：最小权限原则
- [ ] 错误处理：不泄漏敏感信息的错误
- [ ] 日志记录：不记录秘密/令牌
- [ ] 依赖审计：定期审查依赖安全性
- [ ] 沙箱：适当使用沙箱隔离

---

## 19) 文档维护

### 文档更新时机

- 添加新模块或 trait
- 更改公共 API
- 修改配置 schema
- 更改工作流程
- 添加安全边界

### 文档同步

- 修改 CLAUDE.md（英文）时，同步更新 claude_cn.md（中文）
- 保持代码示例与实际代码同步
- 更新相关交叉引用

---

## 20) 故障排查指南

### 常见问题

**问题**：工具调用失败
- 检查工具参数 schema 是否正确
- 验证工具实现是否正确处理输入
- 查看错误日志了解失败原因

**问题**：渠道无法连接
- 验证认证凭据
- 检查网络连接
- 运行 `doctor` 命令诊断

**问题**：内存检索不返回结果
- 确认内存后端已正确配置
- 检查查询格式
- 验证内存已正确存储

### 调试模式

使用环境变量启用调试输出：

```bash
RUST_LOG=debug zeroclaw <command>
```

### 日志位置

- 主日志：控制台输出
- 审计日志：`src/security/audit.rs`
- 错误日志：观察者记录的错误事件

---

*文档版本：2026-02-17*
*适用于 ZeroClaw 主分支*
