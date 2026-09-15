# delegate-heavy-work

An agent skill that decides **where** a task runs before any of it runs: inline, one subagent,
an Agent fan-out, or a Workflow — and on which engine and model tier.

The main model stays the decision layer. Repetitive, wide work (3+ searches, 3+ files, a loop
that repeats) goes to the execution layer. Single commands stay inline.

## Install

```bash
npx skills add mhdibrahimcn/delegate-heavy-work -g
```

Drop `-g` to install into the current project instead of your user directory. Add
`--agent claude-code` (or `--agent '*'`) to pick which agents get it.

Verify:

```bash
npx skills ls -g
```

Then start a new agent session — skills are read at session start.

## Manual install

If you would rather not use the CLI, copy the directory:

```bash
git clone https://github.com/mhdibrahimcn/delegate-heavy-work.git
cp -R delegate-heavy-work/delegate-heavy-work ~/.claude/skills/delegate-heavy-work
```

Other agents read from different paths — `.cursor/rules`, `.opencode/skill`,
`.github/skills` — which is what the `skills` CLI handles for you.

## What it covers

- **Shape** — the floor below which work stays inline, and the three conditions that justify a
  Workflow instead of one subagent.
- **Engine** — asked once per session (OpenCode / Codex / Sonnet / stay put), then reused
  silently. Includes the preflight that checks an engine is configured, not just on PATH.
- **The delegation prompt** — what to withhold, what to always send, the anchor requirement, the
  size bound, the bounded write mandate.
- **Handles** — every spawned agent gets a short random one-word name, posted as a handle→job
  map at spawn time and reused when the findings come back.
- **Model tier inside a workflow** — cheap tier for sweeps, strongest for verification, never
  the reverse.
- **Verification after it runs** — why in-workflow verification does not discharge yours.

## Update

```bash
npx skills update delegate-heavy-work -g
```

## License

MIT
