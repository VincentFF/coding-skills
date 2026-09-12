---
name: pi-installation
description: 在 macOS / Linux 机器上安装、修复并同步 pi coding agent 环境：CLI、skills、extensions、共享 settings、MCP、pi-web。多端重跑本指引可收敛到同一套配置；可共享的部分统一装在 ~/.agents 下，供其它 agent 复用。
disable-model-invocation: true
---

# Pi 环境安装与同步指引（macOS / Linux）

两个用途：

1. **安装 / 修复**：在干净机器上从零装好 pi 全套环境。
2. **同步 / 收敛**：已装好的机器重跑本指引，卸载清单外的 extensions / skills / MCP server，补齐缺失项，使多端配置一致。

全程幂等，可重复执行。共享配置（`settings.shared.json`）随本 repo 同步；模型、凭据、机器特有项等本地配置不随 repo 走（见第 6 节末尾的例外表）。

**共享层原则**：凡 pi 与其它 agent 能共用的东西，实体都装在 `~/.agents/` 下——skills（`~/.agents/skills/`）、全局指令（`~/.agents/AGENTS.md`）、MCP 配置（`~/.agents/mcp.json`），pi 侧只放软链；pi 专有的（extensions / packages、settings）才留在 `~/.pi/agent/`。其它 agent 把各自的 skills / 指令 / MCP 路径指向 `~/.agents` 下对应文件，即可复用同一套能力。

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

## 3. 共享层：skills 与 AGENTS.md

pi 原生从 `~/.agents/skills/` 发现 skills，这一层是跨 agent 公共目录，其它 agent 指向同一目录即可复用，无需另配。全局指令同理：实体放 `~/.agents/AGENTS.md`，pi 侧软链过去。

采用「实体在 `~/.agents`，pi 侧只放软链」的两层结构（repo → `~/.agents/skills/` → `~/.pi/agent/skills/`；AGENTS.md 同理）：

> **注意**：`pi install`（第 4 节）会清空 `~/.pi/agent/skills/` 再同步 package skills。因此在干净机器上应按 4 → 3 的顺序执行；或按本文顺序执行后，**在第 4 节装完 extensions 再重跑一次本节的软链循环**（幂等）。日常新增 skill 软链不受影响，但凡是跑过 `pi install` / `pi update --extensions` 之后，都检查一次 `ls ~/.pi/agent/skills/`。

```bash
REPO=~/.config/coding-skills
SKILLS="code-review codebase-design confluence-pages diagnosing-bugs \
doc-writing domain-modeling git-commit grill-me grill-with-docs \
grilling research writing-great-skills"
mkdir -p ~/.agents/skills ~/.pi/agent/skills

ln -sfn "$REPO/prompts/AGENTS.md" ~/.agents/AGENTS.md
ln -sfn ~/.agents/AGENTS.md ~/.pi/agent/AGENTS.md

for name in $SKILLS; do
  src=$(find "$REPO" -mindepth 2 -maxdepth 2 -type d -name "$name" | head -1)
  [ -n "$src" ] || { echo "skill not found in repo: $name" >&2; continue; }
  ln -sfn "$src" ~/.agents/skills/"$name"
  ln -sfn ~/.agents/skills/"$name" ~/.pi/agent/skills/"$name"
done
```

`SKILLS` 只列在用的 skill；要启用 repo 里的其他 skill（如 markdown-check），把名字加进列表重跑本节。pi-installation 自身不入列表：它是 bootstrap 指引，新机器上直接从 repo 路径阅读执行，无需软链。

## 4. 安装 extensions（pi packages）

Extensions 是 pi 专有机制，装在 `~/.pi/agent/` 下，无法共享给其它 agent；可共享的是个别 package 附带的全局 CLI（如 context-mode，见下）。Packages 记录在 `~/.pi/agent/settings.json` 的 `packages` 字段：

```bash
pi install npm:@upstash/context7-pi      # Context7 文档查询
pi install npm:pi-mcp-adapter            # 接入 MCP server
pi install npm:@tintinweb/pi-subagents   # 子 agent / workflow 编排
pi install npm:pi-web-access             # Web 搜索与内容抓取
pi install npm:@narumitw/pi-btw          # 附加工具集
pi install npm:@janvitos/pi-plan-build   # Plan / Build 工作流、显式审批与实现交接
pi install npm:context-mode              # 大输出沙箱处理、FTS5 知识库与会话续接
```

