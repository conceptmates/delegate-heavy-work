---
name: delegate-heavy-work
description: "Decide where work runs before doing it. Use this skill whenever a task involves sweeping several sources or doing web research, grepping or searching across a codebase, running a test/QA/simulator loop more than once, reviewing a diff or branch, or applying a bulk edit across many files — and also whenever you are about to spawn subagents or author a Workflow script, or ultracode is on. It sets the shape (inline, one subagent, or a workflow), the engine (OpenCode, Codex, or Sonnet), what the delegation prompt must carry, and how to verify what comes back. Applies even if the user never says subagent, delegate, OpenCode, Codex, or workflow. Does not apply to single commands or when the user wants to watch the work."
---

# Delegate heavy work

The main model is the **decision layer**; subagents are the **execution layer**. The line is
not the topic — it is whether the step needs a judgment.

| Main model | Subagent |
|---|---|
| Decides **what** to research, test, search for, review | Runs the searches, suites, greps, review passes |
| **Writes the prompt** | Executes it, nothing beyond it |
| **Picks shape, engine, model tier** | — |
| **Verifies** the result against sources it chose | Reports what it found, with anchors |
| **Writes** the answer the user receives | — |

Judgment is what the expensive model is for. Reading twelve pages or running a suite eleven
times gets the same result from a cheap model at a fraction of the price. Deciding *which*
twelve pages, or what a passing test would prove, is where the expensive model earns its cost.

**The floor.** This applies only to work that is repetitive or wide: 3+ searches, 3+ files, or
a loop that runs more than once. A single test run, grep, file read or build is done inline,
immediately — no routing question, no preflight — even when it falls under a category below.

**The scope.** Delegate: web research and source sweeps; QA, test and simulator loops;
grepping and codebase exploration; code review; bulk mechanical edits. This is a guide to the
shape of qualifying work, not an allowlist — anything else repetitive and wide qualifies too.

Keep inline, at any size: the approach and architecture decisions; ambiguous debugging where
the hard part is knowing what to suspect; judging whether the subagent is right; writing the
final answer or code; anything irreversible.

## When not to use this skill

Delegation is opaque by construction — the user sees a result, not the work. Whatever the
size, work inline when:

- The user is **working alongside you** — debugging live, learning the code, thinking aloud.
  The reasoning is what they asked for; a summary is not it.
- They said **show me**, **walk me through**, **explain as you go**.
- The answer is **blocking them right now**. A fast partial beats a complete one they waited on.
- The work will hit a **judgment call mid-flight** they would want to make. Surface it; do not
  let a subagent decide it and report the decision as a finding.

In these cases say nothing about routing or engines. Just do the work.

## Decision 1: shape

Decided by you, per task. Never asked of the user.

| Shape | When |
|---|---|
| Inline | Below the floor, or anything in the section above. |
| One subagent | One question, one sweep, one review — or work that is inherently sequential. |
| Agent fan-out | A single round of independent tasks whose results you read yourself. Launch them in one message; no script. |
| Workflow | Stages that feed each other, findings needing adversarial verification, or enough items to want a pipeline. |

**The workflow threshold** — all three must hold:

1. **Enumerable units.** You can name them before the script runs — eight domains, forty files,
   five lenses. If you cannot, it is one subagent exploring, not a workflow.
2. **Three or more that run concurrently.** Two is one agent with a list. Sequential-by-nature
   work (fix, re-run, fix again) is one agent however many rounds it takes.
3. **Five or more agents total**, counting verification. Under five, the script costs more than
   it saves.

One override: if the output must be checked by someone other than its author — adversarial
verify, judge panel, completeness critic — that is a workflow even at small counts, because a
single subagent cannot grade itself.

Unsure? Run one subagent. A workflow that should have been one agent is the expensive mistake;
one agent that should have been a workflow just takes longer.

**Cost gate.** Before spawning a workflow, count the agents and compare against doing it
inline. Over ~12 agents, name the number to the user and get a yes first. Inside the script,
check `budget.remaining()` before each fan-out phase and cut the lowest-value phase rather than
overrunning — and `log()` every cut, because bounded coverage reported as complete coverage is
the worst output this skill can produce.

## This rule does not propagate

This skill governs the **main model only**. A spawned agent *is* the execution layer and does
not re-apply the skill to its own work. Without this, the cheap layer becomes another decision
layer and the saving inverts.

