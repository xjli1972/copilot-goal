# copilot-goal

A GitHub Copilot CLI skill that brings Claude Code's [`/goal <condition>`](https://docs.claude.com/en/docs/claude-code/slash-commands) bounded-autonomous-loop UX to GitHub Copilot CLI.

> Set a stop condition. Step away. Come back to a finished task.

## What it does

`goal <condition>` sets a user-defined stop condition, then the agent keeps working autonomously (inside autopilot mode) until the condition is met — at which point it announces `✅ Goal achieved` and calls `task_complete`.

Mirrors Claude Code 2.1.x's `/goal` surface so you don't have to learn a second mental model:

| Claude Code `/goal` | This skill (Copilot CLI) |
|---|---|
| `/goal <condition>` sets a goal | `goal <condition>` |
| `/goal` shows current state | `goal` (no args) |
| `/goal clear` cancels | `goal clear` |
| `🎯 /goal active` indicator | `🎯 Goal active (iter N/50): …` |
| `Goal achieved` toast | `✅ Goal achieved: …` + auto `task_complete` |
| `No goal set` | `No goal set. Usage: ` `` `goal <condition>` `` |
| Per-session state, dies on quit | Per-session SQL store, dies on Copilot session end |

## Install

```bash
git clone https://github.com/xjli1972/copilot-goal.git
mkdir -p ~/.copilot/skills/goal
cp copilot-goal/skills/goal/SKILL.md ~/.copilot/skills/goal/SKILL.md
```

That's it — Copilot CLI auto-loads user skills from `~/.copilot/skills/` on session start.

## Use

```
goal all tests pass in eval/
goal don't stop until bug 1234 is closed
goal max 100 iters: get all the lint clean
goal             # show current status
goal clear       # cancel
```

Natural-language synonyms also trigger it: *"keep going until tests pass"*, *"don't stop until the build is green"*, *"loop until X"*.

## Guardrails (designed in)

- **4000-char max** on the condition string (enforced; rejects with a helpful error).
- **Exactly one active goal** at a time (singleton SQL row with `CHECK (id = 1)`).
- **50-iteration cap** + **3-turn stall detector** — both abort the loop without calling `task_complete` so you can intervene.
- **Evidence-required predicate evaluation** — "tests pass" needs a real exit-0; "PR merged" needs a fresh `gh pr view`; subjective conditions trigger a one-time clarification stored in the goal's notes.
- Honors content-exclusion / git-push / secret-handling guardrails. Pauses rather than retrying around blocks.
- Does **not** override Copilot's autopilot / YOLO flags — you choose your autonomy level at launch time.

## Three orthogonal stop modes

| Outcome | Indicator | Calls `task_complete`? |
|---|---|---|
| Condition met | `✅ Goal achieved: …` | **Yes** |
| User issues `goal clear` | `⏹ Goal cleared` | No |
| Safety stop (iter cap / stall / blocked action) | `⛔ Goal aborted: <reason>` | No |

## How it differs from Copilot CLI's built-in autopilot + YOLO

- **Autopilot** ≈ "decide and act autonomously, don't keep checking with me" — closest *behavior*, but unbounded.
- **YOLO** (`--allow-all-tools`) ≈ permission bypass; orthogonal to looping.
- **`goal` (this skill)** = autopilot **scoped to a user-defined stop condition**, with explicit active/achieved/cleared/aborted state and a `task_complete` contract.

## State storage

A singleton row in the session's SQL store:

```sql
CREATE TABLE IF NOT EXISTS goal_state (
  id          INTEGER PRIMARY KEY CHECK (id = 1),
  condition   TEXT NOT NULL,
  status      TEXT NOT NULL DEFAULT 'active',   -- active | achieved | cleared | aborted
  set_at      TEXT NOT NULL,
  achieved_at TEXT,
  iterations  INTEGER NOT NULL DEFAULT 0,
  last_check  TEXT,
  notes       TEXT
);
```

State lives only for the current Copilot CLI session — re-set after each new session if you need continuity (Claude Code's `/goal` behaves the same way).

## License

MIT — see [LICENSE](LICENSE).

## Credits

Inspired by Anthropic's [Claude Code](https://www.anthropic.com/claude-code) `/goal` slash command. This is an independent, behavior-compatible port for GitHub Copilot CLI.