也可跳过逐条安装：第 6 节的 settings 合并已带上 `packages` 列表，合并后执行 6.3 的 `pi update --extensions` 会补齐缺失的 package。

### context-mode：额外的两步

pi package 只提供 context-mode 的会话内工具，完整能力还需：

1. **npm 全局安装**——提供 MCP server 二进制与 CLI，全局二进制对其它 agent 同样可用。npm ≥11 默认阻止 install scripts，必须显式放行，否则 better-sqlite3 原生模块不编译、装出来是残的：

   ```bash
   npm install -g --allow-scripts=context-mode,better-sqlite3 context-mode
   ```

   想永久放行可执行 `npm config set allow-scripts=context-mode,better-sqlite3 --location=user`，之后普通 `npm install -g context-mode` 即可。

2. **配置 MCP**——写入 `~/.agents/mcp.json`。该文件是共享层的一部分：pi 通过 pi-mcp-adapter 读它，其它 agent 指向同一文件即可。通用 server 只保留这一个；机器特有的 server（如 mcp-atlassian）按需自行追加，并登记进 6.5 的 `MCP_KEEP`：

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

## 6. 同步共享配置，收敛环境

重跑本节即完成多端同步：卸载清单外的 extensions / skills / MCP server，合并共享 settings，补齐缺失项。**按顺序执行**（6.1 必须先于 6.2）。

### 6.1 卸载多余 extensions

期望清单 = `settings.shared.json` 的 `packages`。本机特有的本地路径 package（如 pi-web 注册的 relays）不在其列，自动保留；其余多余的一律用 `pi remove` 卸载。必须先于 6.2 的合并执行——合并会整体覆盖 `packages` 键，之后再删只会留下磁盘残留。

```bash
SKILL=~/.config/coding-skills/common/pi-installation
comm -23 \
  <(jq -r '(.packages // [])[]' ~/.pi/agent/settings.json | grep -E '^(npm|git):' | sort) \
  <(jq -r '.packages[]' "$SKILL/settings.shared.json" | sort) \
| while read -r pkg; do pi remove "$pkg"; done
```

### 6.2 合并共享 settings

skill 目录下的 `settings.shared.json` 是要在各桌面保持一致的部分：

| 键 | 内容 |
|----|------|
| `theme` / `defaultThinkingLevel` / `hideThinkingBlock` | 界面与思考偏好 |
| `packages` | extensions 清单（第 4 节） |

用 `jq` 合并写入——只覆盖共享键，保留机器本地的 `defaultProvider` / `defaultModel` / `enabledModels` 等模型配置；本机特有的本地路径 package 自动保留：

```bash
[ -f ~/.pi/agent/settings.json ] || echo '{}' > ~/.pi/agent/settings.json
PKGS_LOCAL=$(jq -c '[(.packages // [])[] | select(startswith("npm:") or startswith("git:") | not)]' ~/.pi/agent/settings.json)
jq -s --argjson local "$PKGS_LOCAL" \
  '.[0] + (.[1] | {theme, defaultThinkingLevel, hideThinkingBlock}) + {packages: ((.[1].packages + $local) | unique)}' \
  ~/.pi/agent/settings.json "$SKILL/settings.shared.json" > /tmp/pi-settings.json \
  && mv /tmp/pi-settings.json ~/.pi/agent/settings.json
```

改共享配置时，编辑 `settings.shared.json`、commit、push，在另一台机器上 `git pull` 后重跑本节。

### 6.3 补齐 / 更新 extensions

```bash
pi update --extensions   # 按 settings 里的 packages 补齐缺失，并更新已有的
```

跑完后检查 `ls ~/.pi/agent/skills/`：pi 同步 package skills 时可能清掉了第 3 节的软链，缺了就重跑第 3 节。

### 6.4 收敛 skills

期望清单 = 第 3 节 `$SKILLS`（repo 软链）；本机保留的外部 skill 列入 `SKILLS_KEEP`（find-skills、gitops-* 来自 fluxcd/agent-skills 等外部仓库，alicloud 同理，需要时单独安装）。其余一律删除——未登记的即本机漂移：

```bash
SKILLS="code-review codebase-design confluence-pages diagnosing-bugs \
doc-writing domain-modeling git-commit grill-me grill-with-docs \
grilling research writing-great-skills"   # 与第 3 节相同
SKILLS_KEEP="find-skills gitops-cluster-debug gitops-knowledge gitops-repo-audit alicloud"

for entry in ~/.agents/skills/*; do
  name=$(basename "$entry")
  case " $SKILLS $SKILLS_KEEP " in
    *" $name "*) ;;                            # 受管或保留
    *) echo "remove: $name"; rm -rf "$entry" ;;  # 软链或实体目录都直接删
  esac
done
find ~/.pi/agent/skills -type l ! -exec test -e {} \; -delete   # 清掉因此失效的软链
```

