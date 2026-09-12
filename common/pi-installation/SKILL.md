---
name: pi-installation
description: Install, repair, and sync the pi coding agent environment on macOS / Linux machines — CLI, skills, extensions, shared settings, MCP servers, pi-web daemon. Re-running this guide on multiple machines converges them to the same configuration. Everything shareable is installed under ~/.agents so other agents can reuse it.
disable-model-invocation: true
---

# Pi Environment Installation & Sync Guide (macOS / Linux)

Two purposes:

1. **Install / repair**: set up the full pi environment from scratch on a clean machine.
2. **Sync / converge**: re-run this guide on an already-configured machine to uninstall extensions / skills / MCP servers that are not on the managed lists, install missing items, and bring all machines to the same configuration.

Everything is idempotent and safe to re-run. Shared configuration (`settings.shared.json`) travels with this repo; local configuration (models, credentials, machine-specific entries) does not — see the exceptions table at the end of section 6.

**Shared-layer principle**: anything usable by both pi and other agents lives as real files under `~/.agents/` — skills (`~/.agents/skills/`), global instructions (`~/.agents/AGENTS.md`), MCP configuration (`~/.agents/mcp.json`) — with the pi side holding only symlinks. pi-only resources (extensions / packages, settings) stay under `~/.pi/agent/`. Other agents reuse the same capabilities by pointing their skills / instructions / MCP paths at the corresponding files under `~/.agents`.

## Prerequisites

| Dependency | Purpose |
|------------|---------|
| Node.js + npm | pi is an npm package; extension installation also depends on npm |
| git | clone the repo; install git-sourced pi packages |
| GitHub SSH key | the repo and some packages use SSH URLs; key must be registered with GitHub |

Windows is not supported.

## 1. Install the pi CLI

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
pi --version   # verify
```

pnpm also works: `pnpm add -g @earendil-works/pi-coding-agent`.

## 2. Clone the coding-skills repo

The repo always lives at `~/.config/coding-skills` (this skill is inside it), cloned via its SSH URL. Below, `$SKILL` refers to `~/.config/coding-skills/common/pi-installation`.

```bash
if [ -d ~/.config/coding-skills/.git ]; then
  git -C ~/.config/coding-skills pull --ff-only
else
  git clone git@github.com:VincentFF/coding-skills.git ~/.config/coding-skills