- **Never ask the routing question from a subagent.** No user is on the other end.
- **A subagent never spawns a subagent.** If a delegated task looks too big, that is your shape
  decision to redo — split it into more units from the top.
- **Never call `workflow()` from inside a workflow agent.** Nesting is one level, and it is the
  script that nests. An agent that tries it throws, `agent()` returns null, and the unit
  vanishes from the results with no error you will notice.

Enforce it in the prompt, not by hope. End every delegation prompt with this line verbatim:

> You are the execution layer for this task. Do the work yourself: do not spawn subagents, do
> not start a workflow, do not delegate any part of this, and do not ask the user anything.

## When ultracode is on

Ultracode settles the **shape** question and nothing else: every substantive task is a
workflow, verification is adversarial by default.

- **Do not ask the routing question.** The shape is decided and the engine collapses into
  per-agent `model` inside the script. Say in one line that it is running as a workflow, and go.
- **"Token cost is not a constraint" does not mean run greps on Opus.** It licenses *more*
  agents and *deeper* verification, not a pricier model for work where model strength changes
  nothing. Keep the tier table below exactly as written.
- **Standing user instructions still win.** "Stop delegating" or "stay on the current model"
  overrides ultracode's default shape.

## Decision 2: engine — ask once, then it sticks

Ask on the **first** delegable task of a session, using `AskUserQuestion`. Name the actual task
so the choice is concrete, and label each option from what the preflight actually found:

- **OpenCode subagent** — free models, no login; cheap volume work.
- **Codex subagent** — OpenAI's plugin; strongest as a second opinion on review. Needs a login.
- **Sonnet subagent** — native, no setup, results return straight into this session.
- **Stay on the current model** — the override, when accuracy beats cost.

