# agent-skill-tailnetctl

![AI-Assisted](https://img.shields.io/badge/Code-AI--Assisted-blueviolet)

A portable Agent Skill for [Tailscale](https://tailscale.com/) operations,
wrapping the vendored [`tailnetctl`](https://github.com/andrewintw/tailnetctl)
CLI. Use it to join a tailnet, inspect connection state, list devices, SSH to a
peer, or open an SSH port-forward (web tunnel).

## Features

- Enable/join a tailnet with an auth key.
- Report whether this machine is joined to the tailnet and online.
- List tailnet devices, including self.
- SSH into a selected peer.
- Open an SSH port-forward (web tunnel) to a selected peer.

## Layout

```
agent-skill-tailnetctl/
├── .claude-plugin/
│   └── marketplace.json   # Claude Code plugin marketplace
├── LICENSE
├── NOTICE
└── skills/
    └── tailnetctl/
        ├── SKILL.md
        ├── scripts/
        │   ├── tailnetctl.py   # vendored, unmodified
        │   ├── tailnetctl       # injects --env-file when it exists
        │   ├── check_setup      # JSON readiness check, never prints secrets
        │   ├── ssh_peer         # TTY -> ssh, non-TTY -> --dry-run
        │   └── web_peer         # TTY -> web, non-TTY -> --dry-run
        ├── references/
        │   ├── env.md
        │   ├── env.example      # safe template, no secrets
        │   └── troubleshooting.md
        └── evals/evals.json
```

## Requirements

- Linux or macOS (Windows via WSL)
- Python 3.8+
- `tailscale` client and daemon
- OpenSSH client (`ssh`)

## Install

Install the whole `skills/tailnetctl/` directory, not only `SKILL.md`; the
skill needs its `scripts/` and `references/` files. Restart the agent after
installing so it re-scans skills.

### Claude Code plugin marketplace

```text
/plugin install tailnetctl --marketplace andrewintw/agent-skill-tailnetctl
```

Or add the marketplace and install in two steps:

```text
/plugin marketplace add andrewintw/agent-skill-tailnetctl
/plugin install tailnetctl@agent-skill-tailnetctl
```

### GitHub CLI (cross-agent)

Requires a GitHub CLI build that includes the `gh skill` preview command.

```bash
# install the named skill for one agent
gh skill install andrewintw/agent-skill-tailnetctl tailnetctl \
  --agent opencode --scope user

# or install every skill in the repo (there is one)
gh skill install andrewintw/agent-skill-tailnetctl --all \
  --agent opencode --scope user
```

Use `--agent claude-code`, `--agent codex`, or `--agent gemini-cli` as needed.

### skills CLI (Node.js)

```bash
npx skills add andrewintw/agent-skill-tailnetctl \
  --skill tailnetctl -a opencode -g
```

Use `-a claude-code`, `-a codex`, or `-a gemini-cli`; omit `-g` for project
scope.

### Gemini CLI

```bash
gemini skills install \
  https://github.com/andrewintw/agent-skill-tailnetctl.git \
  --path skills/tailnetctl --consent
```

### Manual (always works)

```bash
git clone https://github.com/andrewintw/agent-skill-tailnetctl \
  ~/repos/agent-skill-tailnetctl

mkdir -p ~/.agents/skills ~/.claude/skills

# opencode / Codex / Gemini / Antigravity
ln -s ~/repos/agent-skill-tailnetctl/skills/tailnetctl \
  ~/.agents/skills/tailnetctl

# Claude Code
ln -s ~/repos/agent-skill-tailnetctl/skills/tailnetctl \
  ~/.claude/skills/tailnetctl
```

If an agent does not follow symlinks, copy the directory instead:

```bash
cp -r ~/repos/agent-skill-tailnetctl/skills/tailnetctl \
  ~/.agents/skills/tailnetctl
```

Installing the skill does not configure the auth key. After installing, run:

```bash
python3 ~/.agents/skills/tailnetctl/scripts/check_setup
```

See [Secrets and first-time setup](#secrets-and-first-time-setup) and
[`skills/tailnetctl/references/env.md`](skills/tailnetctl/references/env.md).

## Secrets and first-time setup

The env file is only needed for `join` (or re-authentication). `status`, `list`,
`ssh`, and `web` work without it.

The skill expects an env file at:

```
~/.config/tailscale/tailnetctl.env
```

Override with `TAILNETCTL_ENV_FILE`. Create it from the bundled template:

```bash
mkdir -p ~/.config/tailscale && chmod 700 ~/.config/tailscale
install -m 600 skills/tailnetctl/references/env.example \
  ~/.config/tailscale/tailnetctl.env
```

Create an auth key at <https://console.tailscale.com/admin/settings/keys>, then
put it in the env file (`TAILSCALE_AUTH_KEY`) or, preferably, a separate
`0600` file referenced by `TAILSCALE_AUTH_KEY_FILE`.

Verify without printing the key:

```bash
python3 skills/tailnetctl/scripts/check_setup
```

`ready: true` means `join` can proceed. When the env file is missing, `join`
stops with setup instructions; the wrapper does not block read-only commands.

Do not commit the env file or the auth key. The repository `.gitignore` excludes
`*.env` and `authkey`. See
[`skills/tailnetctl/references/env.md`](skills/tailnetctl/references/env.md).

## Usage

Ask the agent naturally, for example:

- "啟用 Tailscale"
- "Tailscale 連上了嗎？有沒有 join 到 tailnet？"
- "列出 tailnet 有哪些裝置"
- "幫我 SSH 到 my-host"
- "開 web tunnel 到 peer 的 80 port"

Or call the wrapper directly:

```bash
SKILL=~/repos/agent-skill-tailnetctl/skills/tailnetctl
python3 "$SKILL/scripts/check_setup"
python3 "$SKILL/scripts/tailnetctl" status
python3 "$SKILL/scripts/tailnetctl" status --json
python3 "$SKILL/scripts/tailnetctl" list --json
python3 "$SKILL/scripts/tailnetctl" join --dry-run
python3 "$SKILL/scripts/tailnetctl" join
python3 "$SKILL/scripts/ssh_peer" --peer <host> --remote-user <user>
python3 "$SKILL/scripts/web_peer" --peer <host> --port 80
```

`ssh_peer` and `web_peer` run interactively when a TTY is available; otherwise
they add `--dry-run` and print the equivalent command. This keeps agent tool
calls from hanging on an interactive session.

## Upstream

`skills/tailnetctl/scripts/tailnetctl.py` is vendored from:

- Repository: <https://github.com/andrewintw/tailnetctl>
- Commit: `1526eb0`

The file is unmodified. To update:

1. Copy the new `tailnetctl.py` over the vendored file.
2. Verify with `cmp` against the upstream clone.
3. Update the commit hash in this README and in `NOTICE`.
4. Re-run the read-only checks: `status`, `list`, `ssh --dry-run`,
   `web --dry-run`, `join --dry-run`.

## License

MIT. See [`LICENSE`](LICENSE). The vendored `tailnetctl.py` is from the
upstream project; see [`NOTICE`](NOTICE).
