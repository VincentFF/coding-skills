---
name: pi-installation
description: 在干净的 macOS / Linux 机器上安装或修复 pi coding agent 全套环境：CLI、skills 软链、extensions、共享 settings、pi-web 系统守护。
disable-model-invocation: true
---

# Pi Agent 安装指引（macOS / Linux）

在一台干净的 macOS 或 Linux 机器上安装 pi coding agent 全套环境，全程幂等，可重复执行。共享配置（`settings.shared.json`）随本 repo 同步，各桌面保持同一套；模型、凭据、机器特有的 MCP server 等本地配置不随 repo 走。

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

Repo 统一放在 `~/.config/coding-skills`（本 skill 即在其中），使用 SSH 协议地址。下文 `$SKILL` 指 `~/.config/coding-skills/common/pi-installation`。

```bash
if [ -d ~/.config/coding-skills/.git ]; then
  git -C ~/.config/coding-skills pull --ff-only
else
  git clone git@github.com:VincentFF/coding-skills.git ~/.config/coding-skills
fi
```

## 3. 创建软链

pi 从 `~/.agents/skills/` 和 `~/.pi/agent/skills/` 发现 skills，全局指令读 `~/.pi/agent/AGENTS.md`。因此每个在用的 skill 建两层软链（repo → `~/.agents/skills/` → `~/.pi/agent/skills/`），外加一条 AGENTS.md 软链。

> **注意**：`pi install`（第 4 节）会清空 `~/.pi/agent/skills/` 再同步 package skills。因此在干净机器上应按 4 → 3 的顺序执行；或按本文顺序执行后，**在第 4 节装完 extensions 再重跑一次本节的软链循环**（幂等）。日常新增 skill 软链不受影响，但凡是跑过 `pi install` / `pi update --extensions` 之后，都检查一次 `ls ~/.pi/agent/skills/`。

```bash
REPO=~/.config/coding-skills
SKILLS="code-review codebase-design confluence-pages diagnosing-bugs \
doc-writing domain-modeling git-commit grill-me grill-with-docs \
grilling research writing-great-skills"
mkdir -p ~/.agents/skills ~/.pi/agent/skills

ln -sfn "$REPO/prompts/AGENTS.md" ~/.pi/agent/AGENTS.md

for name in $SKILLS; do
  src=$(find "$REPO" -mindepth 2 -maxdepth 2 -type d -name "$name" | head -1)
  [ -n "$src" ] || { echo "skill not found in repo: $name" >&2; continue; }
  ln -sfn "$src" ~/.agents/skills/"$name"
  ln -sfn ~/.agents/skills/"$name" ~/.pi/agent/skills/"$name"
done
```

`SKILLS` 只列在用的 skill；要启用 repo 里的其他 skill（如 markdown-check），把名字加进列表重跑本节。pi-installation 自身不入列表：它是 bootstrap 指引，新机器上直接从 repo 路径阅读执行，无需软链。

## 4. 安装 extensions（pi packages）

Extensions 以 pi package 形式安装，记录在 `~/.pi/agent/settings.json` 的 `packages` 字段：

```bash
pi install npm:@upstash/context7-pi      # Context7 文档查询
pi install npm:pi-mcp-adapter            # 接入 MCP server
pi install npm:@tintinweb/pi-subagents   # 子 agent / workflow 编排
pi install npm:pi-web-access             # Web 搜索与内容抓取
pi install npm:@narumitw/pi-btw          # 附加工具集
pi install npm:context-mode              # 大输出沙箱处理、FTS5 知识库与会话续接
```

也可跳过逐条安装：第 6 节的 settings 合并已带上 `packages` 列表，合并后执行 `pi update --extensions` 会补齐缺失的 package。

### context-mode：额外的两步

pi package 只提供 context-mode 的会话内工具，完整能力还需：

1. **npm 全局安装**——提供 MCP server 二进制与 CLI。npm ≥11 默认阻止 install scripts，必须显式放行，否则 better-sqlite3 原生模块不编译、装出来是残的：

   ```bash
   npm install -g --allow-scripts=context-mode,better-sqlite3 context-mode
   ```

   想永久放行可执行 `npm config set allow-scripts=context-mode,better-sqlite3 --location=user`，之后普通 `npm install -g context-mode` 即可。

