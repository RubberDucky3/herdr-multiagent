---
name: herdr-multiagent
description: Lead a fleet of free and small-model coding agents running in Herdr panes. Use when the agent is inside Herdr (HERDR_ENV=1) and the user asks to delegate, parallelize, fan out, offload, or save tokens across multiple agents, panes, or workers, or invokes /herdr-multiagent. Not for work in a single pane, and never when Herdr is unavailable.
---

# Herdr multiagent — one lead, many workers

You are the lead. Free agents in Herdr panes are your hands. Your context is the
scarce resource; theirs is not. Spend yours on deciding *what* should happen and
on checking that it did; spend theirs on producing the characters. This is the
same bargain as the `orchestra` skill, except the workers are live panes you can
see, steer, and read, instead of one-shot CLI runs.

## 0. Confirm you are inside Herdr

```powershell
if ($env:HERDR_ENV -ne '1') { 'NOT IN HERDR'; exit }
```

```bash
test "${HERDR_ENV:-}" = 1 || { echo "NOT IN HERDR"; exit 1; }
```

If the check fails, say you are not running inside Herdr and stop. Do not try to
drive a Herdr session from outside one. This skill is inert outside Herdr.

## 1. Survey the fleet (once, not per task)

```powershell
herdr agent list                                   # live agents + states
herdr pane list --workspace "$env:HERDR_WORKSPACE_ID"
herdr pane layout --pane "$env:HERDR_PANE_ID"      # decides split direction
herdr integration status                            # what is installed vs missing
```

Prefer **free** worker kinds that are already installed. On this machine the
free CLIs are typically `pi`, `opencode`, `kilo`, `hermes` (plus `cline` when
installed); the free/small *models* live behind them (local Ollama models such as
`ollama/qwen3-coder:30b`, `ollama/gpt-oss:20b`, and the `freemodel` gateway).
Treat `claude` and `codex` as premium: use them only if the user asks or no free
worker fits. `herdr integration status` is the authority — do not assume.

Agent states you will act on: `idle`/`done` = ready or finished, `working` =
busy, `blocked` = waiting at a question/approval, `unknown` = present but
unclassifiable (not proof of completion).

## 2. Decide what to delegate

Delegate work that is **specifiable in a paragraph and verifiable by reading the
result**: scaffolding, boilerplate, mechanical refactors across many files, test
stubs, docstrings, config, format conversions, repetitive edits, "and do this in
each of the other N files".

Keep it yourself when correctness depends on judgment you cannot compress into a
brief: architecture, security-sensitive code, subtle debugging, tricky
concurrency, anything where a plausible-looking wrong answer is expensive. The
free/small models are meaningfully weaker than you; a delegated task you have to
rewrite costs more than one you never delegated.

If a task is half judgment and half typing, split it: you make the decisions,
write them into the brief, and the worker types.

## 3. Write a brief a weaker model can follow

This is the actual skill and where your tokens earn their keep. Every brief is
self-contained:

- Name exact file paths. Never "the config file" — say `src/config.ts`.
- State acceptance criteria concretely: what function, what signature, what it
  returns, what must still pass.
- Include each decision you already made (library, naming, error handling) so
  the worker does not improvise one.
- Say what NOT to touch. Weak models wander.
- One coherent unit of work per worker. Two features means two panes.

## 4. Fan out

Default topology: a sibling pane in the current tab, same working directory, user
focus left alone. Split wide panes to the right, narrow/tall panes down; avoid
repeated same-direction splits that create unusable columns.

Create the pane, read its ID from JSON, start the worker, then submit:

```powershell
$pane = (herdr pane split --current --direction right --cwd "$PWD" --no-focus |
         ConvertFrom-Json).result.pane.pane_id
herdr agent start worker1 --kind opencode --pane $pane
```

