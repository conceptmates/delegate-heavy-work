# Engines: preflight, setup, and running

Read this once an engine is in play — before the engine question, so the options offered are
real, and again before the first delegated run on OpenCode or Codex. Skip it entirely for
Sonnet subagents and for anything below the floor.

## Preflight

Preflight **installs**; it does not file a report about what is missing. Offering an engine as
"needs setup" and moving on is how a session ends up on the fallback forever.

Two things have to be true before an engine is real: the **plugin** is installed in Claude Code,
and the plugin says the **CLI** under it is ready. Detect the first, then let the plugin answer
the second.

```bash
grep -Eil 'opencode|codex' ~/.claude/plugins/installed_plugins.json \
     ~/.claude/plugins/known_marketplaces.json 2>/dev/null
command -v opencode >/dev/null && echo "opencode CLI present"
command -v codex    >/dev/null && timeout 10 codex --version 2>/dev/null
```

Sub-second. It tells you what to install, and nothing about readiness. Absence of those JSON
files is not proof of absence — if detection is ambiguous, offer the engine as *unverified*
rather than missing.

### Install the missing plugin

Codex — marketplace `openai-codex`, plugin `codex`:

```
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
```

OpenCode — marketplace `tasict-opencode-plugin-cc`, plugin `opencode`:

```
! curl -fsSL https://raw.githubusercontent.com/tasict/opencode-plugin-cc/main/install.sh | bash
/reload-plugins
```

That curl pipes a remote script into a shell — show it and let the user approve rather than
running it unannounced. `tasict/opencode-plugin-cc` is upstream; identical-looking forks exist,
so do not substitute a fork URL without the user deliberately confirming it. A plugin is not
loaded until `/reload-plugins`, so do not verify it in the same breath as installing it.

### Readiness is the plugin's own answer

Do not hand-roll the CLI check. Each plugin ships one, it returns in 2–3s, and it installs the
CLI itself — via its own single `AskUserQuestion` then `npm i -g` — when the binary is absent.
That is why this skill carries no CLI install commands of its own.

```
/codex:setup      →  {"ready": true,      "codex": {…}, "auth": {"loggedIn": true, …}}
/opencode:setup   →  {"installed": true,  "version": "…", "providers": [], "reviewGate": false}
```

Branch on `ready` for Codex, `installed` for OpenCode.

**`codex --version` is not the readiness check.** Codex availability is two probes, not one —
`codex --version` *and* `codex app-server --help`. A machine passes the first, fails the second,
and then every plugin command throws *"missing required runtime support"*. Use `--version` only
as the free gate for whether anything is installed at all.

**Codex auth cannot be shelled.** There is no `codex login status`. The plugin opens an
app-server client and reads the account back over the protocol, so `/codex:setup`'s
`auth.loggedIn` is the only authoritative answer. Codex wants a ChatGPT account (free tier
included) or an `OPENAI_API_KEY`, and runs count against your Codex usage limits.

**OpenCode needs no login at all.** `providers: []` is cosmetic — the plugin prints it and never
gates on it. The CLI ships free models on the Zen endpoint that run at zero credentials, so for
OpenCode, on PATH *is* ready. `opencode auth login` is optional and only buys paid Zen models or
your own provider keys. Never block on it, and never run it unattended — it is interactive and
will hang the shell until something kills it.

**Neither plugin has a timeout.** Both wrap a bare `spawnSync` with no timeout option, and both
can hang on a server that never answers. Bound them from your side. A probe that hits the bound
is *unverified*, never *missing* — offer the engine, labelled. Do not retry: you have already
spent the budget proving it is slow.

Both plugins need Node 18.18+. If an install fails, say what broke, fall back to Sonnet, and
carry on — do not loop.

## Running the engines

**Spawn through the `Agent` tool.** Both plugins register a subagent type, and that is how a
delegated run starts — not a CLI shelled out through Bash, which is untracked, unstoppable, and
hands you stdout to parse.

| Engine | Launch |
|---|---|
| OpenCode | `Agent` with `subagent_type: "opencode:opencode-rescue"`, or `/opencode:*` |
| Codex | `Agent` with `subagent_type: "codex:codex-rescue"`, or `/codex:*` |
| Sonnet | `Agent` with `model: "sonnet"` |

Read the exact type name off the session's agent list before you call it. A plugin installed but
not reloaded has registered nothing, and a guessed `subagent_type` fails the call outright.

The run-time flags below stay inside the plugin subagent, which builds the command itself. You
need them in two cases: a workflow agent calling `opencode run` directly — the one sanctioned
Bash invocation — and diagnosing a plugin run that came back wrong.

**OpenCode.** Resolve the model at run time — `opencode models` (optionally filtered by
provider). Hardcoded IDs go stale and fail at the worst moment. Pick cheapest-capable for
research, grepping, QA loops and bulk work; mid-tier for codegen from a spec; strongest for
review — and for review prefer a different provider family than the session's own, since the
same family shares the same blind spots.

```bash
opencode run -m <provider/model> --variant <effort> --auto "<the prompt>"
```

`--auto` auto-approves non-denied permissions and is what makes an unattended run work — omitting
it is what hangs one. `--variant` is the reasoning-effort dial from the table above. Other flags:
`--agent`, `-f` to attach files, `--dir`, `-c`/`-s` to continue a session. `--format json` emits
raw JSON *events*, not a single answer — extract the final assistant message, or omit it for
formatted text. Redirect long runs to a file.

**Give it room.** A cold `opencode run` costs tens of seconds before the model says anything, and
a real sweep runs minutes. A default two-minute tool timeout kills it mid-run and you pay for the
tokens anyway. Background it, or set the bound explicitly.

The free tier is the default choice for volume work — free models carry a `-free` suffix and need
no credentials. Never hardcode one; the lineup rotates:

```bash
opencode models | grep -- '-free'
```

Slash commands: `/opencode:review`, `/opencode:adversarial-review`, `/opencode:rescue`,
collected with `/opencode:status` and `/opencode:result`, stopped with `/opencode:cancel`.

**Codex.** `/codex:review`, `/codex:adversarial-review` (pressure-tests design decisions, not
just line-level bugs), `/codex:rescue`, `/codex:transfer`, with `/codex:status`,
`/codex:result`, `/codex:cancel`. Reviews are slow — background them.

Both plugins ship their **review gate** off (`reviewGateEnabled: false`). Leave it there — it
loops the two agents against each other at stop time and drains usage limits for little gain
over one review pass.

**Sonnet subagent.** `Agent` tool with `model: "sonnet"`. `Explore` for read-only search and
grepping, `general-purpose` when it must also write or run commands. Nothing to install and no
CLI underneath it, so there is no preflight for this one.
