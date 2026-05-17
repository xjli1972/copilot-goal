---
name: goal
description: Set a stop condition and loop autonomously inside autopilot until that condition is met, then call task_complete. Triggers on /goal, "set a goal", "goal: ...", "keep going until ...", "don't stop until ...", "loop until ...", "achieve <condition>". Mirrors Claude Code's /goal UX. Subcommands: `goal <condition>` to set, `goal` (no args) to show status, `goal clear` to cancel.
user-invocable: true
argument-hint: <condition up to 4000 chars> | clear | (empty to show status)
---

# /goal — Bounded Autonomous Loop

A Copilot CLI port of Claude Code's `/goal <condition>` command. Sets a user-defined stop condition; the agent then keeps working autonomously (inside autopilot) until the condition is met, after which it calls `task_complete` and surrenders the loop.

The goal lives in this session's SQL store. It survives turns but dies when the Copilot session ends. **Re-set it after every session if you want continuity.**

---

## Subcommands

| Invocation | Action |
|---|---|
| `goal <condition>` | Set a new goal. Replaces any existing goal. Up to **4000 chars**. |
| `goal` | Show the current goal, its status, and iteration count. |
| `goal clear` | Clear the goal. Loop ends; do **not** call `task_complete` (user is bailing). |

Natural-language synonyms count too: *"keep going until tests pass"*, *"don't stop until the build is green"*, *"goal: bug 1234 is closed"* → treat the predicate as `<condition>` and set it.

---

## Hard rules

1. **Max condition length is 4000 characters.** If exceeded, refuse with a one-line error and do not change state. Tell the user how many chars they're over and offer to truncate.
2. **Exactly one active goal at a time.** Setting a new goal replaces the old one with no merge attempt; announce the replacement.
3. **The goal is the contract for `task_complete`.** While a goal is active, do not call `task_complete` for any other reason. Wait for the goal predicate to evaluate true, then call `task_complete` once.
4. **Never claim the goal is met without evidence.** "Tests pass" requires a passing test command in the last 10 turns; "build green" requires a successful build; "bug X is closed" requires a fresh ADO/IcM query. If the predicate can't be objectively checked, ask the user once for clarification, then proceed.
5. **Safety stops.** End the loop and report (without `task_complete`) if any of these trip:
   - Iteration count exceeds **50 agent turns** since the goal was set.
   - **3 consecutive turns** with zero file/diff/command change AND no measurable progress toward the predicate.
   - The same destructive action would need to repeat (avoid retry-storms on push/delete).
   - User issues `goal clear` or any explicit "stop" message.
6. **Do not violate other guardrails to chase a goal.** Git push, secret leaks, mass deletes, content-exclusion-blocked files — refuse and pause the loop. Tell the user what blocked you.

---

## State storage

Use the session SQL store. Schema (create idempotently the first time you set or read a goal):

```sql
CREATE TABLE IF NOT EXISTS goal_state (
  id          INTEGER PRIMARY KEY CHECK (id = 1),  -- singleton row
  condition   TEXT NOT NULL,
  status      TEXT NOT NULL DEFAULT 'active',       -- active | achieved | cleared | aborted
  set_at      TEXT NOT NULL,                         -- ISO timestamp
  achieved_at TEXT,
  iterations  INTEGER NOT NULL DEFAULT 0,
  last_check  TEXT,                                  -- ISO timestamp of last self-evaluation
  notes       TEXT                                   -- free-form progress trail
);
```

Operations:

- **Set**: `INSERT OR REPLACE INTO goal_state (id, condition, status, set_at, iterations, notes) VALUES (1, ?, 'active', datetime('now'), 0, NULL);`
- **Read**: `SELECT condition, status, set_at, iterations, achieved_at, notes FROM goal_state WHERE id = 1;`
- **Bump iteration**: `UPDATE goal_state SET iterations = iterations + 1, last_check = datetime('now') WHERE id = 1 AND status = 'active';`
- **Achieve**: `UPDATE goal_state SET status = 'achieved', achieved_at = datetime('now') WHERE id = 1;`
- **Clear**: `UPDATE goal_state SET status = 'cleared' WHERE id = 1;`
- **Abort** (safety stop): `UPDATE goal_state SET status = 'aborted', notes = COALESCE(notes,'') || ? WHERE id = 1;`

---

## UX rules — match Claude Code's `/goal` surface

Render goal state at the top of every visible status line / progress update while active.

| State | Render |
|---|---|
| **Active** | `🎯 Goal active (iter N/50): <first 80 chars of condition>…` |
| **Achieved** | `✅ Goal achieved: <first 80 chars>` (also call `task_complete` once) |
| **Cleared** | `⏹ Goal cleared` |
| **Aborted (safety stop)** | `⛔ Goal aborted: <one-line reason>` |
| **No goal set** | `No goal set. Usage: \`goal <condition>\`` |

Match these literal strings — they are part of the contract that makes the skill recognisable as the `/goal` analogue.

---

## Behavior

### When the skill is first invoked