`agent start` returns only after Herdr detects the agent and it is ready to take
input. To aim a worker at a smaller/local model, pass the agent's own flags after
`--` (verify the flag against that CLI's `--help` first — do not invent it):

```powershell
herdr agent start worker1 --kind opencode --pane $pane -- --model ollama/qwen3-coder:30b
```

Submit the brief. For **parallel** work, submit to every worker first, then wait,
so the free models run concurrently:

```powershell
herdr agent prompt worker1 "<brief 1>"     # returns once input is written
herdr agent prompt worker2 "<brief 2>"
herdr agent wait worker1 --timeout 600000  # then wait on each
herdr agent wait worker2 --timeout 600000
```

For a single worker, `herdr agent prompt worker1 "<brief>" --wait --timeout 600000`
is shorter. Local models are slow — 600s+ is normal for anything non-trivial.

`agent prompt` rejects an agent already waiting at a question/approval with
`agent_blocked` and sends nothing. On `agent_prompt_stalled`/`timeout`, do not
blindly resubmit: inspect first (step 6).

## 5. Keep the crew in sync

One writer per file. If two workers could touch the same file, either serialize
them or have them edit disjoint files. When workers must agree on an interface,
you define it in the briefs *before* they start — never let two weak models
negotiate a contract mid-flight.

Optional verifier: a second free agent in its own pane whose only brief is "read
this diff and report actionable findings". Give it the diff, not the transcript.
Its job is to catch the seam where two models' work meets.

## 6. Collect compactly, then verify by reading the diff

Workers share the lead's working directory, so the cheap result is the diff:

```powershell
git status --short
git --no-pager diff
```

Read an agent only when the diff is missing, wrong, or the agent looks stuck:

```powershell
herdr agent get worker1
herdr agent read worker1 --source recent-unwrapped --lines 200
```

If a larger read still does not show the answer, ask the worker to write its
result as Markdown to a temp path and reply with only the path — do not request
file output up front, and never pull a whole transcript into your context.

Trust nothing: confirm the changed files do what the brief asked. Close but wrong
→ fix it yourself. Structurally wrong → the brief was underspecified: rewrite it
and re-run.

## 7. Blocked agents and failover

On `blocked` or `agent_blocked`: read the pane/agent, then ask the user before
answering. Never auto-approve a question or approval dialog on the user's behalf.

If a free worker stalls, errors out, or its provider is unreachable, start the
next free kind in a fresh sibling pane and re-brief with **handover context** —
what the previous worker already built and why it stopped. It resumes; it does
not restart.

```powershell
$pane = (herdr pane split --current --direction right --cwd "$PWD" --no-focus |
         ConvertFrom-Json).result.pane.pane_id
herdr agent start worker1b --kind kilo --pane $pane
herdr agent prompt worker1b "Earlier, worker1 (opencode) got as far as <X> then <stopped why>. Continue from <X>. <brief>" --wait --timeout 600000
```

Keep a lightweight mission log the user can resume from — briefs, worker→kind
map, and per-worker outcome — at `.herdr/missions/<id>/`. A fresh session can
pick up where you stopped.

## 8. Safety and focus rules

- Background workers always get `--no-focus`; keep the user's focus in your pane
  unless they ask to switch.
- Target by `--current`, an explicit pane ID, or a unique agent name. Never rely
  on another client's focused pane.
- Parse IDs from JSON responses; never derive them from sidebar order.
- Do not close panes, tabs, or workspaces you did not create unless the user
  explicitly asks.
- Names match `[a-z][a-z0-9_-]{0,31}` and must be unique among live agents.
- Client and server versions can differ after an update; check `herdr status`
  before relying on a new server feature.

## 9. Token discipline

- Never paste file contents into a brief when a path will do — the worker can read.
- Never read a full agent log to confirm success; the diff is the signal.
- Batch related mechanical edits into one brief, not one worker per file.
- Do not delegate something you could finish in a couple of edits — the round
  trip costs more than the work.
- A task is done only when its acceptance criteria pass and you have actually
  read the verifying diff.

## Worked example — "add the same handler to 6 files"

```
1. Survey: herdr agent list → free kinds opencode, kilo, pi available.
2. Judge: mechanical, specifiable, verifiable → delegate. Pick 3 workers x 2 files.
3. Split 3 panes (--current --direction right --cwd "$PWD" --no-focus). Read 3 IDs.
4. Start: worker1=opencode, worker2=kilo, worker3=pi (each --model local).
5. Briefs: exact file paths, the handler signature copied verbatim, "touch no
   other file", acceptance = "cargo test handler_*".
6. Submit all three, then `herdr agent wait` on each (timeout 600000).
7. Verify: git --no-pager diff → read the 6 hunks. One worker missed a file →
   fix that file yourself or re-brief in a new pane.
8. Report: what changed, which worker did what, what you re-ran and its output.
```
