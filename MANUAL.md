# grok — Install · Build · Deploy · Use

Practical manual for the `grok` CLI/TUI (SpaceXAI's terminal AI coding agent).
Companion to [README.md](README.md); focuses on operational recipes, not
architecture.

- **Repo:** [x.ai/cli](https://x.ai/cli)
- **Docs:** [docs.x.ai/build/overview](https://docs.x.ai/build/overview)
- **User guide (in-tree):** [crates/codegen/xai-grok-pager/docs/user-guide/](crates/codegen/xai-grok-pager/docs/user-guide/)

---

## 1. Install

Four paths. Pick by trust posture, not habit.

### 1.1 Quick — pipe installer

macOS / Linux / Git Bash:

```sh
curl -fsSL https://x.ai/cli/install.sh | bash
```

Windows PowerShell:

```powershell
irm https://x.ai/cli/install.ps1 | iex
```

Verify:

```sh
grok --version
```

**Blast radius:** user home only (`~/.grok/`). No sudo. No system PATH edit
on Windows. Modifies `~/.bashrc` / `~/.zshrc` / fish config (with `.bak.<epoch>`
backup) so `~/.grok/bin` is on PATH for future shells.

### 1.2 Safe — inspect before running (recommended for shared machines)

```sh
mkdir -p /tmp/grok-install
curl -fsSL https://x.ai/cli/install.sh -o /tmp/grok-install/install.sh
less /tmp/grok-install/install.sh          # audit
bash /tmp/grok-install/install.sh          # then run
```

What to look for in the audit:
- `BASE_URL_PRIMARY` / `BASE_URL_FALLBACK` — the two download hosts
- `BIN_DIR` — install target (defaults to `~/.grok/bin`)
- shell-config block (`# >>> grok installer >>>`) — the `.bashrc` insert
- `GROK_DEPLOYMENT_KEY` branch — only fires when the env var is set

### 1.3 Pin a version

```sh
curl -fsSL https://x.ai/cli/install.sh | bash -s 0.2.103
```

### 1.4 Channels

```sh
GROK_CHANNEL=alpha      bash <(curl -fsSL https://x.ai/cli/install.sh)
GROK_CHANNEL=enterprise bash <(curl -fsSL https://x.ai/cli/install.sh)
# default: stable
```

### 1.5 Update / rollback

```sh
grok update                # install the latest for the current channel
grok update 0.2.100        # pin an older release (rollback)
```

### 1.6 Uninstall

```sh
rm -rf ~/.grok
# then remove the "# >>> grok installer >>>" block from ~/.bashrc (or restore ~/.bashrc.bak.*)
```

Windows extra: to use `grok` from cmd.exe / PowerShell, add
`%USERPROFILE%\.grok\bin` to the user PATH via
*Settings → System → About → Advanced system settings → Environment Variables*.

---

## 2. Build from source

### 2.1 Prereqs

- **Rust** — pinned by [`rust-toolchain.toml`](rust-toolchain.toml); rustup
  installs it on first `cargo` invocation
- **DotSlash** — required so hermetic tools under [`bin/`](bin/) (notably
  `bin/protoc`) can download and run

  ```sh
  cargo install dotslash
  dotslash --help
  ```

- **protoc** — resolved via DotSlash, or falls back to `$PROTOC` / `protoc` on PATH
- macOS + Linux are the tested build hosts. Windows: best-effort.

### 2.2 Commands

```sh
cargo check -p xai-grok-pager-bin            # fast validation
cargo run   -p xai-grok-pager-bin            # build + launch TUI
cargo build -p xai-grok-pager-bin --release  # → target/release/xai-grok-pager
```

Binary artifact is named `xai-grok-pager`. Official installs ship it as `grok`.
To use the built binary as `grok`:

```sh
cp target/release/xai-grok-pager ~/.grok/bin/grok
cp target/release/xai-grok-pager ~/.grok/bin/agent
```

### 2.3 Per-crate development

```sh
cargo check   -p <crate>       # workspace-wide builds are slow; target crates
cargo test    -p xai-grok-config
cargo clippy  -p <crate>
cargo fmt --all
```

Root `Cargo.toml` is generated. Edit per-crate `Cargo.toml` files instead.

### 2.4 Build output verification

```sh
./target/release/xai-grok-pager --version
./target/release/xai-grok-pager inspect      # dump discovered config for cwd
```

---

## 3. Deploy

### 3.1 Single user

Section 1 covers it. Nothing else needed.

### 3.2 Team / enterprise (deployment key)

```sh
GROK_DEPLOYMENT_KEY=<key> bash <(curl -fsSL https://x.ai/cli/install.sh)
```

Effect: fetches `managed_config.toml` + `requirements.toml` from
`https://cli-chat-proxy.grok.com/v1/deployment/config` and writes them to
`~/.grok/`. Managed config overrides user config; requirements pin the
approved model / MCP / plugin set.

### 3.3 CI (headless)

```sh
grok --single "review the diff and flag correctness bugs" \
     --output-format json \
     --permission-mode dontAsk \
     --sandbox strict
```

Key flags for CI:

| Flag | Purpose |
|------|---------|
| `-p` / `--single` | one-shot: prompt in, response out, exit |
| `--output-format json` | machine-readable |
| `--permission-mode dontAsk` | fail on any prompt-worthy action instead of hanging |
| `--sandbox <profile>` | filesystem + network sandbox |
| `--max-turns N` | hard cap on agent loop |
| `--check` | append self-verification loop to the prompt |
| `--best-of-n N` | run in parallel, pick the best result |
| `--json-schema '{...}'` | constrain output to a JSON Schema |

Auth in CI: set `GROK_DEPLOYMENT_KEY` in the runner's secret store. Never
commit `~/.grok/auth.json`.

### 3.4 Editor embedding (ACP)

Grok speaks the [Agent Client Protocol](https://agentclientprotocol.com).
Supported editors: Zed, Neovim (via `zed-industries/agent-client-protocol`),
VS Code, JetBrains. Configure the editor's ACP agent to run `grok` with the
editor's stdio channel; grok auto-detects and switches to ACP mode.

### 3.5 Container / Docker

```dockerfile
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y curl ca-certificates
ARG GROK_VERSION=0.2.103
RUN curl -fsSL https://x.ai/cli/install.sh | bash -s ${GROK_VERSION}
ENV PATH=/root/.grok/bin:$PATH
ENTRYPOINT ["grok"]
```

Provide auth at runtime: `-e GROK_DEPLOYMENT_KEY=...` or bind-mount
`~/.grok/auth.json` (`-v $HOME/.grok:/root/.grok:ro`).

### 3.6 Leader process (multi-session)

```sh
grok leader list                         # list running leaders
grok leader info <pid>                   # inspect one
grok leader kill                         # stop all
grok --leader-socket /tmp/grok-x.sock    # dedicated leader socket
```

One leader manages many sessions; use a custom socket path to isolate teams
sharing a host.

---

## 4. Value-added use cases

Recipes in increasing complexity. Chain them.

### 4.1 Interactive coding (TUI)

```sh
grok                             # blank session, cwd = project root
grok "fix the auth bug in login.rs"
grok -c                          # continue most recent session
grok -r                          # pick a session to resume
```

Slash commands inside TUI: `/help`, `/model`, `/skills`, `/mcp`, `/plan`,
`/permissions`, `/sessions`, `/export`. Full list:
[04-slash-commands.md](crates/codegen/xai-grok-pager/docs/user-guide/04-slash-commands.md).

### 4.2 Headless scripting

Pipe grok into any shell tool.

```sh
# Generate a commit message from staged diff
git diff --staged | grok -p "write a concise commit message for this diff" > msg.txt

# Extract structured data
grok -p --json-schema '{"type":"object","properties":{"bugs":{"type":"array"}}}' \
     "list correctness bugs in src/parser.rs"
```

### 4.3 Worktree isolation (safe experiments)

```sh
grok -w feat/new-cache "implement the LRU cache from BLUEPRINT.md"
# → runs in a fresh git worktree; main tree untouched
```

Combines with `--worktree-ref` to base off a branch/tag/commit.

### 4.4 MCP servers (extend the agent)

Add project-scoped MCP servers so grok reaches your own tools:

```sh
grok mcp add filesystem npx -- @modelcontextprotocol/server-filesystem /workspace
grok mcp add postgres   npx -- @modelcontextprotocol/server-postgres postgresql://...
grok mcp list
```

See [07-mcp-servers.md](crates/codegen/xai-grok-pager/docs/user-guide/07-mcp-servers.md).

### 4.5 Skills (reusable workflows)

Skills are markdown-defined workflows (`~/.grok/skills/<name>/SKILL.md`).
The `codebase-memory`, `deep-research`, and `verify` skills are common;
project teams write their own for repeatable operations (release cut,
migration checklist, incident triage).

Docs: [08-skills.md](crates/codegen/xai-grok-pager/docs/user-guide/08-skills.md).

### 4.6 Sandbox modes

```sh
grok --sandbox read-only  "audit the config for secrets"      # read anywhere, write only ~/.grok + /tmp
grok --sandbox workspace  "refactor the api layer"            # normal dev: writes CWD + ~/.grok + /tmp
grok --sandbox strict     "review this untrusted PR"          # read CWD + system only, no child network
grok --sandbox off        "install deps and run migrations"   # unrestricted (default)
```

Enforcement: Landlock (Linux), Seatbelt (macOS). Windows has no kernel
sandbox — `--sandbox` is a no-op there. Custom profiles: `~/.grok/sandbox.toml`
or per-project `.grok/sandbox.toml`. See
[18-sandbox.md](crates/codegen/xai-grok-pager/docs/user-guide/18-sandbox.md).

Combine with `--permission-mode`:

| Mode | Effect |
|------|--------|
| `default` | ask before each side-effect |
| `acceptEdits` | auto-approve file edits, ask for shell |
| `auto` | approve routine ops, ask for risky ones |
| `dontAsk` | fail on any prompt (CI-friendly) |
| `bypassPermissions` | approve everything (danger) |
| `plan` | produce a plan, execute nothing |

### 4.7 Sub-agents and best-of-N

```sh
grok -p --best-of-n 3 "port this module to async"
# runs 3 parallel attempts, picks the winner
```

Sub-agent definitions live in `~/.grok/agents/` or per-project
`.grok/agents/`. Use them for role separation (`code-reviewer`, `Explore`,
`Plan`) — same pattern as Claude Code sub-agents.

### 4.8 DevFlow cascade (kafcade pattern)

Kafcade is a project-agnostic DevFlow overlay: `kc B` (Build), `kc Ci` (CI
gate), `kc E` (Explore/audit), `kc Bl` (Blueprint check), `kc P` (Push),
`kc D` (Deploy). Grok fits the same shape headlessly:

```sh
# B — implement next roadmap item
grok -p --permission-mode acceptEdits \
     "read BLUEPRINT.md, implement the first unchecked item, run tests"

# Ci — gate on lint + type + test
grok -p --sandbox workspace --permission-mode dontAsk \
     "run cargo fmt --check, cargo clippy --all-targets, cargo test; report failures"

# E — read-only audit (parallel-safe with Ci)
grok -p --sandbox strict \
     "audit BLUEPRINT.md against actual code; report drift"

# Bl — blueprint refresh
grok -p --permission-mode acceptEdits \
     "update BLUEPRINT.md counts and Module Map from current source"
```

Chain in a Makefile or CI job. RRSS discipline
(Robust · Reliable · Solid · Systematic) applies: circuit-break on Ci
failure, auto-detect config, close B only when Ci passes + verify returns 200.

### 4.9 Terse output (kafca overlay)

Force minimal output for pipes / dashboards:

```sh
grok -p --system-prompt-override \
     "Output only the answer. No preamble, no closing summary, no bullets unless asked." \
     "why did the last deploy fail?"
```

Or embed it as a project rule in `.grok/rules.md` / `CLAUDE.md`.

### 4.10 Session export + trace

```sh
grok export <session-id> > session.md
grok trace  <session-id> --upload           # send to observability backend
grok sessions list
```

Useful for post-mortems, review PRs, and training-data curation.

### 4.11 Dashboard + usage monitoring

```sh
grok dashboard              # open the live Agent Dashboard
```

Watch tokens, cost, tool calls, and running sub-agents. Docs:
[23-dashboard.md](crates/codegen/xai-grok-pager/docs/user-guide/23-dashboard.md),
[24-monitoring-usage.md](crates/codegen/xai-grok-pager/docs/user-guide/24-monitoring-usage.md).

---

## 5. Troubleshooting

| Symptom | Fix |
|---------|-----|
| `grok: command not found` after install | `export PATH="$HOME/.grok/bin:$PATH"` (add to shell rc) |
| `bash.exe` as `$SHELL` skipped `.bashrc` update | Add PATH line manually or run `bash -c 'export SHELL=/bin/bash; bash <(curl -fsSL https://x.ai/cli/install.sh)'` |
| Duplicated `[cli]` block in `~/.grok/config.toml` | Harmless. Edit to keep one block. |
| Windows: `grok` works in Git Bash, not cmd | Add `%USERPROFILE%\.grok\bin` to Windows user PATH |
| `grok update` fails with 403 | Auth expired: `grok logout && grok login` |
| Headless run hangs in CI | `--permission-mode dontAsk` + `--max-turns N` |
| DotSlash download blocked (corp proxy) | Set `HTTPS_PROXY`; or vendor `bin/protoc` manually and `PROTOC=/path/to/protoc cargo build` |

Log location: `~/.grok/logs/` — attach when filing a bug.

---

## 6. File layout on disk

```
~/.grok/
├── auth.json                 # OIDC / legacy tokens (chmod 600)
├── config.toml               # user config
├── managed_config.toml       # server-pushed config (deployment-key installs)
├── requirements.toml         # approved model/MCP/plugin set
├── bin/                      # grok.exe, agent.exe (or symlinks on unix)
├── downloads/                # versioned binaries
├── completions/{bash,zsh}/   # shell completions
├── skills/                   # user skills
├── agents/                   # sub-agent definitions
├── logs/                     # session logs
└── leader.sock               # multi-session IPC socket
```

---

## 7. See also

- [README.md](README.md) — repo overview + license
- [SECURITY.md](SECURITY.md) — how to report vulnerabilities
- [User guide](crates/codegen/xai-grok-pager/docs/user-guide/) — in-tree docs (25 topics)
- [docs.x.ai/build/overview](https://docs.x.ai/build/overview) — online docs
- [x.ai/build/changelog](https://x.ai/build/changelog) — release notes