2. **配置 MCP**——写入 `~/.agents/mcp.json`。该文件只保留这一个通用 server；机器特有的 server（如 mcp-atlassian）按需自行追加：

   ```json
   {
     "mcpServers": {
       "context-mode": { "command": "context-mode" }
     }
   }
   ```

## 5. 安装 pi-web（系统守护）

pi-web（<https://github.com/agegr/pi-web>）以系统守护方式运行（Linux → systemd，macOS → launchd），随机器启动，端口固定 **10803**。

注意：上游 README 只介绍前台运行（`npx @agegr/pi-web@latest` / `pi-web`），**没有** systemd/launchd 章节，守护化需自行配置。步骤如下（以 Linux systemd user service 为例，Node ≥ 22.19）：

1. 全局安装并确认参数：

   ```bash
   npm install -g @agegr/pi-web@latest
   pi-web --help    # 确认 --port / --no-open 等选项
   ```

2. 写 systemd user unit `~/.config/systemd/user/pi-web.service`（`pi-web` 在 nvm 下，路径按 `command -v pi-web` 实际值填）：

   ```ini
   [Unit]
   Description=Pi Web - browser UI for the pi coding agent
   After=network.target

   [Service]
   ExecStart=%h/.nvm/versions/node/v24.20.0/bin/pi-web --port 10803 --no-open
   Restart=on-failure
   RestartSec=3

   [Install]
   WantedBy=default.target
   ```

3. 启动并设置开机自启（user service 需 linger 才能在未登录时自启）：

   ```bash
   systemctl --user daemon-reload
   systemctl --user enable --now pi-web
   loginctl enable-linger "$USER"
   ```

4. 验证 `curl http://127.0.0.1:10803/` 有响应。

它是 **user service**，查询/管理都要带 `--user`：`systemctl --user status pi-web`、`journalctl --user -u pi-web -f`。macOS 上没有 systemd，可改用 launchd 的 `~/Library/LaunchAgents/agegr.pi-web.plist` 达到同样效果（字段对应：ProgramArguments 填 `pi-web --port 10803 --no-open`，KeepAlive=true，RunAtLoad=true）。

pi 侧的 web 访问配置由各机器自行处理，不在本指引同步范围。

## 6. 同步共享配置

skill 目录下的 `settings.shared.json` 是要在各桌面保持一致的部分：

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

改共享配置时，编辑 `settings.shared.json`、commit、push，在另一台机器上 `git pull` 后重跑本节。

不同步的例外（各机器自行配置）：

| 配置 | 位置 | 说明 |
|------|------|------|
| 模型与 provider | `~/.pi/agent/settings.json` 的 `defaultProvider` / `defaultModel` / `enabledModels` | 各机器可能不同，由合并逻辑保留 |
| 登录凭据 | `~/.pi/agent/auth.json` | 每台机器用 `/login` 重新登录 |
| MCP server | `~/.agents/mcp.json` | 通用项只有 context-mode（见第 4 节）；其余 server 各机器自行追加 |
| 其他来源的 skills | `~/.agents/skills/` 下的实体目录 | `find-skills`、`gitops-*` 来自 fluxcd/agent-skills 等外部仓库，需要时单独安装 |

## 7. 验证

```bash
pi list                    # 列出已安装的 package
ls -la ~/.pi/agent/skills  # 每个 skill 应为指向 ~/.agents/skills 的软链
ls -la ~/.agents/skills    # 每个 skill 应为指向 repo 的软链
ls -la ~/.pi/agent/AGENTS.md
command -v context-mode                          # 全局二进制存在
jq -e '.mcpServers["context-mode"]' ~/.agents/mcp.json   # MCP 已配置
jq '{theme, defaultThinkingLevel, hideThinkingBlock, packages}' \
  ~/.pi/agent/settings.json > /tmp/pi-settings-live.json \
  && diff /tmp/pi-settings-live.json "$SKILL/settings.shared.json" \
  && echo settings OK
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:10803/   # pi-web 有响应
systemctl --user is-active pi-web           # user service 状态（注意带 --user）
loginctl show-user "$USER" -p Linger        # 应为 Linger=yes（开机自启前提）
```

启动 `pi` 后输入 `/reload`，skills 会出现在系统提示的 available skills 列表中。