fi
```

## 3. Shared layer: skills and AGENTS.md

pi natively discovers skills from `~/.agents/skills/` — a cross-agent common directory that other agents can point at directly, no extra wiring needed. Same for global instructions: the real file lives at `~/.agents/AGENTS.md`, and the pi side symlinks to it.

So the layout is "real files under `~/.agents`, symlinks on the pi side" (repo → `~/.agents/skills/` → `~/.pi/agent/skills/`; same for AGENTS.md):

> **Note**: `pi install` (section 4) wipes `~/.pi/agent/skills/` and re-syncs package skills. On a clean machine, run section 4 before section 3; or, after finishing section 4, **re-run this section's symlink loop** (idempotent). Day-to-day additions of skill symlinks are unaffected, but after every `pi install` / `pi update --extensions`, check `ls ~/.pi/agent/skills/` once.

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

`SKILLS` lists only the skills in active use; to enable another skill from the repo (e.g. markdown-check), add its name to the list and re-run this section. pi-installation itself is not in the list: it is a bootstrap guide, read and executed directly from the repo path on new machines, no symlink needed.

## 4. Install extensions (pi packages)

Extensions are a pi-specific mechanism, installed under `~/.pi/agent/`, and cannot be shared with other agents; what IS shareable is the global CLI some packages provide (e.g. context-mode, see below). Packages are recorded in the `packages` field of `~/.pi/agent/settings.json`:

```bash
pi install npm:@upstash/context7-pi      # Context7 documentation lookup
pi install npm:pi-mcp-adapter            # MCP server integration
pi install npm:@tintinweb/pi-subagents   # sub-agent / workflow orchestration
pi install npm:pi-web-access             # web search and content fetching
pi install npm:@narumitw/pi-btw          # additional toolset
pi install npm:@janvitos/pi-plan-build   # Plan / Build workflow, explicit approval and implementation handoff
pi install npm:context-mode              # large-output sandbox, FTS5 knowledge base, session continuation
```

You can skip the individual installs: the settings merge in section 6 already carries the `packages` list, and running `pi update --extensions` in 6.3 afterwards installs whatever is missing.

### context-mode: two extra steps

The pi package only provides context-mode's in-session tools. Full capability additionally requires:

1. **Global npm install** — provides the MCP server binary and CLI; the global binary is usable by other agents too. npm ≥11 blocks install scripts by default; you must explicitly allow them, otherwise the better-sqlite3 native module never compiles and you get a broken install:

   ```bash
   npm install -g --allow-scripts=context-mode,better-sqlite3 context-mode
   ```

   To allow permanently: `npm config set allow-scripts=context-mode,better-sqlite3 --location=user`, after which a plain `npm install -g context-mode` works.

2. **Configure MCP** — write to `~/.agents/mcp.json`. This file is part of the shared layer: pi reads it via pi-mcp-adapter, and other agents can point at the same file. Keep only this one common server in it; append machine-specific servers (e.g. mcp-atlassian) as needed and register them in `MCP_KEEP` in 6.5:

   ```json
   {
     "mcpServers": {
       "context-mode": { "command": "context-mode" }
     }
   }
   ```

## 5. Install pi-web (system daemon)

pi-web (<https://github.com/agegr/pi-web>) runs as a system daemon (Linux → systemd, macOS → launchd), starts with the machine, fixed port **10803**.

Note: the upstream README only covers foreground runs (`npx @agegr/pi-web@latest` / `pi-web`) and has **no** systemd/launchd section — daemonization is your own job. Steps below (Linux systemd user service example, Node ≥ 22.19):

1. Install globally and confirm the flags:

   ```bash
   npm install -g @agegr/pi-web@latest
   pi-web --help    # confirm --port / --no-open etc.
   ```

2. Write the systemd user unit `~/.config/systemd/user/pi-web.service` (under nvm, fill in the actual path from `command -v pi-web`):

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

3. Start it and enable autostart (a user service needs linger to start without login):

   ```bash
   systemctl --user daemon-reload
   systemctl --user enable --now pi-web
   loginctl enable-linger "$USER"
   ```

4. Verify `curl http://127.0.0.1:10803/` responds.

It is a **user service** — all queries and management need `--user`: `systemctl --user status pi-web`, `journalctl --user -u pi-web -f`. macOS has no systemd; use a launchd `~/Library/LaunchAgents/agegr.pi-web.plist` instead (field mapping: ProgramArguments = `pi-web --port 10803 --no-open`, KeepAlive=true, RunAtLoad=true).

pi-side web access configuration is per-machine and out of scope for this guide.

## 6. Sync shared configuration and converge the environment

Re-running this section IS the multi-machine sync: uninstall off-list extensions / skills / MCP servers, merge the shared settings, install what is missing. **Run in order** (6.1 must come before 6.2).

### 6.1 Uninstall extra extensions

The expected list is `packages` in `settings.shared.json`. Machine-specific local-path packages (e.g. the relays package registered by pi-web) are auto-detected and preserved; everything else extra is uninstalled with `pi remove`. This must run before the merge in 6.2 — the merge overwrites the whole `packages` key, and deleting afterwards would only leave orphaned files on disk.

```bash
SKILL=~/.config/coding-skills/common/pi-installation
comm -23 \
  <(jq -r '(.packages // [])[]' ~/.pi/agent/settings.json | grep -E '^(npm|git):' | sort) \
  <(jq -r '.packages[]' "$SKILL/settings.shared.json" | sort) \
| while read -r pkg; do pi remove "$pkg"; done
```

### 6.2 Merge shared settings

`settings.shared.json` in the skill directory is the part that must stay identical across machines:

| Key | Content |
|-----|---------|
| `theme` / `defaultThinkingLevel` / `hideThinkingBlock` | UI and thinking preferences |
| `packages` | extensions list (section 4) |

