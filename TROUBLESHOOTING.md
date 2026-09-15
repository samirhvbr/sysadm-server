# Troubleshooting

Common failure scenarios for the `srv.py` agent, the exact message you will
see for each one, and how to resolve it. Messages below are quoted verbatim
from `srv.py` so you can `grep` this file by the text on your screen.

## Agent exits immediately with "ERRO: TOKEN não carregado!"

`main()` refuses to run without a token (`TOKEN`, loaded once at import time
by `load_token()`). Checked in this order:

1. `BLUE3_TOKEN` environment variable.
2. `TOKEN=` line in `/etc/blue3-agent.conf`.
3. If neither exists, `load_token()` prompts interactively for a token and
   writes it to `/etc/blue3-agent.conf` — this only works in an interactive
   terminal, so it does nothing useful under cron/systemd and the agent will
   keep failing there.

**Fix:** for unattended runs (cron, systemd, containers), set the token
non-interactively before the first run:

```bash
sudo bash -c 'umask 077; read -rsp "Token: " token; printf "\nTOKEN=%s\n" "$token" > /etc/blue3-agent.conf'
```

This prompts without echoing the token and keeps the credential out of shell
history and the process command line. The `umask` makes the file private from
the moment it is created.

or export `BLUE3_TOKEN` in the environment the agent runs under.

## "Token inválido ou muito curto" during the interactive prompt

`load_token()` rejects an empty token or one shorter than 20 characters. Copy
the full token issued by the platform — a truncated paste is the usual cause.

## "❌ Arquivo de token vazio"

`/etc/blue3-agent.conf` exists but has no usable `TOKEN=` line, or the file
was hand-edited into an unparseable shape. `read_config()` accepts either
`KEY=value` lines or, as a fallback, a file containing only the raw token
with no `=` in it. Rewrite the file in the `KEY=value` form shown in
[README.md](README.md) and re-check permissions (`chmod 600`).

## Metrics never arrive / "❌ TOKEN vazio!" from `send_metrics`

`send_metrics()` checks the already-loaded `TOKEN` again before sending. If
you changed `/etc/blue3-agent.conf` after the process started, the running
process still has the old value in memory — restart the agent, since `TOKEN`
is read once at import time, not per send.

## Connection errors: `[Tentativa N] Erro: ...`

`send_metrics()` retries 3 times with a 2-second pause between attempts
before giving up (`Falha ao enviar métricas`). Common causes:

- No outbound HTTPS to `sys.blue3.cloud` (firewall/proxy) — test with
  `curl -v https://sys.blue3.cloud/api/metrics`.
- DNS resolving to an IPv6-only address the network can't route: the agent
  forces IPv4 for all sockets via `force_ipv4()`, so if IPv4 itself is
  unreachable the request will keep failing regardless of IPv6 connectivity.
- A `403`/`401` response body printed as `Erro resposta: ...` means the
  request reached the API but the token was rejected — re-check the token
  value, not the network.

## Git auto-update never applies

`check_update()` runs before every metrics collection and returns early
(silently, from the agent's point of view) in several cases — read the
printed line to tell them apart:

- `"Verificando updates via git no ramo: <branch>"` never appears at all →
  `UPDATE_BRANCH`/`BLUE3_UPDATE_BRANCH` resolved to something unexpected;
  print `load_update_branch()`'s result or check
  `/etc/blue3-agent.conf`.
- `"Git não encontrado. Tentando instalar automaticamente..."` followed by
  `"ERRO: não foi possível instalar git automaticamente."` → none of the
  package managers `ensure_git_available()` tries (`apt-get`, `apt`, `dnf`,
  `yum`, `apk`, `zypper`) are present, or `sudo -n` (non-interactive sudo)
  isn't permitted for this user. Install `git` manually.
- `"Versão remota X não é superior à versão atual Y"` → the branch you
  configured has an older or equal `CURRENT_VERSION` compared to the running
  agent. Confirm you pushed to the branch the agent is actually tracking
  (`UPDATE_BRANCH`), not a different one.
- `"ERRO: HTML detectado no download"` → the configured `UPDATE_REPO_URL`
  is wrong and git is fetching something other than the real repository
  (e.g. a redirected login page mirrored into the clone). Recheck
  `UPDATE_REPO_URL`.
- `"ERRO: hash inválido! Update abortado."` → `version.json` in the update
  branch and the actual `srv.py` bytes disagree. Regenerate `version.json`
  with `./update_version.sh <branch>` from a clean checkout and push both
  files together — see the "Version flow" section in
  [README.md](README.md).

## `./update_version.sh` fails or produces an unexpected `version.json`

- `"❌ Não foi possível encontrar CURRENT_VERSION"` → `srv.py`'s
  `CURRENT_VERSION` line must match `CURRENT_VERSION = "X.Y.Z"` exactly
  (the script greps for the literal string).
- `"⚠️ srv.py possui mudanças locais."` → you have an uncommitted diff in
  `srv.py`; the generated `sha256` will describe your working copy, not what
  you are about to push. Commit `srv.py` first, or you will ship a hash that
  does not match the file once pushed.
- The script always computes the branch from the current checkout unless you
  pass one explicitly (`./update_version.sh testing`) — verify you are on
  the branch you intend to release before running it with no argument.

## Commit rejected by the local git hooks

This repository ships `tools/git-hooks/commit-msg` and
`tools/git-hooks/pre-push`, generated from the house-wide versioning norm.
They are not installed automatically by `git clone` — copy them into
`.git/hooks/` once per checkout if you want the checks to run locally:

```bash
cp tools/git-hooks/commit-msg tools/git-hooks/pre-push .git/hooks/
chmod +x .git/hooks/commit-msg .git/hooks/pre-push
```

- `commit-msg: REJECTED — subject does not start with a version...` → the
  commit subject must be `X.Y.Z - description`, no Conventional Commits
  prefix, description at least 12 characters and not a placeholder word
  ("update", "wip", "fix", ...).
- `commit-msg: REJECTED — subject says X, version.md says Y` → bump
  `version.md` in the same commit as the subject's version, or fix the
  subject to match the version the commit actually carries.
- `pre-push: REJECTED — version X is already on master` → bump `version.md`
  before pushing; the remote's default branch already has that version.
- Both hooks read `REPODOCS_NO_HOOK=1` as an escape hatch for exceptional
  cases (a merge, a revert) — it is not a way around describing a real
  change correctly.

## Where to look when nothing above matches

The agent has no dedicated log file; everything goes to stdout, prefixed
with the message text quoted throughout this document. Run it in the
foreground first (`python3 srv.py`) before wiring it into cron or systemd,
so you see the exact failure instead of a silent non-zero exit.
