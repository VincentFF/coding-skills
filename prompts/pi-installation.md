# Pi Agent 安装指引（macOS / Linux）

在一台干净的 macOS 或 Linux 机器上安装 pi coding agent、extensions、skills 和 prompts。全程幂等，可重复执行。

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

Repo 统一放在 `~/.config/coding-skills`，默认使用 SSH 协议地址。

```bash
if [ -d ~/.config/coding-skills/.git ]; then
  git -C ~/.config/coding-skills pull --ff-only
else
  git clone git@github.com:VincentFF/coding-skills.git ~/.config/coding-skills
fi
```

## 3. 创建软链

目录结构：repo 按 `coding/`、`common/`、`work-tool/` 分组存放 skills，`prompts/` 存放全局 AGENTS.md。pi 从 `~/.pi/agent/skills/` 和 `~/.agents/skills/` 两个位置发现 skills，全局上下文文件读 `~/.pi/agent/AGENTS.md`。因此创建两层软链：

1. `~/.pi/agent/AGENTS.md` → repo 的 `prompts/AGENTS.md`（全局指令）
2. repo 中每个含 `SKILL.md` 的目录 → `~/.agents/skills/<skill名>`
3. `~/.pi/agent/skills/<skill名>` → `~/.agents/skills/<skill名>`

```bash
REPO=~/.config/coding-skills
mkdir -p ~/.agents/skills ~/.pi/agent/skills

# 3.1 全局 AGENTS.md
ln -sfn "$REPO/prompts/AGENTS.md" ~/.pi/agent/AGENTS.md

# 3.2 repo 内所有 skill → ~/.agents/skills
find "$REPO" -mindepth 2 -maxdepth 2 -name SKILL.md -print0 |
  while IFS= read -r -d '' f; do
    name=$(basename "$(dirname "$f")")
    ln -sfn "$(dirname "$f")" ~/.agents/skills/"$name"
  done

# 3.3 ~/.agents/skills → ~/.pi/agent/skills
for d in ~/.agents/skills/*/; do
  name=$(basename "$d")
  ln -sfn "$d" ~/.pi/agent/skills/"$name"
done
```

当前 repo 中的 13 个 skills：

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
| confluence-pages | work-tool | Confluence 页面操作 |
| markdown-check | work-tool | Markdown 检查 |

`find` 命令按 `SKILL.md` 自动发现，repo 新增 skill 无需修改本指引。

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

## 5. 验证

```bash
pi list                    # 列出已安装的 package
ls -la ~/.pi/agent/skills  # 每个 skill 应为指向 ~/.agents/skills 的软链
ls -la ~/.agents/skills    # 每个 skill 应为指向 repo 的软链
ls -la ~/.pi/agent/AGENTS.md
```

启动 `pi` 后输入 `/reload`，skills 会出现在系统提示的 available skills 列表中。

## 6. 机器相关配置（不属于本指引的同步范围）

| 配置 | 位置 | 说明 |
|------|------|------|
| 模型与 provider | `~/.pi/agent/settings.json` 的 `defaultProvider` / `enabledModels`、`~/.pi/agent/auth.json` | 每台机器用 `/login` 重新登录 |
| MCP server | `~/.agents/mcp.json` | 指向本机代理（如 atlassian、grafana），按机器实际情况配置 |
| 其他来源的 skills | `~/.agents/skills/` 下的实体目录 | `find-skills`、`gitops-*` 来自 fluxcd/agent-skills 等外部仓库，需要时单独安装 |
