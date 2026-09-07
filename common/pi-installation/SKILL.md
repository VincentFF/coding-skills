---
name: pi-installation
description: 在干净的 macOS / Linux 机器上安装或修复 pi coding agent 全套环境：CLI、coding-skills repo、常用 skills 软链、extensions、共享 settings、以及以 systemd/launchd 守护运行的 pi-web。幂等，可重复执行。
disable-model-invocation: true
---

# Pi Agent 安装指引（macOS / Linux）

在一台干净的 macOS 或 Linux 机器上安装 pi coding agent、extensions、skills、prompts、共享配置和 pi-web。全程幂等，可重复执行。共享配置（`settings.shared.json`）作为本 skill 的文件随 repo 同步，Linux 与 macOS 桌面保持同一套；各机器的模型、MCP、凭据等本地配置不随 repo 走。

## 前置条件

| 依赖 | 用途 |
|------|------|
| Node.js + npm | pi 是 npm 包，extensions 的安装也依赖 npm |
| git | clone repo、安装 git 来源的 pi package |
| GitHub SSH key | repo 与部分 package 使用 SSH 协议，需已配置到 GitHub |

Windows 不在支持范围内。

## 1. 安装 pi CLI

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
pi --version   # 验证
```

也可用 pnpm：`pnpm add -g @earendil-works/pi-coding-agent`。

## 2. Clone coding-skills repo

Repo 统一放在 `~/.config/coding-skills`，默认使用 SSH 协议地址。本 skill 就在 repo 内，以下 `$SKILL` 指 `~/.config/coding-skills/common/pi-installation`。

```bash
if [ -d ~/.config/coding-skills/.git ]; then
  git -C ~/.config/coding-skills pull --ff-only
else
  git clone git@github.com:VincentFF/coding-skills.git ~/.config/coding-skills
fi
```

## 3. 创建软链

目录结构：repo 按 `coding/`、`common/`、`work/` 分组存放 skills，`prompts/` 存放全局 AGENTS.md。pi 从 `~/.pi/agent/skills/` 和 `~/.agents/skills/` 两个位置发现 skills，全局上下文文件读 `~/.pi/agent/AGENTS.md`。因此创建两层软链：

1. `~/.pi/agent/AGENTS.md` → repo 的 `prompts/AGENTS.md`（全局指令）
2. repo 中在用的 skill → `~/.agents/skills/<skill名>`
3. `~/.pi/agent/skills/<skill名>` → `~/.agents/skills/<skill名>`

只链接下面 `SKILLS` 列表中的常用 skill，repo 里其余 skill（如 markdown-check）不链接。要启用某个时，把名字加进列表再重跑本节。

```bash
REPO=~/.config/coding-skills
SKILLS="code-review codebase-design confluence-pages diagnosing-bugs \
doc-writing domain-modeling git-commit grill-me grill-with-docs \
grilling pi-installation research writing-great-skills"
mkdir -p ~/.agents/skills ~/.pi/agent/skills

# 3.1 全局 AGENTS.md
ln -sfn "$REPO/prompts/AGENTS.md" ~/.pi/agent/AGENTS.md

# 3.2 + 3.3 列表内每个 skill 建两层软链
for name in $SKILLS; do
  src=$(find "$REPO" -mindepth 2 -maxdepth 2 -type d -name "$name" | head -1)
  [ -n "$src" ] || { echo "skill not found in repo: $name" >&2; continue; }
  ln -sfn "$src" ~/.agents/skills/"$name"
  ln -sfn ~/.agents/skills/"$name" ~/.pi/agent/skills/"$name"