1. Parse args.
   - **No args** → run **Read**, render the appropriate state line above, exit.
   - **`clear`** (case-insensitive, only token) → run **Clear**, render `⏹ Goal cleared`, exit. Do **not** call `task_complete`.
   - **Anything else** → treat the entire remainder as `<condition>`.
2. Length check. If `len(condition) > 4000`, refuse:
   `⛔ Goal too long: <N> chars (max 4000). Trim and retry, or say "set goal anyway and truncate" to use the first 4000 chars.`
3. If an active goal exists, show:
   `Replacing existing goal: "<old first 80 chars>" → "<new first 80 chars>"`
4. Run **Set**.
5. Announce: `🎯 Goal set (iter 0/50): <first 80 chars>`. Then immediately begin working toward it (do NOT return control to user unless the very first evaluation already satisfies the goal).

### On every subsequent agent turn while `status = 'active'`

This is the core loop. The agent should self-trigger this even when the user didn't re-invoke `goal`.

1. **Bump iteration.** Run the Bump SQL.
2. **Render the active line** at the top of the user-facing update.
3. **Check the predicate** with real evidence — not vibes. Examples of acceptable evidence:
   - "tests pass" → look at the *most recent* test invocation's exit code in this turn or the previous one.
   - "build succeeds" → most recent build command exit code.
   - "PR #X merged" → fresh `gh pr view X --json state` returns `MERGED`.
   - "bug 1234 closed" → fresh issue/work-item lookup (e.g. `gh issue view 1234 --json state` or the equivalent ADO tool) returns `Closed`/`Done`.
   - "report file exists with N rows" → `wc -l` on the file.
   - Subjective conditions ("the design feels right") → ask the user once for a concrete proxy, store it in `notes`, then use that proxy thereafter.
4. **Branch on result:**
   - **Met** → run **Achieve**, render `✅ Goal achieved: <condition first 80>`, then call `task_complete` with a summary that includes the goal text, the evidence, and the iteration count.
   - **Not met** → keep working. Take one concrete next step toward the goal. Append a 1-line note to `notes` describing what you tried this turn (`iter N: ran pytest → 3 failures in tests/auth/`).
5. **Safety check.** Before returning to step 1 of the next turn:
   - If `iterations >= 50`, run **Abort** with reason `"Hit iteration cap (50)"` and render `⛔ Goal aborted: hit iteration cap`. Do not call `task_complete`. Tell the user what was last tried and what's still blocking.
   - If 3 prior consecutive `notes` lines show no diff/command/result change, run **Abort** with `"Stalled: 3 turns no progress"`. Same exit behavior.
   - If a guardrail tripped, run **Abort** with the blocking reason and explain.

### When the user types `goal clear` mid-flight

Run **Clear**, render `⏹ Goal cleared`, exit. Do **not** call `task_complete`. Return control to user.

### When the user invokes `goal` (no args) mid-flight

Render the active state line with iter/N/notes count. Do not interrupt the loop.

---

## How this maps to Claude Code's `/goal`

| Claude Code `/goal` behavior | This skill |
|---|---|
| `/goal <condition>` sets a goal | `goal <condition>` (skill) |
| `/goal` shows status | `goal` (no args) |
| `/goal clear` cancels | `goal clear` |
| "🎯 /goal active" indicator | `🎯 Goal active` line at the top of progress updates |
| "Goal achieved" toast | `✅ Goal achieved` + `task_complete` call |
| "No goal set" | `No goal set. Usage: \`goal <condition>\`` |
| Per-session state, dies on quit | Per-session SQL store, dies on Copilot session end |
| Requires trusted workspace + hooks enabled | Requires autopilot mode (i.e., `--allow-all-tools` + non-interactive permissioning) |

---

## Examples

**Set a test-pass goal:**
> `goal all tests pass in the eval/ folder`
>
> → `🎯 Goal set (iter 0/50): all tests pass in the eval/ folder`
> → agent runs pytest, iterates on failing tests, re-runs, repeats until exit 0
> → `✅ Goal achieved: all tests pass in the eval/ folder` + `task_complete`

**Cancel mid-flight:**
> `goal clear`
> → `⏹ Goal cleared`

**Check status:**
> `goal`
> → `🎯 Goal active (iter 7/50): rewrite docs/onboarding.md without slop and pass stop_slop review`

**Over the limit:**
> `goal <5000-char wall of text>`
> → `⛔ Goal too long: 5012 chars (max 4000). Trim and retry, or say "set goal anyway and truncate" to use the first 4000 chars.`

---

## What the skill does NOT do

- Does not persist across Copilot sessions (Claude Code's `/goal` doesn't either; if you want cross-session goals, use Memories or a markdown file).
- Does not override Copilot's content-exclusion policies, secret-handling rules, git-push protections, or other base guardrails — those always win.
- Does not modify autopilot or YOLO mode flags. The skill assumes the user already launched Copilot with the autonomy level they want.
- Does not retry destructive actions (no auto-`rm -rf`, no auto-`git push --force`).

If the user wants different bounds, they can override at set time:
> `goal max 100 iters: get all the lint clean`
> → store the override in `notes`, use 100 instead of 50.
