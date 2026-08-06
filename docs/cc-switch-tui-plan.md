# cc-switch-cli TUI 方案 (v2)

> 参考 [`farion1231/cc-switch`](https://github.com/farion1231/cc-switch) 的功能模型（Tauri 桌面 app），把它的核心抽象搬到 TUI 形态：**Provider / Preset / Tool**。本仓库不做 Skills、Sessions、Proxy、Tray、Cloud Sync、Deep Link、MCP 的 GUI，只做"管 provider + 切换 + 编辑模型映射"。

## 1. 目标

### 1.1 解决的痛点

1. **不再手写 JSON / TOML**：从 preset 选 → 填表 → 自动生成 live config 落盘。
2. **模型映射可调**：Codex 的 `model` / `model_provider` / `small_fast_model` / `model_reasoning_effort`、Claude 的 `ANTHROPIC_MODEL` / `ANTHROPIC_SMALL_FAST_MODEL` 这些字段在 TUI 表单里直接编辑。
3. 顺带解决 v1 列出的：profile 列表、diff、env 摊平、scope 切换、批量管理。

### 1.2 显式不做的事

- 不做：Skills / Prompts / Sessions / 本地代理 / 托盘 / 云同步 / Deep Link / MCP 管理 / 使用统计。这些是 farion1231/cc-switch 桌面 app 的范畴，TUI 不追这条线。
- 不做：网络请求（speedtest / 拉取 preset 列表），preset 全部本地内置。
- 不引入 SQLite。SSOT 用一份 JSON：`~/.cc-switch/providers.json` + `~/.cc-switch/backups/`。
- 不为 Web 前端 / Tauri 留接口，TUI 即终态。

### 1.3 与现 CLI 的关系

现 `cc-switch-cli` bash 脚本的"profile = JSON 文件"模型与新的 Provider 模型不再 1:1 兼容。处置：

- **新二进制 `cc-switch-cli` 接管所有功能**（CLI + TUI）。
- bash 脚本旧子命令的 plain 输出仍保留字节级兼容，但语义上变成"操作 Claude Code provider"的薄壳——`cc-switch-cli list` 列 Claude Code provider，`cc-switch-cli use <name>` 切换 Claude Code provider。
- 提供 `cc-switch-cli import-legacy`，扫 `~/.claude/profiles/*.json` + `./.claude/profiles/*.json`，按文件名生成 Claude Code provider，一次性迁移。迁完后旧目录不再被读写。
- 旧 `--project` 语义保留：project scope 视作"作用域 = 当前 PWD"，写到 `./.cc-switch/providers.json`（或就地写 `./.claude/...` live config，不维护项目级 SSOT；详见 §6.4）。

## 2. 技术栈

承接 v1 §2 的结论：**Rust + Ratatui + Crossterm + Clap + serde**。理由不重复，新方案需要的额外能力它都覆盖：

- TOML 读写：`toml = "0.8"`（Codex 的 `~/.codex/config.toml`）。
- 表单：`tui-input` 单行 + `tui-textarea` 多行（自由编辑兜底）。
- Diff：`similar`。
- 文件锁：`fs2` 给 SSOT 写盘加排他锁，避免两个 TUI 实例同时写。

## 3. 核心抽象

```rust
/// 用户面对的可切换实体。一个 Provider 配一个工具一份配置。
pub struct Provider {
    pub id: Uuid,              // 内部 id，stable 跨重命名
    pub name: String,          // 显示名，唯一约束 per category
    pub category: ToolKind,    // ClaudeCode | Codex | …
    pub preset_id: Option<String>, // 来自哪个 preset；自定义为 None
    pub endpoint: Option<String>,  // base URL（preset 给默认值，用户可覆盖）
    pub auth: Auth,            // token / api key / oauth ref
    pub model_map: ModelMap,   // 见 §3.2
    pub extras: serde_json::Value, // tool-specific 自由字段（如 Claude 的 hooks）
    pub created_at: SystemTime,
    pub updated_at: SystemTime,
}

pub enum ToolKind { ClaudeCode, Codex }   // P0 只这两个

pub enum Auth {
    BearerToken(SecretString),     // ANTHROPIC_AUTH_TOKEN / OpenAI api key
    ApiKeyEnv { var: String, value: SecretString },
    OAuthFile { path: PathBuf },   // Codex 官方登录沿用其 auth.json
    None,                           // OpenCode 等无凭据的本地工具
}
```

`SecretString` 用 `secrecy` crate，禁止 `Debug`/`Display` 输出明文，渲染时显式 `expose_secret()` + masking。

### 3.1 ToolAdapter trait

每个工具一份适配器，`Tool` 只暴露窄接口：

```rust
pub trait ToolAdapter: Send + Sync {
    fn kind(&self) -> ToolKind;

    /// 把 Provider 渲染为该工具的 live config 文件。
    /// 返回所有要写盘的 (path, bytes)；调用方做原子写。
    fn render(&self, p: &Provider, scope: &Scope) -> Result<Vec<RenderedFile>>;

    /// 反向：读 live config，patch 当前 active provider 的可被回填字段。
    /// 用于"用户绕过 TUI 改了 live 文件，TUI 编辑 active provider 时同步"。
    fn backfill(&self, p: &mut Provider, scope: &Scope) -> Result<()>;

    /// 当前 live 文件指向哪个 provider id（如有标记）。
    fn active(&self, scope: &Scope) -> Result<Option<Uuid>>;

    /// 该工具的"模型映射字段定义"，用于 §5.2 表单。
    fn model_fields(&self) -> &'static [ModelField];
}
```

实现：

- `ClaudeCodeAdapter`：渲染 `<scope>/settings.json`（user → `~/.claude/settings.json`，project → `./.claude/settings.local.json`），保留旧 symlink 语义可选；详见 §6.3。
- `CodexAdapter`：渲染 `~/.codex/config.toml` + `~/.codex/auth.json`。

active 标记：在 live config 里塞一个 `// cc-switch:provider=<uuid>` 注释（TOML 注释 / JSON 用 `_cc_switch` 字段），`active()` 解析它。Backfill 时读出来匹配 SSOT，找不到则当作"外部修改"。

### 3.2 ModelMap

```rust
pub struct ModelMap {
    pub primary: Option<String>,        // 主模型
    pub small_fast: Option<String>,     // 轻量模型
    pub reasoning_effort: Option<String>, // low/medium/high (Codex)
    pub extra: IndexMap<String, String>, // 兜底自由 KV
}

pub struct ModelField {
    pub key: &'static str,
    pub label: &'static str,
    pub kind: ModelFieldKind,           // FreeText | Enum(&'static [&'static str]) | OptionalEnum(&'static [&'static str])
    pub help: &'static str,
}
```

`ClaudeCodeAdapter::model_fields()` 返回 `ANTHROPIC_MODEL` / `ANTHROPIC_SMALL_FAST_MODEL` / `ANTHROPIC_DEFAULT_OPUS_MODEL` / 等等；`CodexAdapter` 返回 `model` / `model_provider` / `model_reasoning_effort` / `small_fast_model` / `disable_response_storage`。

字段定义是数据，写在 `crates/cc-switch-core/data/model_fields.rs`（const 数组），不读外部文件。

### 3.3 Preset

```rust
pub struct Preset {
    pub id: &'static str,        // "anthropic-official" / "packy" / "deepseek" / ...
    pub label: &'static str,
    pub category: ToolKind,
    pub endpoint: Option<&'static str>,
    pub auth_template: AuthTemplate, // 用户要填什么
    pub model_defaults: &'static [(&'static str, &'static str)], // 预填 model_map
    pub notes: Option<&'static str>, // 简短中文提示，例如"需要在控制台开 Claude Code 入口"
}
```

Preset 列表用 const 数组写在 `crates/cc-switch-core/data/presets/`，按 category 分文件。**纯静态，编译时存在**：用户即使离线也能用。Preset 增减 = 改源码 + 发版，不走运行时拉取。

P0 内置 preset（最小集，覆盖痛点 1）：

| id | category | endpoint | 备注 |
|---|---|---|---|
| `anthropic-official` | ClaudeCode | (空) | 官方登录，让用户走 OAuth |
| `claude-code-custom` | ClaudeCode | (空，用户填) | 自定义 token + base URL |
| `packycode` | ClaudeCode | `https://api.packycode.com` | bearer token |
| `deepseek-claude` | ClaudeCode | `https://api.deepseek.com/anthropic` | bearer token |
| `glm-claude` | ClaudeCode | 智谱 endpoint | bearer token |
| `kimi-claude` | ClaudeCode | Moonshot endpoint | bearer token |
| `dmxapi-claude` | ClaudeCode | DMX endpoint | bearer token |
| `openai-official` | Codex | (空) | OAuth via auth.json |
| `codex-custom` | Codex | (空，用户填) | 自定义 model_provider |
| `deepseek-codex` | Codex | DeepSeek endpoint | model_provider 模板 |

P1 再开放 `cc-switch-cli preset import <file.json>` 让用户增 custom preset；存到 `~/.cc-switch/custom-presets.json`。

## 4. 用户入口

```
cc-switch-cli [global flags] <subcommand> [args]

subcommands:
  ui                                 启动 TUI（默认；裸跑 cc-switch-cli 等价 cc-switch-cli ui）
  list [--tool claude|codex|all]    plain 文本列出 provider
  use <name|id> [--tool …]           按名字 / id 切换
  current [--tool …]                 显示当前 active
  add --preset <id> --name <n> …     无 TUI 也能加 provider（脚本场景）
  rm <name|id> [--tool …]
  export <name|id> > file.json       provider 导出（含 token，明文，需用户自负）
  import < file.json                 provider 导入
  import-legacy                      把 ~/.claude/profiles/*.json 一次性迁过来
  path                               打印 SSOT / scope / live config 路径
  pick                               旧 fzf 行为兼容
```

全局选项：

- `--scope user|project`（auto-detect 同 v1：PWD 下有 `.cc-switch/` 或 `.claude/profiles/` 时认 project）
- `--ssot <path>` 显式指定 SSOT 文件（测试用）
- `--no-color` / `--plain`（与 `ui` 互斥）

## 5. 屏幕地图

五个屏幕，`1`-`5` 切换；`?` 全局帮助；`q` 退出（带未保存修改时弹确认）。

```
┌─ cc-switch-cli ─────── tool: Claude Code · scope: user (auto) ─┐
│ [1] Providers  [2] Editor  [3] Models  [4] Diff  [5] Tools │
├──────────────────────────────────────────────────────────────┤
│                          screen body                         │
└── status: active=packy · live=~/.claude/settings.json ──────┘
```

顶部"tool"指示当前正在管理哪个工具（Claude Code / Codex），`t` 在 Tools 屏切换；其他屏只显示当前 tool 的 provider。

### 5.1 Providers 屏

`Table`：列 `*`、name、preset、endpoint（截断）、model（primary）、updated\_at。右侧 `Paragraph` 显示选中 provider 的渲染预览（即 `ToolAdapter::render()` 的输出，masking 后）。

键位：

| 键 | 动作 |
|---|---|
| `Enter` / `s` | switch（对当前 tool 执行 `render()` + 原子写 live config） |
| `n` | new provider → 进 Editor 屏，先弹 preset 选择模态 |
| `e` | 编辑 → Editor 屏（载入选中 provider，先 backfill） |
| `d` | duplicate（新 id，name 加 `-copy`） |
| `r` | rename |
| `x` / `Delete` | delete（active 拒删，提示先 switch 走） |
| `m` | toggle copy/symlink mode（仅 Claude Code 适用，Codex 永远 copy） |
| `/` | 过滤 name |
| `g` / `G` | 列表首/尾 |

### 5.2 Editor 屏（彻底取代 v1 的 JSON Editor）

表单驱动，字段由 `ToolAdapter` + `Preset` 给出。布局示意（Claude Code）：

```
┌─ Editor: packy (Claude Code · preset: packycode) ───────────┐
│ Basic                                                        │
│   Name        [ packy_______________ ]                       │
│   Preset      [ packycode      ▾ ]   ← 切 preset 弹确认      │
│   Endpoint    [ https://api.packycode.com_______ ]           │
│   Token       [ sk-…last4    ] [show] [paste]                │
│                                                              │
│ Models                                                       │
│   ANTHROPIC_MODEL              [ claude-3-5-sonnet-… ]       │
│   ANTHROPIC_SMALL_FAST_MODEL   [ claude-3-5-haiku-…  ]       │
│   + add custom env                                           │
│                                                              │
│ Advanced                                                     │
│   [ ] copy mode (instead of symlink)                         │
│   [ ] include shared snippet (hooks/agents from current)     │
│   [edit raw JSON…]   ← 兜底，进 textarea 自由编辑             │
│                                                              │
│ status: 1 unsaved change                                     │
├──────────────────────────────────────────────────────────────┤
│ Ctrl-S save   Ctrl-V validate   Ctrl-R reset   Esc back     │
└──────────────────────────────────────────────────────────────┘
```

Codex 的 Editor 同结构，Models 段字段不同（见 §3.2），Endpoint 段多一个 `model_provider` 选择器。

行为细节：

- **Token mask**：默认显示 `sk-…last4`，`show` 切显式；剪贴板 `paste` 调系统命令读，按下立即 mask 回去。
- **保存**：`Ctrl-S` → 校验必填字段（name 唯一、token 非空 if preset 要求）→ 写 SSOT (`tempfile + persist`) → 若该 provider 是当前 active，立即重新 `render()` 一次同步到 live config。
- **保存前 backfill 冲突检测**：进入 Editor 屏时 snapshot live 文件 mtime，`Ctrl-S` 时若 mtime 变了且 backfill 出新值，弹"merge?"模态（保留 TUI 改动 / 接受 live / 三方 diff）。
- **raw JSON / TOML 兜底**：`edit raw…` 进 textarea，给高级用户。退出 textarea 时反向解析填回表单；解析失败保留 raw 模式，状态栏给出错误位置。
- **shared snippet**：参考 farion1231 的"Shared Config Panel"。在 Claude Code 下，把当前 active provider 的 `hooks` / `agents` / `mcpServers` / `permissions` 等非凭据字段抽出来作为 snippet，新建 provider 时勾选即注入。snippet 存在 SSOT 顶层 `shared.claude_code` 下。

### 5.3 Models 屏

集中查看 / 调整当前 tool 下所有 provider 的模型映射，表格形式：

```
 Provider     primary                small_fast            reasoning
 packy        claude-3-5-sonnet-…    claude-3-5-haiku-…    -
*deepseek-cc  deepseek-chat          deepseek-chat         -
 official     (default)              (default)             -
```

- 单元格直接编辑（按 `Enter` 进编辑态，`Tab` 移动）。
- 顶部一行预设：`Reset row to preset default`、`Apply column to all rows`。
- 这是用户痛点 #2 的主入口，比逐个进 Editor 更快。

对 Codex，列变成 `model` / `model_provider` / `small_fast_model` / `reasoning_effort`。

### 5.4 Diff 屏

两栏并排，对象层级 diff（不是文本 diff）：

- 选两个 provider，渲染各自结构化字段对比：endpoint / token (mask) / model_map / extras。
- 关键差异（model_map）色块高亮。
- `e` 把右侧某字段 apply 到左侧（写进 Editor buffer，需 `Ctrl-S` 落盘）。

保留 raw JSON/TOML 文本 diff 作为 `Shift-D` 切换视图。

### 5.5 Tools 屏

顶级"我现在在管哪个工具"切换 + 该工具的全局信息：

```
 Tool          live config path                  active provider     status
*Claude Code   ~/.claude/settings.json           packy               ok
 Codex         ~/.codex/config.toml              -                   no provider
```

- `Enter` 切换工具（其他屏的列表跟着换）。
- 在某工具行按 `i` 触发 init（创建 live config 目录、写一份 minimal config）。
- 按 `o` 打开该工具 live config 所在目录（调 `xdg-open`/`open`，缺则提示）。
- 显示"是否被 cc-switch-cli 管理"——live 文件里有没有 `_cc_switch` 标记。

## 6. 数据存储

### 6.1 SSOT

`~/.cc-switch/providers.json`：

```json
{
  "version": 1,
  "shared": { "claude_code": { "hooks": [], "agents": [] } },
  "providers": [
    { "id": "uuid…", "name": "packy", "category": "claude_code" },
    { "id": "uuid…", "name": "official", "category": "codex" }
  ]
}
```

- 写盘：`tempfile::NamedTempFile::new_in("~/.cc-switch")` → `persist(providers.json)`，原子。
- 写入前 `fs2` 排他锁 `providers.json.lock`，避免两个 TUI 并发写。
- 每次写入前快照旧文件到 `~/.cc-switch/backups/providers.<ts>.json`，rotate 保留最近 20 份。
- token 在 SSOT 里**明文存储**，与 farion1231/cc-switch 一致。`~/.cc-switch/` 启动时自动 `chmod 700`，文件 `chmod 600`。`secrecy` 只防内存泄露 / 日志泄露，不防磁盘读。

### 6.2 live config 由 `ToolAdapter::render()` 写

- Claude Code：`<base>/settings.json`（user：`~/.claude/`；project：`./.claude/settings.local.json`），保留 v1 的 symlink/copy 选择。symlink target 指向 `~/.cc-switch/live/claude-code/<provider-id>.json` —— 一份 render 缓存，而不是 `~/.claude/profiles/<name>.json`，因为 provider 不再是文件。
- Codex：`~/.codex/config.toml` 整体覆盖（`tempfile + persist`）；`~/.codex/auth.json` 仅在 preset 是 OAuth 时不动，否则按 token 写。
- 渲染缓存目录 `~/.cc-switch/live/<tool>/` 持久化最近一次 render 输出，`Diff 屏` 与 `backfill` 用得上。

### 6.3 旧 `~/.claude/profiles/` 处置

- `cc-switch-cli import-legacy`：扫目录、JSON parse、按文件名生成 Claude Code provider 写进 SSOT，**不动原文件**。
- 迁移完成后旧目录冻结：`cc-switch-cli list / use / add` 都走 SSOT，不再读 `profiles/`。
- 给一条逃生口：`cc-switch-cli-cli export-legacy --dir ~/.claude/profiles` 把所有 Claude Code provider 反向落成 JSON，便于卸载或回滚到旧 bash 脚本。

### 6.4 project scope

- 检测：PWD 下有 `.cc-switch/providers.json` 或 `.claude/profiles/` 时认 project（与 v1 同向但优先 `.cc-switch/`）。
- SSOT 路径变 `./.cc-switch/providers.json`；live 路径仍然按工具走（Claude Code → `./.claude/settings.local.json`，Codex 仍然 `~/.codex/...`，因为 Codex 没有 project-local 概念）。
- 顶部状态栏标 `scope: project (auto)`；首次创建仍要显式 `--scope project`。

## 7. 项目骨架

```
cc-switch-cli/
  Cargo.toml                       # workspace
  rust-toolchain.toml
  cc-switch-cli                        # 旧 bash 脚本，迁移完成后删
  crates/
    cc-switch-core/
      Cargo.toml                   # serde / serde_json / toml / indexmap / tempfile / fs2 / secrecy / uuid / thiserror
      src/
        lib.rs
        provider.rs                # Provider, Auth, ModelMap
        preset.rs                  # Preset, AuthTemplate
        scope.rs
        ssot.rs                    # 读写 providers.json + 锁 + backup rotate
        tools/
          mod.rs                   # ToolAdapter trait
          claude_code.rs
          codex.rs
        data/
          presets/                 # const Preset 数组，按 category 分文件
          model_fields.rs
        diff.rs
    cc-switch-cli/                     # 二进制：CLI + TUI
      Cargo.toml                   # 上 + clap / ratatui / crossterm / tui-input / tui-textarea / similar / anyhow
      src/
        main.rs
        cli/
          list.rs use_.rs add.rs current.rs path.rs pick.rs import_legacy.rs export.rs
        ui/
          mod.rs app.rs keymap.rs theme.rs
          screens/
            providers.rs editor.rs models.rs diff.rs tools.rs
          forms/
            text_input.rs select.rs token_field.rs
  tests/
    cli_legacy_compat.rs           # bash 旧 plain 输出 vs 新二进制 plain 输出
    integration_claude.rs          # 全流程：preset → switch → 读 settings.json
    integration_codex.rs
```

## 8. 验证

### 8.1 core 单测

- `Provider` 序列化 round-trip（含 `secrecy` 字段，明文落盘）。
- `ClaudeCodeAdapter::render`：preset=packycode + token + model_map → 期望 settings.json 字节匹配 fixture。
- `ClaudeCodeAdapter::backfill`：手动改 settings.json → adapter 解析回 ModelMap，覆盖率含 env / model 字段。
- `CodexAdapter::render`：preset=deepseek-codex → 期望 config.toml + auth.json fixture。
- `Preset` 静态校验：所有内置 preset 的 `model_defaults` key 都在对应 adapter 的 `model_fields()` 里（编译期 const_assert 或 build script，运行期再兜一层 unit test）。
- SSOT 锁：两个进程同时写 → 第二个等锁，最终 backups/ 留下第一个版本。

### 8.2 CLI legacy 兼容

- `import-legacy`：fixture `~/.claude/profiles/{work,gpt}.json` → 跑迁移 → SSOT 含两个 Claude Code provider，name 与文件名一致。
- 迁移后 `cc-switch-cli list --plain` 输出与旧 bash `cc-switch-cli list --plain` 在"name 列出顺序 + active 标记"维度等价（不强求字节相等，因为 plain 现在多列；新增 `cc-switch list --plain --legacy` 输出 strict 旧格式）。
- `cc-switch-cli use <name>` 行为：切到对应 provider，settings.json 内容等价于旧 `~/.claude/profiles/<name>.json`（render 模板对 import-legacy 出来的 provider 是恒等）。

### 8.3 TUI 端到端（`ratatui::backend::TestBackend` + crossterm event 注入）

1. 启动 → Tools 屏看到 Claude Code / Codex 两行。
2. Providers 屏 `n` → preset 选择 modal 显示 §3.3 列表 → 选 `packycode` → Editor 屏字段已预填 endpoint。
3. 输入 name + token，`Ctrl-S` → SSOT 多一条 provider；live config 不动（未 switch）。
4. 回 Providers 屏，`Enter` → `~/.claude/settings.json` 出现，含 `_cc_switch` 标记 + 正确的 env 段。
5. Models 屏改 `ANTHROPIC_MODEL` → `Ctrl-S` → 该 provider 是 active，live config 同步更新。
6. Codex 流程同构：换 Tools 屏到 Codex，preset=deepseek-codex，switch 后 `~/.codex/config.toml` 字节匹配 fixture。
7. 冲突合并：进 Editor 后用 shell 改 settings.json → `Ctrl-S` 弹 merge modal。

## 9. 实施阶段

| 阶段 | 范围 | 退出条件 |
|---|---|---|
| **P0** Cargo workspace + core 骨架 | crate 拆分、`Provider` / `Preset` / `ToolAdapter` trait、`ClaudeCodeAdapter` 渲染 + backfill、SSOT 读写 + 锁 + backup | `cargo test -p cc-switch-core` 全绿；`cc-switch-cli import-legacy` 能跑通 |
| **P1** CLI 子命令 | `list / current / use / add / rm / path / export / import / pick / import-legacy`；保留 `--plain --legacy` 与旧脚本字节兼容 | §8.2 全绿 |
| **P2** TUI Providers + Editor + 切换 | preset 选择 modal、表单、`Ctrl-S` 落盘、`Enter` switch；token mask + clipboard | §8.3 用例 1-4 |
| **P3** Models 屏 + Diff 屏 | 模型映射表格编辑、对象层级 diff | §8.3 用例 5、Diff 手测 |
| **P4** Codex 适配器 + Tools 屏 | `CodexAdapter`、TOML 渲染、Tools 屏切工具 | §8.3 用例 6 |
| **P5** 冲突合并 + shared snippet + project scope | merge modal、Shared Config Panel、project SSOT | §8.3 用例 7 |
| **P6** 打包 + 文档 | `cargo install --git` 路径、Release CI、README 重写、CHANGELOG、删旧 bash 脚本（或保留为 v0 的 wrapper） | release tag |

P0–P2 是 MVP：Claude Code 一个工具、preset → 表单 → 切换闭环。痛点 #1 在 P2 结束时已解决；痛点 #2 在 P3 结束时解决（Claude Code 模型映射）。Codex 在 P4 加入。

## 10. 风险与备选

- **Codex live config schema 漂移**：Codex CLI 仍在快速演进，`config.toml` 字段会变。对策：`CodexAdapter::render` 输出最小集（`model_provider` / `model` / `model_reasoning_effort`），其它字段从 backfill 来的 `extras` 原样透传；adapter 不假装理解全部 schema。
- **Preset 清单维护成本**：内置 preset 写死，endpoint / 默认模型变了要发版。对策：preset 数据用版本号标记（`preset.version`），TUI 启动时若发现 SSOT 里某 provider 的 `preset_id` 在新版本 endpoint 已变，状态栏提示"preset updated, re-pick to refresh"，不强行覆盖用户改动。custom preset (`~/.cc-switch/custom-presets.json`) 让用户自维护。
- **shared snippet 越界**：Claude Code 的 `hooks` / `agents` / `mcpServers` 字段语义是 Claude 决定的，cc-switch-cli 只搬运不解释。对策：snippet 整段以 `serde_json::Value` 存，渲染时浅合并到目标 provider 的 extras，不做字段级 diff/校验。
- **token 落盘明文**：与 farion1231/cc-switch 一致选择。要做加密的话，引入 `keyring` crate（系统钥匙串）作为可选后端，不在 P0–P6 范围。
- **二进制大小**：ratatui + crossterm + clap + serde + similar 估约 4–6 MB stripped。可接受。LTO + `panic=abort` 进一步压。
- **A 纯 bash / B Python 备选**：v1 已论证否决，新模型（结构化 Provider + TOML + 表单）让 bash 更不可行；Python + Textual 可做但仍受 v1 §2 列出的所有缺点约束，且 Provider 结构化 + TOML 让 `serde` 比 Python 的同等代码量优势更明显。结论不变。

## 11. 与 v1 的差异速查

- **抽象**：v1 的"profile = JSON 文件"被 v2 的 `Provider` 结构体替代；profiles 目录降级为遗留迁移源。
- **Editor**：v1 是 JSON textarea + env 摊平；v2 是 preset-driven 表单，textarea 仅作兜底。
- **多工具**：v1 只 Claude Code；v2 加 Codex，框架上 `ToolAdapter` 可继续加 Gemini CLI / OpenCode。
- **存储**：v1 直接读写 `~/.claude/profiles/`；v2 SSOT = `~/.cc-switch/providers.json`，live config 由 adapter 渲染。
- **新屏幕**：Models 屏（痛点 #2 主入口）、Tools 屏（多工具切换）。
- **CLI 兼容**：v1 承诺字节级兼容；v2 提供 `--legacy` 子集 + `import-legacy` / `export-legacy` 迁移命令，承认非完全兼容是必要代价。