done
```

`SKILLS` 列表当前包含 13 个在用的 skill：

| Skill | 分组 | 用途 |
|-------|------|------|
| code-review | coding | 按 Standards / Spec 两轴审查改动 |
| codebase-design | coding | 深度模块设计词汇表 |
| diagnosing-bugs | coding | 定位 bug 的系统性流程 |
| domain-modeling | coding | 领域建模 |
| git-commit | coding | 自动/手动 commit 规范 |
| grill-me | coding | 对计划进行压力测试 |
| grill-with-docs | coding | 结合文档的压力测试 |
| grilling | coding | 追问式质询 |
| research | coding | 调研并产出 Markdown 结论 |
| writing-great-skills | coding | 编写 skill 的规范 |
| doc-writing | common | 技术文档写作规范 |
| confluence-pages | work | Confluence 页面操作 |
| pi-installation | common | 本安装指引 |

repo 中还有 markdown-check（work）当前未链接。

## 4. 安装 extensions（pi packages）

Extensions 以 pi package 形式安装，记录在 `~/.pi/agent/settings.json` 的 `packages` 字段。当前安装的 5 个：

| Package | 提供的能力 |
|---------|-----------|
| `npm:@upstash/context7-pi` | Context7 文档查询工具 |
| `npm:pi-mcp-adapter` | 接入 MCP server |
| `npm:@tintinweb/pi-subagents` | 子 agent / workflow 编排 |
| `npm:pi-web-access` | Web 搜索与内容抓取 |
| `npm:@narumitw/pi-btw` | 附加工具集 |

安装：

```bash
pi install npm:@upstash/context7-pi
pi install npm:pi-mcp-adapter
pi install npm:@tintinweb/pi-subagents
pi install npm:pi-web-access
pi install npm:@narumitw/pi-btw
```

或直接向 `~/.pi/agent/settings.json` 写入 `"packages"` 数组后执行 `pi update --extensions`，pi 会自动安装缺失的 package。

## 5. 安装 pi-web（系统守护）

pi-web（<https://github.com/agegr/pi-web>）必须以系统守护方式安装运行：Linux 用 systemd，macOS 用 launchd，随机器启动，不依赖登录会话。端口固定 **10803**。

执行本节时**先获取官方 README 的安装文档并遵循其最新步骤**，不要凭记忆执行可能过期的命令：

1. 抓取 <https://github.com/agegr/pi-web> 的 README，按其当前文档安装最新版。
2. 守护方式按官方文档：Linux → systemd unit（`systemctl` 管理）；macOS → launchd（官方若推荐 `brew services` 亦可）。
3. 端口配置为 10803。
4. 启动并验证服务运行中，且 `curl http://127.0.0.1:10803/` 有响应。

pi-web 装好后，pi 侧的 web 访问配置（如需写入 `~/.agents/mcp.json` 或 settings）由各机器自行处理，不属于本指引的同步范围。

## 6. 同步共享配置

skill 目录下的 `settings.shared.json` 是要在两台桌面保持一致的部分：

| 键 | 内容 |
|----|------|
| `theme` / `defaultThinkingLevel` / `hideThinkingBlock` | 界面与思考偏好 |
| `packages` | extensions 列表（与第 4 节一致） |

用 `jq` 合并写入——只覆盖共享键，保留机器本地的 `defaultProvider` / `defaultModel` / `enabledModels` 等模型配置：

```bash
SKILL=~/.config/coding-skills/common/pi-installation
jq -s '.[0] + (.[1] | {theme, defaultThinkingLevel, hideThinkingBlock, packages})' \
  ~/.pi/agent/settings.json "$SKILL/settings.shared.json" > /tmp/pi-settings.json \
  && mv /tmp/pi-settings.json ~/.pi/agent/settings.json
```

要改共享配置时，编辑 `settings.shared.json`、commit、push，在另一台机器上 `git pull` 后重跑本节。

不同步的例外（各机器自行配置）：

| 配置 | 位置 | 说明 |
|------|------|------|
| 模型与 provider | `~/.pi/agent/settings.json` 的 `defaultProvider` / `defaultModel` / `enabledModels` | 各机器可能不同，由合并逻辑保留 |
| 登录凭据 | `~/.pi/agent/auth.json` | 每台机器用 `/login` 重新登录 |
| MCP server | `~/.agents/mcp.json` | 各机器安装的 MCP 不同，自行配置 |
| 其他来源的 skills | `~/.agents/skills/` 下的实体目录 | `find-skills`、`gitops-*` 来自 fluxcd/agent-skills 等外部仓库，需要时单独安装 |

## 7. 验证

```bash
pi list                    # 列出已安装的 package
ls -la ~/.pi/agent/skills  # 每个 skill 应为指向 ~/.agents/skills 的软链
ls -la ~/.agents/skills    # 每个 skill 应为指向 repo 的软链
ls -la ~/.pi/agent/AGENTS.md
jq '{theme, defaultThinkingLevel, hideThinkingBlock, packages}' \
  ~/.pi/agent/settings.json > /tmp/pi-settings-live.json \
  && diff /tmp/pi-settings-live.json "$SKILL/settings.shared.json" \
  && echo settings OK
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:10803/   # pi-web 有响应
```

启动 `pi` 后输入 `/reload`，skills 会出现在系统提示的 available skills 列表中。