`~/.pi/agent/skills/` 下的 package skills 由 pi 自己管理，不手动动。

### 6.5 收敛 MCP server

通用 server 只有 context-mode（第 4 节）；`MCP_KEEP` 登记本机特有的 server，其余从 `~/.agents/mcp.json` 剔除：

```bash
MCP_KEEP="mcp-atlassian mcp-grafana aliyun-openapi-core"   # 按机器实际情况填
keep=$(printf '%s\n' context-mode $MCP_KEEP | jq -Rn '[inputs]')
jq --argjson keep "$keep" \
  '.mcpServers |= with_entries(select(.key as $k | $keep | index($k)))' \
  ~/.agents/mcp.json > /tmp/mcp.json && mv /tmp/mcp.json ~/.agents/mcp.json
```

注意 `mcp.json` 被其它 agent 共用，剔除前确认没有别的 agent 依赖该 server。

### 6.6 散装 pi 资源

`~/.pi/agent/extensions/`、`~/.pi/agent/agents/`、`~/.pi/agent/themes/` 里的散装文件是 pi 专有、无清单可比，有内容即本机漂移，确认后删除（或纳入 repo 管理）：

```bash
ls ~/.pi/agent/extensions/ ~/.pi/agent/agents/ ~/.pi/agent/themes/ 2>/dev/null
```

### 不随 repo 同步的本机配置

| 配置 | 位置 | 说明 |
|------|------|------|
| 模型与 provider | `~/.pi/agent/settings.json` 的 `defaultProvider` / `defaultModel` / `enabledModels` | 各机器可能不同，由 6.2 的合并逻辑保留 |
| 登录凭据 | `~/.pi/agent/auth.json` | 每台机器用 `/login` 重新登录 |
| 本机 MCP server | `~/.agents/mcp.json` | 登记进 6.5 的 `MCP_KEEP` 即保留 |
| 外部 skills | `~/.agents/skills/` 下的实体目录 | 登记进 6.4 的 `SKILLS_KEEP` 即保留 |
| 本机 package | `settings.json` 里本地路径来源的条目 | 6.1 / 6.2 自动识别并保留 |

## 7. 验证

```bash
SKILL=~/.config/coding-skills/common/pi-installation

pi list                    # 应为共享清单 + 本机特有的本地路径 package
# extensions 无多余、无缺失（两条输出都应为空）：
comm -23 <(jq -r '(.packages // [])[]' ~/.pi/agent/settings.json | grep -E '^(npm|git):' | sort) \
  <(jq -r '.packages[]' "$SKILL/settings.shared.json" | sort)
comm -13 <(jq -r '(.packages // [])[]' ~/.pi/agent/settings.json | sort) \
  <(jq -r '.packages[]' "$SKILL/settings.shared.json" | sort)

ls -la ~/.agents/skills    # 每个受管 skill 为指向 repo 的软链；无清单外条目
ls -la ~/.pi/agent/skills  # 每个 skill 应为指向 ~/.agents/skills 的软链
find ~/.pi/agent/skills -type l ! -exec test -e {} \;   # 无失效软链
ls -la ~/.agents/AGENTS.md ~/.pi/agent/AGENTS.md

command -v context-mode                                    # 全局二进制存在
jq -e '.mcpServers["context-mode"]' ~/.agents/mcp.json     # MCP 已配置
jq -r '.mcpServers | keys[]' ~/.agents/mcp.json            # 应为 context-mode + MCP_KEEP 登记项

# settings 共享键一致：
jq '{theme, defaultThinkingLevel, hideThinkingBlock}' ~/.pi/agent/settings.json > /tmp/a.json
jq '{theme, defaultThinkingLevel, hideThinkingBlock}' "$SKILL/settings.shared.json" > /tmp/b.json
diff /tmp/a.json /tmp/b.json && echo settings OK

curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:10803/   # pi-web 有响应
systemctl --user is-active pi-web           # user service 状态（注意带 --user）
loginctl show-user "$USER" -p Linger        # 应为 Linger=yes（开机自启前提）
```

启动 `pi` 后输入 `/reload`，skills 会出现在系统提示的 available skills 列表中。