**If they pick OpenCode or Codex, ask two more in the same call** — model, and effort on a 1–5
scale. Three questions, one interruption. Populate the model options from live output
(`opencode models`, or the Codex plugin's own model list), never from memory.

| effort | | OpenCode `--variant` | Codex `model_reasoning_effort` | workflow `effort` |
|---|---|---|---|---|
| 1 | skim | `minimal` | `minimal` | `low` |
| 2 | light | `low` | `low` | `low` |
| 3 | default | `medium` | `medium` | `medium` |
| 4 | deep | `high` | `high` | `high` |
| 5 | max | `max` | `xhigh` | `xhigh` |

After that, reuse all three silently for the session and state in one line where the work went.
The **engine** sticks outright. The model and effort answers are a **ceiling and a default**,
not a pin: drop below them for a grep, never go above them without asking. Research still gets
the cheap end, review still gets the strong end, inside whatever ceiling was set.

Keep the choice recoverable: naming the engine in that one-line report ("ran on OpenCode, as
before") means it survives in the transcript rather than only in the question that set it,
which compaction will eventually eat. If you truly cannot tell whether they chose, do not
re-ask for a small delegation — use the Sonnet subagent and say so. Re-asking an answered
question is worse than a defaulted answer.

Re-ask only if the user changes it (a one-off override does not replace the default; a general
instruction does) or the engine breaks — then say what failed, fall back to Sonnet, and carry
on rather than interrupting to ask.

The default does not carry into a new session. **If nobody is present** (scheduled, headless,
or an earlier question went unanswered): use the Sonnet subagent, say so, carry on. Never stall
on this question. Note that interactively-authenticated tools may be missing in headless runs,
which is a second reason the native subagent is the right unattended default.

## Preflight

Before the engine question, confirm what is actually installed, and install what is missing —
offering an engine as "needs setup" and moving on is how a session ends up on the fallback
forever.

```bash
grep -Eil 'opencode|codex' ~/.claude/plugins/installed_plugins.json \
     ~/.claude/plugins/known_marketplaces.json 2>/dev/null
command -v opencode >/dev/null && echo "opencode CLI present"
command -v codex    >/dev/null && timeout 10 codex --version 2>/dev/null
```

Sub-second, and it tells you only what to install — never readiness. Readiness is the plugin's
own answer: `/codex:setup` → `ready`, `/opencode:setup` → `installed`. Both self-install their
CLI, so this skill carries no `npm install` of its own.

**Read `references/engines.md` before you offer or run either engine.** It carries the plugin
install commands, why `codex --version` is not a readiness check, why OpenCode needs no login,
and the run-time flags for both.

## Writing the delegation prompt

This is your real work on a delegated task. Split what you withhold from what you must supply.

**Never send** your reasoning, what you suspect, the user's history, or the conversation so far
— a cheap model cannot check any of it and will hand your assumptions back as findings.

**Always send** the constraints its output must satisfy: the repo's conventions, framework and
version, paths that are off limits, the output format, and anything the user already ruled out.
A subagent inherits none of this session's loaded skills or project instructions and will apply
generic defaults otherwise. If another loaded skill governs the artifact being produced,
restate its binding rules here or keep that step inline.

Also include:

- **The question**, stated as a question — not a topic to "look into." Vague scope produces a
  wall of notes instead of an answer.
- **Scope bounds** and **what counts as done**.
- **The answer's shape**: ranked findings, a verdict with evidence, a file→symbol table.
- **An anchor requirement**: every load-bearing claim arrives with a file:line, URL, or command
  and its output. Require it in the schema too.
- **A size bound.** Delegation moves tokens off the expensive model; it does not delete them.
  What comes back you read at full price, so the saving is on what the subagent *consumed*, not
  what it hands you. Say "at most N findings, one line each, no preamble" — never "report
  everything." If the evidence is genuinely large, have it written to a file and return the path
  plus a summary. A delegation whose return you have to skim is one you should have done inline.
- **A bounded write mandate**, for anything that edits files: name the exact files or an explicit
  glob, forbid widening it, and forbid every state-changing git command — no commit, checkout,
  stash, reset, or branch deletion. Confirm the tree is clean or committed first, so the whole
  thing is one `git diff` from being undone.

## Authoring a workflow script

Do not write a Workflow script from memory, and do not look for its API here. Invoke the
`workflow-authoring` skill and follow it — it carries the current `agent`/`pipeline`/`parallel`
contract, the `meta` literal, resume behavior, the quality patterns, and the things that throw.
The division runs both ways: that skill explicitly does not authorize running a workflow, and
this one does not describe the API. This skill decides *whether* and *at what shape*; that one
decides *how*. If it is unavailable, do not guess — run the work as one subagent or an Agent
fan-out instead.

What this skill owns inside a script:

**Model tier — the one place to override the Workflow reference.** Workflow agents inherit the
session model when `opts.model` is omitted, and that reference tells you to omit it. That
default is right for workflows whose agents all carry judgment. It is wrong here: omitting it
runs grunt work on the expensive model and costs more than not delegating at all. Note that
`effort` is not a substitute — it buys less thinking on the *same* model, and no amount of
`effort: 'low'` makes an expensive agent cheap.

| Agent's job | `model` | `effort` |
|---|---|---|
| Search, grep, sweep, run suites, apply a known pattern | set explicitly to the cheap tier | `low`–`medium` |
| Code review, adversarial verify, judge scoring | omit (inherit) or set the strongest | `high`+ |
| Final synthesis | do not delegate it — you write it | — |

Never downgrade the agents doing verification: that is where a weak model costs most.

**Fanning out edits.** Agents share one working directory. Read-only fan-out is safe. The moment
two agents write concurrently they overwrite each other, and it does not surface as an error.
Prefer the cheap fixes first: partition by file so no two agents touch the same path; or fan out
read-only agents that *propose* edits and apply them yourself in one pass. Only if agents must
genuinely edit the same tree at once, pay for `isolation: 'worktree'`. Merging worktrees,
resolving conflicts and committing stay with you.

**Both plugins are main-session only.** Neither their slash commands nor their `subagent_type`s
exist inside a workflow agent — only here. That splits by shape:

- **One subagent** — drive the plugin: `Agent` with `subagent_type: "codex:codex-rescue"` or
  `"opencode:opencode-rescue"`, or `/codex:review`, `/opencode:review` and friends, which track
  the job, survive a long run, and give you `status` / `result` / `cancel`.
- **Workflow** — the plugin is unreachable from inside the script, by both routes. OpenCode still
  works there as a raw CLI call (`opencode run … --auto`) from a Bash-capable agent; this is the
  only place that call belongs. Codex has no equivalent worth the trouble, so run the workflow on
  Claude agents and hand Codex the single review pass afterwards, on the workflow's output. Say
  so in the one-line note, so the choice does not look ignored.

## Running the engines

**Spawn through the `Agent` tool, not Bash.** Every subagent this skill launches is an `Agent`
call naming a `subagent_type` — never a coding CLI shelled out through Bash. A Bash-launched CLI
is not a tracked task: you cannot poll, stop, or get notified on it, it either blocks the turn or
orphans itself past a tool timeout, and what comes back is raw stdout you have to parse instead
of a structured result. Bash is for probes and for the one workflow case below.

| Engine | Launch |
|---|---|
| Sonnet | `Agent` with `model: "sonnet"` — `Explore` for read-only search and grepping, `general-purpose` when it must also write or run commands. No setup, nothing to read. |
| OpenCode | `Agent` with `subagent_type: "opencode:opencode-rescue"`, or `/opencode:*` for a long job you want to track. |
| Codex | `Agent` with `subagent_type: "codex:codex-rescue"`, or `/codex:*` for a long job you want to track. |

The two plugin subagent types drive their own CLI inside their own context, so the flags and the
session handling stay with them. Check the session's agent list for the exact type name before
naming it — an installed plugin that has not been reloaded registers neither its agents nor its
slash commands, and a guessed `subagent_type` fails the call.

**The one exception is inside a workflow script**, where plugin slash commands and plugin
subagent types are both unreachable: there a Bash-capable agent calls `opencode run … --auto`
itself. Nowhere else.

`references/engines.md` has the rest — model resolution, the effort flags, `--auto`, the timeout
a cold start needs, and which slash commands track a long job. Read it before the first
OpenCode or Codex run of a session; do not reconstruct the flags from memory.

## Every subagent gets a handle

Give each spawned agent a **short random handle** — one word, no job description in it: `otter`,
`finch`, `mako`, `sable`, `wren`, `kite`, `perch`, `lynx`. Draw a fresh one per spawn and never
reuse one inside a session, including for a retry: a second `otter` makes every earlier line
about `otter` ambiguous.

| Where | How |
|---|---|
| `Agent` tool | `description: "otter"` |
| Workflow `agent()` | `label: 'otter'` |
| Addressing it later | `SendMessage({to: "otter", …})`, `ListAgents` to confirm it is still live |

A handle stays unique when two agents are doing the identical job on different slices, which is
exactly when a descriptive label collides and you can no longer tell which one to re-run or
stop. It is also short enough to say in a report line without crowding out the finding.

**The handle carries no meaning, so you carry it.** Nothing in `otter` says what otter was for.
Post the map in your own text the moment you spawn — one line per agent, handle then job:

> `otter` — review the auth diff · `finch` — sweep docs for stale flags

Do that at spawn time, not at the end. The transcript is the only place that mapping lives, and
a reader scrolling back to a bare `finch` with no key has to open the tool call to recover it.
Reuse the same handles when you report, so the two halves line up.

Do not put the handle inside the delegation prompt. The agent has no use for its own name, and a
name in the prompt invites it to sign findings or refer to itself in the third person.

## After it runs

**Two verifications, and neither substitutes for the other.** In-workflow verification
(refuters, judge panels) tests whether a claim survives attack *inside the workflow's own
frame* — same sources, same framing. It cannot catch a wrong frame, because every agent
inherited it. Yours tests what the workflow structurally cannot: did this answer the question
the user actually asked, and do the load-bearing claims hold against a source the workflow did
not pick for itself. Unanimous internal agreement is evidence of consistency, not correctness.
So open one or two primary sources yourself, re-run the arithmetic, read the diff it produced.
A workflow enlarges this step rather than discharging it.

**Anchors.** Open them for every claim you will act on. A claim with no anchor is a **lead, not
a finding**: check it yourself if it is load-bearing; if you cannot, pass it to the user
explicitly labelled unverified. Never promote it on the subagent's say-so — and never delete it
silently, since a lead the user is told about beats a deletion they never learn about. When
there are too many to check, check all of them for claims you will act on and say which remain
unverified.

**Synthesis is a candidate, not the reply.** A workflow's synthesis stage hands you the
best-organised input you will ever get, and it is still input. Never forward it verbatim, and
never present a workflow-written file as done before reading it. If it is right, restating it
costs a paragraph; if it is subtly wrong, forwarding it is how that reaches the user wrapped in
a structure that makes it look verified.

**Plan for the follow-up.** Only the final message enters this conversation — the searches and
file reads do not. For any wide delegation you will plausibly be asked about, require the full
findings written to a file with anchors and returned as a path. Keep the path in your reply and
answer follow-ups from it. Re-delegating to recover evidence you chose not to keep is the most
expensive mistake available here.

Then report what was found or changed, not how the delegation went — one line on where it ran
and on what model. Where more than one agent ran, attribute each finding to the handle that
produced it, so a follow-up question lands on a specific agent's evidence rather than on the
merged pile.