---
name: pi-installation
description: Install, repair, and sync the pi coding agent environment on macOS / Linux — CLI, skills, extensions, shared settings, MCP, pi-web daemon. Re-running this guide on multiple machines converges them to the same configuration. Everything shareable lives under ~/.agents so other agents can reuse it.
disable-model-invocation: true
---

# Pi Environment Installation & Sync Guide

Two purposes:

1. **Install / repair**: set up the full pi environment from scratch on a clean machine.
2. **Sync / converge**: re-run on a configured machine to uninstall off-list extensions / skills, install missing items, and align shared settings.

**General principles**:

- **Universal**: every step works on both macOS and Linux, using only standard installation methods (official npm/pnpm commands, standard OS service managers). No custom install prefixes or per-machine path tricks.
- **Idempotent**: every step is safe to re-run.
- **Shared layer**: anything usable by pi AND other agents lives as real files under `~/.agents/` (skills, `AGENTS.md`, `mcp.json`); the pi side holds only symlinks. pi-only resources (extensions, settings) stay under `~/.pi/agent/`.
- **Machine-local stays out**: models, credentials, and credential-bearing MCP servers are per-machine — this guide neither documents nor manages them.

`settings.shared.json` (next to this file) holds the configuration that must stay identical across machines; everything else in `~/.pi/agent/settings.json` is machine-local.

## Prerequisites

| Dependency | Purpose |
|------------|---------|
| Node.js + npm | pi and its extensions are npm packages |
| git + GitHub SSH key | the repo clones via its SSH URL |

Windows is not supported.

## 1. Install the pi CLI

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
pi --version   # verify
```

pnpm also works: `pnpm add -g @earendil-works/pi-coding-agent`.

## 2. Clone the coding-skills repo

The repo always lives at `~/.config/coding-skills`. Below, `$SKILL` refers to `~/.config/coding-skills/common/pi-installation`.

```bash
if [ -d ~/.config/coding-skills/.git ]; then
  git -C ~/.config/coding-skills pull --ff-only
else
  git clone git@github.com:VincentFF/coding-skills.git ~/.config/coding-skills
fi
```

## 3. Shared layer: skills and AGENTS.md

Layout: real files in the repo → symlinked into `~/.agents` → symlinked into `~/.pi/agent`.

> **Note**: `pi install` / `pi update --extensions` wipes `~/.pi/agent/skills/` and re-syncs package skills. On a clean machine run section 4 before this section, and re-run this section (idempotent) after every extension update.

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
  agents_dst=~/.agents/skills/"$name"
  pi_dst=~/.pi/agent/skills/"$name"
  for dst in "$agents_dst" "$pi_dst"; do
    [ -e "$dst" ] && [ ! -L "$dst" ] && { echo "real dir in the way, skipped: $dst" >&2; continue 2; }
  done
  ln -sfn "$src" "$agents_dst"
  ln -sfn "$agents_dst" "$pi_dst"
done
```

A real directory in place of a symlink means the machine carries unpushed local content — sync it into the repo first, then re-run; never delete it blindly. To enable another repo skill, add its name to `SKILLS` and re-run. pi-installation itself is not linked: it is a bootstrap guide, executed directly from the repo path.

## 4. Install extensions (pi packages)

Extensions are pi-only (installed under `~/.pi/agent/`); what IS shareable with other agents is the global CLI some of them provide (see context-mode below). The canonical list lives in `settings.shared.json` → `packages`:

```bash
pi install npm:@upstash/context7-pi      # Context7 documentation lookup
pi install npm:pi-mcp-adapter            # MCP server integration
pi install npm:pi-subagents              # sub-agent / workflow orchestration
pi install npm:pi-web-access             # web search and content fetching
pi install npm:@narumitw/pi-btw          # additional toolset
pi install npm:@janvitos/pi-plan-build   # Plan/Build workflow with explicit approval
pi install npm:context-mode              # large-output sandbox, FTS5 knowledge base
```

Individual installs can be skipped: after the settings merge (6.2), `pi update --extensions` (6.3) installs whatever is missing.

### context-mode: two extra steps

The pi package provides only the in-session tools. Full capability additionally requires:

1. **Global npm install** — provides the MCP server binary / CLI, reusable by other agents. npm ≥11 blocks install scripts by default; allow them explicitly, otherwise the better-sqlite3 native module never compiles:

   ```bash
   npm install -g --allow-scripts=context-mode,better-sqlite3 context-mode
   # or allow permanently:
   npm config set allow-scripts=context-mode,better-sqlite3 --location=user
   ```

