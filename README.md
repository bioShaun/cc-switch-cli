# cc-switch-cli

`cc-switch` 用于在终端中快速切换 Claude Code settings 配置。

默认管理：

```text
~/.claude/settings.json
~/.claude/profiles/*.json
```

## 安装

把仓库里的 `cc-switch` 链接到 `PATH` 任意位置即可：

```bash
chmod +x ./cc-switch
ln -sfn "$PWD/cc-switch" ~/.local/bin/cc-switch
```

如果 `~/.local/bin` 不在 `PATH`，加入 shell 配置：

```bash
export PATH="$HOME/.local/bin:$PATH"
```

## 切换

最常见的用法：直接把一个 JSON 文件交给 `use`，自动 add + 激活。

```bash
# profile 名从文件名推导：settings.work.json -> work
cc-switch use ~/.claude/settings.work.json

# 显式指定 profile 名 + 源文件
cc-switch use gpt ~/.claude/settings.gpt.json

# 已经导入过的 profile 直接按名字切
cc-switch use work
```

规则：

- 若当前 scope 的 profiles 目录里已有 `<name>.json`，行为等同 `cc-switch use <name>`，不会重新导入。
- 否则按下面顺序解析源文件：本地 profile → 同名 `<name>.json` / `settings.<name>.json`（base 目录、profiles 目录、~/.claude），找到后复制到当前 scope 的 `profiles/<name>.json` 再切换。
- `--project use <name>` 在项目 profile 不存在时会回退查找全局 `~/.claude/profiles/<name>.json` 并复制一份过来；导入时会打印 `Imported profile from <src> -> <dest>` 显式提示来源。
- 同名 profile 会被静默覆盖（与 `cc-switch add` 一致）。如果只想注册不激活，请用 `cc-switch add`。

## 查看

```bash
cc-switch list      # 表格视图：NAME / SIZE / MTIME / PATH，行首 * 标记当前激活；末行汇总
cc-switch current   # 多行详情：profile / target / link / size / mtime / json / scope
cc-switch path      # 打印 target / profiles / mode / scope / base
```

脚本里需要稳定输出？加 `--plain`：

```bash
cc-switch --plain list       # 输出 "  name" / "* name"，与旧版 byte-identical
cc-switch --plain current    # 单行：profile 名 / 'none' / 异常说明
```

`--no-color` 可以保留多行格式但去掉颜色。`NO_COLOR` 环境变量同样有效。

## 交互式选择

不想敲名字？

```bash
cc-switch pick           # fzf 在 PATH 里 → 浮窗选择器，右侧用 jq 上色预览 JSON
                         # 没有 fzf  → 编号菜单 + read，零依赖
```

`pick` 透传 `--project` / `--copy` / `--target` / `--profiles`，选定后等价于 `cc-switch [opts] use <chosen>`。

## 项目级配置

在当前项目目录下管理：

```text
./.claude/settings.local.json
./.claude/profiles/*.json
```

**自动 scope 检测**：当 PWD 下存在 `.claude/profiles/` 或 `.claude/settings.local.json` 时，所有子命令默认走项目 scope，不需要手敲 `--project`。`list` / `current` 输出会标 `scope=project (auto)`。

第一次创建项目配置仍要显式 `--project`（因为 `.claude/` 还不存在，无从检测）：

```bash
# 一步切换
cc-switch --project use dev .claude/settings.dev.json

# 已导入的按名字切
cc-switch --project use dev

# 仅注册不激活
cc-switch --project add prod .claude/settings.prod.json

# 从另一个本地/全局 profile 名导入
cc-switch --project add prod work

cc-switch --project current
```

从项目目录里之后再执行任何子命令，都会自动看项目 scope：

```bash
# 在项目目录里，无需再加 --project
cc-switch current        # → scope: project (auto-detected from PWD)
cc-switch list           # → scope=project (auto)
cc-switch use dev        # 等价 cc-switch --project use dev

# 想强制看全局，加 --user
cc-switch --user current
```

`--target` / `--profiles` 显式给的路径不会被自动检测覆盖。

## 仅注册（不激活）

若只想把一个 settings.json 登记为 profile，但不立刻切过去：

```bash
cc-switch add default ~/.claude/settings.default.json
cc-switch add gpt                       # 自动从已知位置查找 gpt.json
cc-switch add ~/.claude/settings.gpt.json   # 名字从文件名推导
```

## 复制模式

默认用软链接，不会覆盖已有的普通 `settings.json` 文件。要覆盖请加 `--copy`：

```bash
cc-switch --copy use work
```

## JSON 校验

切换前会校验源 JSON：

- 优先使用 `jq`
- 没有 `jq` 时回退到 `python3 -m json.tool`
