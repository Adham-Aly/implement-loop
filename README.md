# implement-loop

An **agent skill** that keeps your main agent session lightweight. `/implement-loop <task>` runs the task through three sequential phases — **planning → implementation → review** — each owned by its own orchestrator subagent at your session's model. The main session never plans, implements, tests, or reviews: it creates a branch, briefs and spawns one orchestrator per phase, gates each phase's report, and talks to you. Every orchestrator may fan out to up to 4 subagents of its own — always plain general-purpose agents whose only instructions are the orchestrator's prompt, never a predefined agent type.

Works with any coding agent that supports the SKILL.md agent-skills format — Claude Code, Codex, Cursor, OpenCode, and the rest.

```
main session ──▶ 1. planning orchestrator ───────▶ up to 4 subagents
(coordinates,    2. implementation orchestrator ─▶ up to 4 subagents
 talks to you)   3. review orchestrator ─────────▶ up to 4 subagents
                    (one per phase, in sequence)     (no further nesting)
```

Phases hand context to one another through a temporary `.implement-loop/` folder in the workspace, not through the main session's context.

## Install

```bash
npx skills add Adham-Aly/implement-loop
```

The [skills CLI](https://github.com/vercel-labs/skills) auto-detects the agents on your machine and asks which to install for (or target one explicitly, e.g. `-a claude-code`). Installs into the current project by default; add `-g` for a global install. Start a new agent session (or reload its skills) to pick it up.

## Usage

```
/implement-loop <anything — a feature, a plan you've discussed, a bug, a tiny change>
/implement-loop <task> — grill me   # have the planner question you before it plans
/implement-loop review              # run another review pass on the current run
```

User-invoked only: the skill is marked `disable-model-invocation`, so agents that honor that flag never trigger it on their own. Invoked with no argument, it asks what to implement.

### What a run does

1. **Branch** — creates and switches to a short, descriptive branch for the task (`csv-export`, not `implement-loop/csv-export`), pushing it to the remote when there is one.
2. **Working folder** — creates `.implement-loop/` with `task.md` (the task in full) and a self-ignoring `.gitignore`.
3. **Planning** — the planning orchestrator investigates the codebase (read-only) and writes `plan.md` plus its context file `planning.md`. If you asked to be grilled, it stops to question you first (see below).
4. **Implementation** — the implementation orchestrator executes the plan, keeps docs/context files (AGENTS.md, CLAUDE.md, …) up to date, lints, and writes `implementation.md`.
5. **Review** — the review orchestrator works out how this particular codebase is tested (CI config, test/lint/typecheck commands, rules in AGENTS.md / CLAUDE.md / repo skills), runs those checks, exercises the change end to end — headlessly where possible — reviews the diff, fixes genuine defects, and writes `review-1.md`. Its subagents are read-only: they test, investigate, and report concerns, and the orchestrator alone confirms which are real and makes every fix.
6. **Close** — the main session summarizes the run and **offers** to commit and push. Nothing is committed without your say-so.

Ask for `review` again as many times as you like: each pass is a fresh orchestrator that reads all earlier `review-N.md` files (so it doesn't repeat work) and writes its own `review-N+1.md`.

### Grilling

Add "grill me" (or words to that effect) to your invocation and the planning orchestrator will ask rather than assume: once it has investigated enough to know what is genuinely open, it stops and hands the main session a batch of questions, each with options. The main session relays them to you exactly as written — through a structured question tool if the agent has one, otherwise in chat — records your answers in `task.md`, and sends them back to the same orchestrator, which only then resumes. It may grill you more than once if your answers open new questions, but it is told to prefer one thorough round over many small ones. The main session never rewords, filters, or answers the questions itself.

### The `.implement-loop/` folder

> **Important:** `.implement-loop/` is scratch space for one run. It must be deleted before the work is committed — the main session does this automatically when you accept its offer to commit, and you should do the same if you commit by hand or through another agent. Its `.gitignore` keeps it out of git in the meantime.

| File | Written by | Contents |
|---|---|---|
| `task.md` | main session | the task in full + run constraints (+ grilling questions and answers, if any) |
| `plan.md` | planning orchestrator | the plan, including how the review phase should verify the change |
| `planning.md` | planning orchestrator | phase context |
| `implementation.md` | implementation orchestrator | phase context |
| `review-N.md` | review orchestrator N | phase context, one per review pass |

Context files are written for the next orchestrator, not for people: bullets only, strictly concise.

### Defaults — override any of them in your invocation

| Default | Override example |
|---|---|
| Three phases, one orchestrator each, flowing automatically | "pause after planning so I can approve the plan" |
| The planner resolves ambiguity by assumption and records it | "grill me" — the planner questions you before writing the plan |
| Each orchestrator may spawn up to 4 plain general-purpose subagents and honestly decides how many and in what order (parallel, sequential, mixed); zero is allowed. Review subagents never edit files | "review orchestrator: at most 2 subagents" / "implementation: exactly 4, all in parallel" — per orchestrator or for all |
| Subagents may not spawn subagents (main session → orchestrator → subagent is the limit) | "let the review orchestrator's end-to-end subagent spawn up to 2 helpers" |
| Orchestrators and subagents inherit their parent's model and effort | "planning orchestrator on model X, high effort" / "implementation subagents on model Y" — any mix; anything unspecified inherits |
| Only the main session touches git: the task branch at the start, commit + push only after you approve | "stay on the current branch" / "commit and push when done without asking" / "let the implementation orchestrator commit" |

## Repo layout

```
skills/implement-loop/
├── SKILL.md                        # main-session workflow
├── planning-orchestrator.md        # brief template for the planning phase
├── implementation-orchestrator.md  # brief template for the implementation phase
└── review-orchestrator.md          # brief template for the review phase
```

## License

[MIT](LICENSE)