2. **Register the MCP server** in `~/.agents/mcp.json` — pi reads this file via pi-mcp-adapter, and other agents can point at the same file:

   ```json
   { "mcpServers": { "context-mode": { "command": "context-mode" } } }
   ```

   This is the only MCP server this guide manages. Machine-specific, credential-bearing servers are out of scope — they are neither documented here nor touched by the converge steps, and `mcp.json` is never copied into the repo.

## 5. Install pi-web (system daemon)

pi-web (<https://github.com/agegr/pi-web>) runs as an auto-start daemon on fixed port **10803** — systemd user service on Linux, launchd agent on macOS. (The upstream README covers only foreground runs; daemon setup is yours.)

1. Install globally (Node ≥ 22.19) and note the binary path:

   ```bash
   npm install -g @agegr/pi-web@latest   # or: pnpm add -g @agegr/pi-web
   command -v pi-web
   ```

2. **Linux** — systemd user unit `~/.config/systemd/user/pi-web.service` (use the step-1 path):

   ```ini
   [Unit]
   Description=Pi Web - browser UI for the pi coding agent
   After=network.target

   [Service]
   ExecStart=<pi-web path> --no-open --hostname 127.0.0.1 --port 10803
   Restart=on-failure
   RestartSec=3

   [Install]
   WantedBy=default.target
   ```

   ```bash
   systemctl --user daemon-reload
   systemctl --user enable --now pi-web
   loginctl enable-linger "$USER"   # needed to start without login
   ```

   It is a **user** service — every management command needs `--user`: `systemctl --user status pi-web`, `journalctl --user -u pi-web -f`.

3. **macOS** — launchd agent `~/Library/LaunchAgents/com.pi-web.server.plist`:

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
   <plist version="1.0">
   <dict>
     <key>Label</key>
     <string>com.pi-web.server</string>
     <key>ProgramArguments</key>
     <array>
       <string>__PI_WEB__</string>
       <string>--no-open</string>
       <string>--hostname</string>
       <string>127.0.0.1</string>
       <string>--port</string>
       <string>10803</string>
     </array>
     <key>EnvironmentVariables</key>
     <dict>
       <key>PATH</key>
       <string>__NODE_BIN__:__PNPM_BIN__:/usr/local/bin:/usr/bin:/bin</string>
     </dict>
     <key>KeepAlive</key>
     <true/>
     <key>RunAtLoad</key>
     <true/>
     <key>StandardOutPath</key>
     <string>__HOME__/Library/Logs/pi-web.out.log</string>
     <key>StandardErrorPath</key>
     <string>__HOME__/Library/Logs/pi-web.err.log</string>
   </dict>
   </plist>
   ```

   Replace `__PI_WEB__` with the step-1 path, `__NODE_BIN__` / `__PNPM_BIN__` with the node / pnpm bin directories (the daemon needs node on PATH to spawn pi), and `__HOME__` with the absolute home directory.

   ```bash
   launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.pi-web.server.plist
   # after editing the plist, reload:
   launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.pi-web.server.plist
   launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.pi-web.server.plist
   ```

   Manage with `launchctl list | grep pi-web`; logs at `~/Library/Logs/pi-web.{out,err}.log`.

4. Verify `curl http://127.0.0.1:10803/` responds.

pi-web registers its relay as a machine-local package entry in `settings.json`; sections 6.1 / 6.2 preserve it automatically. pi-side web access configuration is per-machine and out of scope.

## 6. Sync shared configuration and converge the environment

Re-running this section IS the multi-machine sync. **Run in order** (6.1 must come before 6.2).

### 6.1 Uninstall extra extensions

Expected = `packages` in `settings.shared.json`. Machine-local path entries (e.g. the pi-web relay) don't start with `npm:`/`git:` and are preserved automatically; every other extra is uninstalled. Must run before 6.2 — the merge overwrites `packages` wholesale, and removing afterwards would only leave orphaned files on disk.

```bash
SKILL=~/.config/coding-skills/common/pi-installation
comm -23 \
  <(jq -r '(.packages // [])[]' ~/.pi/agent/settings.json | grep -E '^(npm|git):' | sort) \
  <(jq -r '.packages[]' "$SKILL/settings.shared.json" | sort) \
| while read -r pkg; do pi remove "$pkg"; done
```

### 6.2 Merge shared settings

`settings.shared.json` carries the keys that must stay identical across machines: `theme` / `defaultThinkingLevel` / `hideThinkingBlock` / `packages`. The merge overwrites only those, preserving machine-local model settings (`defaultProvider` / `defaultModel` / `enabledModels`) and local-path packages:

```bash
[ -f ~/.pi/agent/settings.json ] || echo '{}' > ~/.pi/agent/settings.json
PKGS_LOCAL=$(jq -c '[(.packages // [])[] | select(startswith("npm:") or startswith("git:") | not)]' ~/.pi/agent/settings.json)
jq -s --argjson local "$PKGS_LOCAL" \
  '.[0] + (.[1] | {theme, defaultThinkingLevel, hideThinkingBlock}) + {packages: ((.[1].packages + $local) | unique)}' \
  ~/.pi/agent/settings.json "$SKILL/settings.shared.json" > /tmp/pi-settings.json \
  && mv /tmp/pi-settings.json ~/.pi/agent/settings.json
```

To change shared configuration: edit `settings.shared.json`, commit, push; on the other machines `git pull` and re-run this section.

### 6.3 Install missing / update extensions

```bash
pi update --extensions
```

Then check `ls ~/.pi/agent/skills/` — package-skill sync may have wiped the section-3 symlinks; re-run section 3 if any are missing.

### 6.4 Converge skills

Expected = `$SKILLS` (section 3) plus machine-kept external skills in `SKILLS_KEEP`; everything else under `~/.agents/skills/` is drift and deleted:

```bash
SKILLS="code-review codebase-design confluence-pages diagnosing-bugs \
doc-writing domain-modeling git-commit grill-me grill-with-docs \
grilling research writing-great-skills"   # same as section 3
SKILLS_KEEP="find-skills gitops-cluster-debug gitops-knowledge gitops-repo-audit alicloud"

for entry in ~/.agents/skills/*; do
  name=$(basename "$entry")
  case " $SKILLS $SKILLS_KEEP " in
    *" $name "*) ;;                            # managed or kept
    *) echo "remove: $name"; rm -rf "$entry" ;;  # symlink or real dir, delete either way
  esac
done
find ~/.pi/agent/skills -type l ! -exec test -e {} \; -delete   # clean up the symlinks this breaks
```

Package skills under `~/.pi/agent/skills/` are managed by pi itself — don't touch them. As in section 3, a **real directory** on the delete list needs a content check first: it may hold unpushed local work.

### 6.5 MCP servers

This guide manages only the `context-mode` entry (section 4). All other entries in `~/.agents/mcp.json` are machine-local and left untouched.

### 6.6 Loose pi resources

Loose files under `~/.pi/agent/{extensions,agents,themes}/` are pi-specific with no managed list — anything present is local drift; review and delete (or bring under repo management):

```bash
ls ~/.pi/agent/extensions/ ~/.pi/agent/agents/ ~/.pi/agent/themes/ 2>/dev/null
```

### Local configuration that does NOT sync with the repo

| Configuration | Location | Note |
|---------------|----------|------|
| Models and provider | `defaultProvider` / `defaultModel` / `enabledModels` in `settings.json` | preserved by the 6.2 merge |
| Login credentials | `~/.pi/agent/auth.json` | `/login` on each machine |
| Machine MCP servers | `~/.agents/mcp.json` | credential-bearing; out of scope, never copied into the repo |
| External skills | real directories under `~/.agents/skills/` | register in `SKILLS_KEEP` (6.4) to keep |
| Machine packages | local-path entries in `settings.json` | preserved by 6.1 / 6.2 |

## 7. Verify

```bash
SKILL=~/.config/coding-skills/common/pi-installation

pi --version
pi list                    # shared list + machine-local path packages, nothing else

# extensions: nothing extra, nothing missing (both outputs should be empty):
comm -23 <(jq -r '(.packages // [])[]' ~/.pi/agent/settings.json | grep -E '^(npm|git):' | sort) \
         <(jq -r '.packages[]' "$SKILL/settings.shared.json" | sort)
comm -13 <(jq -r '(.packages // [])[]' ~/.pi/agent/settings.json | sort) \
         <(jq -r '.packages[]' "$SKILL/settings.shared.json" | sort)

ls -la ~/.agents/skills ~/.pi/agent/skills   # managed skills are symlinks: repo → ~/.agents → pi
find ~/.agents/skills ~/.pi/agent/skills -type l ! -exec test -e {} \; -print   # no broken links
ls -la ~/.agents/AGENTS.md ~/.pi/agent/AGENTS.md

command -v context-mode                                # global CLI exists
jq -e '.mcpServers["context-mode"]' ~/.agents/mcp.json # MCP entry configured

# shared settings keys identical:
diff <(jq -S '{theme, defaultThinkingLevel, hideThinkingBlock}' ~/.pi/agent/settings.json) \
     <(jq -S '{theme, defaultThinkingLevel, hideThinkingBlock}' "$SKILL/settings.shared.json") \
  && echo settings OK

curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:10803/   # pi-web responds
systemctl --user is-active pi-web         # Linux; Linger=yes is the autostart prerequisite
launchctl list | grep com.pi-web.server   # macOS
```

After starting `pi`, type `/reload`; the skills will appear in the available-skills list of the system prompt.
