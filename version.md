# Versão — SYSADM-SERVER

**Versão atual:** `0.1.8`

Agente de administração de servidores, com atualização a partir do GitHub.

> Este arquivo é a **fonte da verdade** da versão do projeto: quem precisar exibir ou
> reportar a versão extrai o **primeiro número semver (`X.Y.Z`)** encontrado aqui.
> Mantenha a linha **"Versão atual"** como a primeira ocorrência de um número.
>
> `0.1.0` marca o início do **versionamento**, não o início do projeto — o que veio
> antes continua no `git log`.

---

## 1. Convenção de Versionamento (`X.Y.Z`)

| Componente | Significado | Como sobe |
|---|---|---|
| **X** | Release estável | Manual |
| **Y** | Mudança estrutural — Novo alvo de plataforma, reescrita de um script, mudança de estrutura do repo. | Manual |
| **Z** | Incremento a cada entrega (ver gatilhos) | A cada entrega |

### Gatilhos de bump do `Z`

- Criar **script novo** ou subcomando novo.
- Mudar o **comportamento** de um script existente (o que ele afeta, o que descobre).
- Alterar **pacote, dependência ou versão-alvo** do sistema provisionado.
- Alterar **flag, parâmetro ou default** de execução.
- Alterar qualquer coisa que **escreva no sistema** (permissão, serviço, disco).

> Correção de texto, comentário e formatação **não** exigem bump.

---

## 2. Formato de Commit Obrigatório

```
X.Y.Z - short description in English
```

**Regras inegociáveis:**

1. A versão **sempre** vem deste `version.md` — bumpe **no mesmo commit** da mudança.
2. Write the message in **English**, specific enough for `git log --grep`.
3. **Proibido** Conventional Commits (`feat:`, `fix:`, `chore:`…) e mensagens vagas
   ("ajuste", "update", "wip").
4. Um objetivo por commit.

> **A skill COMMITTER commita por você neste repo** (existe `.committer.yml` na raiz).
> Escreva a entrada de changelog abaixo ao concluir a entrega: é **dali** que a
> mensagem do commit sai, sem custo de modelo. Sem a entrada, a skill cai num
> fallback que gasta tokens e descreve pior do que você. Detalhe no bloco PS do
> `CLAUDE.md`.

---

## 3. Changelog

> Ordem decrescente (mais recente no topo).

### `0.1.8` — 2026-09-15 — Make the troubleshooting delivery installable and keep tokens out of shell history

The reviewed pull request removed the runtime token log, but its published
`version.json` hash described different `srv.py` bytes. Agents would therefore
download 1.2.87 and reject it as an invalid update. The hash now matches the
delivered script. The unattended configuration example also prompts with echo
disabled instead of embedding a live token in a command, and the commit-format
example now follows the repository-wide English-only rule.

### `0.1.8` — 2026-09-11 — Bump agent CURRENT_VERSION to 1.2.87 and refresh version.json for the debug-log fix

The debug-log removal alone would not have reached already-deployed agents:
`check_update()` only applies a fetched `srv.py` when its `CURRENT_VERSION`
is strictly newer than the running agent's, and validates the download
against the `sha256` recorded in `version.json`. Both were still pointing at
`1.2.86` (the pre-fix content) after the previous commit, so deployed agents
would have seen no version change and silently skipped the update.
`CURRENT_VERSION` is bumped to `1.2.87` and `version.json` regenerated via
`./update_version.sh master` against the corrected `srv.py`. Note this is
the agent's own runtime version (`CURRENT_VERSION` / `version.json`), a
separate numbering from this file's `X.Y.Z` repository version.

### `0.1.8` — 2026-09-11 — Add TROUBLESHOOTING.md covering token, update and git-hook failures

New contributors and operators had no single place mapping the agent's exact
printed error messages to a cause and a fix. Adds `TROUBLESHOOTING.md`,
linked from both `README.md` and `README_br.md`. Documentation only — no
runtime behavior changes, so no separate trigger under section 1 applies;
this entry shares the version bumped by the previous change in this delivery.

### `0.1.8` — 2026-09-11 — Remove token value from debug log in send_metrics retry loop

`send_metrics()` printed the full bearer token to stdout on every send attempt
(`DEBUG TOKEN: '{TOKEN}'`), leaking a live credential into any terminal, cron
log or log collector capturing the agent's output. The debug line is removed;
the retry loop and status reporting are unaffected.

### `0.1.3` — 2026-09-02 — Agent doc: Releases rule and the English-only language rule

Marked echo of the single source at samirhvbr/repodocs. Two rules land here:

1. The `version.md` of the default branch ON GITHUB is what the GitHub Releases
   show, and a commit that bumps it is not finished until that version has a
   tag, a Release and the `Latest` badge — same push, not "later".
2. Everything in this repository is English (US): documents, commit messages,
   pull requests, issues, code comments. The only carve-out is end-user-facing
   product strings. History is not rewritten.

Delimited by a marker, so re-running replaces instead of duplicating.

### `0.1.2` — 2026-09-02 — Regra de Releases no doc de agente: bump e Release sao um ato so

Eco marcado da norma unica em samirhvbr/repodocs (docs/versioning.md). O
`version.md` da branch padrao NO GITHUB e o que as Releases no GitHub mostram, e
um commit que bumpa o `version.md` nao esta terminado ate aquela versao ter tag,
Release e o badge `Latest`.

Bloco delimitado por marcador: rodar de novo substitui, nao duplica.

### `0.1.1` — 2026-09-02 — Releases automaticas: o version.md da master vira tag e Release

O GitHub nao deduz versao de mensagem de commit: sem tag, o numero e string no
`git log` e `git diff` entre versoes nao existe. Entram o
`.github/workflows/release.yml` e o `tools/release.sh`.

**A regra:** o `version.md` da branch padrao **no GitHub** e o que as Releases
**no GitHub** refletem. Checkout local nao entra na conta. Um PR nao publica
nada; no merge, o push do `version.md` dispara o workflow e a Release vira
aquela versao.

Tag e titulo = a versao pura, sem prefixo `v`. Norma:
[samirhvbr/repodocs](https://github.com/samirhvbr/repodocs/blob/master/docs/versioning.md).

### `0.1.0` — 2026-07-30 — Adota o versionamento da casa

Passa a seguir o padrão dos demais repositórios: `version.md` como fonte da verdade,
commits no formato `X.Y.Z - Descrição em português` e changelog como registro de
entrega.

O gatilho foi prático: o repo já participava da skill **COMMITTER**, mas sem
`version.md` não existia o formato da casa — o ciclo reportava e **não commitava**.
Com este arquivo, a skill passa a operar aqui pelo caminho determinístico (sem custo
de modelo), lendo a mensagem da entrada de changelog.

_Gatilhos:_ adoção de infraestrutura de versionamento.
