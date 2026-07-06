# parcels

Package your work — repo, uncommitted changes, and the live agent session — and
ship it over Tailscale to another machine. By default the handoff is safe-idle:
secrets are excluded, the remote copy is not deleted, and the agent is not
launched until you opt in.

Supports **Claude Code**, **Codex**, **pi**, and **Droid** sessions.

## Install

```sh
git clone https://github.com/0xSero/parcels && cd parcels && ./install.sh
```

This symlinks `parcel` into `~/.local/bin`, installs the `/parcel` skill for
Claude Code (`~/.claude/skills/parcel`), and seeds `~/.parcels/targets/` with an
example target config. Requirements: `ssh` access to your targets (Tailscale
MagicDNS names work great), `rsync`, `tmux` + the agent CLIs on the targets.

```
parcel send pop-os                      # ship cwd repo + newest claude session, idle by default
parcel send pop-os --launch             # also start the remote tmux session
parcel send pop-os --agent droid        # same but for a droid session
parcel send spark --idle                # ship but don't launch
parcel send spark --include-secrets     # include .env/secrets only for a trusted target
parcel send spark --delete-remote       # delete remote files missing locally
parcel send pop-os --session <id>       # pick an exact session
parcel send pop-os --prompt "focus on the failing tests first"
parcel status                           # what's running where (tails tmux panes)
parcel doctor pop-os                    # preflight: binaries + auth on the target
parcel auth spark --confirm-copy-credentials
parcel targets                          # list configured targets
```

## How it works

1. **HANDOFF.md** — the sending agent writes its intent (what it was doing, next
   steps, gotchas) at the repo root; `parcel` appends a mechanical state snapshot
   (branch, last commit, `git status`). This is the universal fallback: any agent
   on any machine can pick up from it.
2. **Working tree** — rsync of the whole repo including `.git` and untracked
   files (excludes `node_modules`, `.venv`, `__pycache__`, build dirs). `.env`,
   `.env.*`, key files, and common cloud/auth folders are excluded unless
   `--include-secrets` is passed. `.parcelignore` is honored when present.
3. **Session transplant** — the session transcript is copied into the remote
   agent's expected per-project location (each agent sanitizes the cwd path
   differently; `parcel` knows all four layouts) so native resume works:
   - claude: `~/.claude/projects/<sanitized-cwd>/<uuid>.jsonl` → `claude --resume <uuid>`
   - codex: `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl` → `codex exec resume <uuid>`
   - pi: `~/.pi/agent/sessions/<sanitized-cwd>/*.jsonl` → `pi -c`
   - droid: `~/.factory/sessions/<sanitized-cwd>/<uuid>.jsonl` (+ index entry) → `droid exec -s <uuid>`
4. **Launch** — by default, no tmux session is started. Pass `--launch` to start
   `parcel-<repo>` on the target and resume with your prompt (default:
   "read HANDOFF.md and continue"). Attach any time:
   `ssh -t <host> tmux attach -t parcel-<repo>`.

## Targets

One file per machine in `~/.parcels/targets/<name>.conf`:

```sh
HOST=pop-os      # ssh host (Tailscale MagicDNS name works)
DEST_BASE=ai     # repos land in ~/<DEST_BASE>/<repo-name> on the target
AGENT=claude     # default agent for this target
```

## Auth (one-time per machine, not per handoff)

- **claude** — run `claude login` on the target once, or mint a long-lived token
  on an authed machine with `claude setup-token` and set `CLAUDE_CODE_OAUTH_TOKEN`
  in the target's shell profile.
- **codex / pi** — plain-file creds; `parcel auth <target>
  --confirm-copy-credentials` copies them. Use only for fully trusted machines.
- **droid** — encrypted per-machine; run `droid` once on the target to log in.

`parcel doctor <target>` tells you exactly what's missing.

## Safety flags

- `--include-secrets` includes `.env`, key files, and common cloud/auth folders.
- `--delete-remote` restores the previous overwrite behavior with `rsync --delete`.
- `--trust-remote-repo` pre-accepts Claude Code's trust dialog on the target.
- `--allow-dangerous-agent-flags` restores the previous Claude/Codex bypass flags.

These are intentionally opt-in. Do not use them for shared machines, client
repos, or any repo with credentials unless you have reviewed the target and the
handoff boundary.