Merge with `jq` — overwrites only the shared keys, preserves machine-local model settings like `defaultProvider` / `defaultModel` / `enabledModels`; machine-specific local-path packages are preserved automatically:

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
pi update --extensions   # installs missing packages from settings, updates the rest
```

Afterwards check `ls ~/.pi/agent/skills/`: pi's package-skill sync may have wiped the section-3 symlinks; re-run section 3 if any are missing.

### 6.4 Converge skills

Expected list = `$SKILLS` from section 3 (repo symlinks); machine-kept external skills go in `SKILLS_KEEP` (find-skills, gitops-* come from external repos like fluxcd/agent-skills; same for alicloud — install them separately when needed). Everything else is deleted — unregistered entries are local drift:

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

Package skills under `~/.pi/agent/skills/` are managed by pi itself — don't touch them manually.

### 6.5 Converge MCP servers

The only common server is context-mode (section 4); register machine-specific servers in `MCP_KEEP`, and strip everything else from `~/.agents/mcp.json`:

```bash
MCP_KEEP="mcp-atlassian mcp-grafana aliyun-openapi-core"   # fill in per machine
keep=$(printf '%s\n' context-mode $MCP_KEEP | jq -Rn '[inputs]')
jq --argjson keep "$keep" \
  '.mcpServers |= with_entries(select(.key as $k | $keep | index($k)))' \
  ~/.agents/mcp.json > /tmp/mcp.json && mv /tmp/mcp.json ~/.agents/mcp.json
```

`mcp.json` is shared with other agents — before stripping a server, confirm no other agent depends on it.

### 6.6 Loose pi resources

Loose files in `~/.pi/agent/extensions/`, `~/.pi/agent/agents/`, `~/.pi/agent/themes/` are pi-specific with no managed list to compare against — anything present is local drift; review and delete (or bring under repo management):

```bash
ls ~/.pi/agent/extensions/ ~/.pi/agent/agents/ ~/.pi/agent/themes/ 2>/dev/null
```

### Local configuration that does NOT sync with the repo

| Configuration | Location | Note |
|---------------|----------|------|
| Models and provider | `defaultProvider` / `defaultModel` / `enabledModels` in `~/.pi/agent/settings.json` | may differ per machine; preserved by the 6.2 merge |
| Login credentials | `~/.pi/agent/auth.json` | log in again with `/login` on each machine |
| Machine MCP servers | `~/.agents/mcp.json` | register in `MCP_KEEP` (6.5) to keep |
| External skills | real directories under `~/.agents/skills/` | register in `SKILLS_KEEP` (6.4) to keep |
| Machine packages | local-path entries in `settings.json` | auto-detected and preserved by 6.1 / 6.2 |

## 7. Verify

```bash
SKILL=~/.config/coding-skills/common/pi-installation

pi list                    # should show the shared list + machine-specific local-path packages
# extensions: nothing extra, nothing missing (both outputs should be empty):
comm -23 <(jq -r '(.packages // [])[]' ~/.pi/agent/settings.json | grep -E '^(npm|git):' | sort) \
  <(jq -r '.packages[]' "$SKILL/settings.shared.json" | sort)
comm -13 <(jq -r '(.packages // [])[]' ~/.pi/agent/settings.json | sort) \
  <(jq -r '.packages[]' "$SKILL/settings.shared.json" | sort)

ls -la ~/.agents/skills    # each managed skill is a symlink to the repo; no off-list entries
ls -la ~/.pi/agent/skills  # each skill should be a symlink into ~/.agents/skills
find ~/.pi/agent/skills -type l ! -exec test -e {} \;   # no broken symlinks
ls -la ~/.agents/AGENTS.md ~/.pi/agent/AGENTS.md

command -v context-mode                                    # global binary exists
jq -e '.mcpServers["context-mode"]' ~/.agents/mcp.json     # MCP configured
jq -r '.mcpServers | keys[]' ~/.agents/mcp.json            # should be context-mode + MCP_KEEP entries

# shared settings keys identical:
jq '{theme, defaultThinkingLevel, hideThinkingBlock}' ~/.pi/agent/settings.json > /tmp/a.json
jq '{theme, defaultThinkingLevel, hideThinkingBlock}' "$SKILL/settings.shared.json" > /tmp/b.json
diff /tmp/a.json /tmp/b.json && echo settings OK

curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:10803/   # pi-web responds
systemctl --user is-active pi-web           # user service status (note the --user)
loginctl show-user "$USER" -p Linger        # should be Linger=yes (autostart prerequisite)
```

After starting `pi`, type `/reload`; the skills will appear in the available-skills list of the system prompt.
