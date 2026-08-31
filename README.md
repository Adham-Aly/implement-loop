# implement

An **agent skill** that keeps your main agent session lightweight. `/implement <task>` hands the whole job to an **implementation orchestrator** — a subagent at your session's own model that plans, implements, self-reviews, and reports back, optionally fanning out to up to 5 subagents of its own. Your main session stays big-picture: it composes the brief, reviews the result, and verifies documentation upkeep.

Works with any coding agent that supports the SKILL.md agent-skills format — Claude Code, Codex, Cursor, OpenCode, and the rest.

```
main session ──▶ implementation orchestrator ──▶ up to 5 subagents
 (coordinates)      (implements + integrates)      (implement; no further nesting)
```

## Install

```bash
npx skills add Adham-Aly/implement
```

The [skills CLI](https://github.com/vercel-labs/skills) auto-detects the agents on your machine and asks which to install for (or target one explicitly, e.g. `-a claude-code`). Installs into the current project by default; add `-g` for a global install. Start a new agent session (or reload its skills) to pick it up.

## Usage

```
/implement <anything — a feature, a plan you've discussed, a tiny change>
```

User-invoked only: the skill is marked `disable-model-invocation`, so agents that honor that flag never trigger it on their own. Invoked with no argument, it asks what to implement.

### Defaults — override any of them in your invocation

| Default | Override example |
|---|---|
| One orchestrator gets the whole task | "use 3 orchestrators, split by layer" — work is split with disjoint file ownership, parallel where possible, sequential where genuinely dependent; your requested split/ordering always wins |
| Orchestrator may spawn up to 5 subagents (per orchestrator) | "use at most 2 subagents" / "use exactly 4" / "at least 1" — it always weighs the truly optimal number, and zero is allowed |
| No state-changing git by anyone; read-only git allowed | "commit the result to a new branch" |
| No tests — the run ends with lint + a code-review sanity pass by the orchestrator | (testing is deliberately out of scope; run your own workflow for it) |
| Orchestrator fully updates affected context files (AGENTS.md, CLAUDE.md, affected skills, ...); main session verifies | — |

## Repo layout

```
skills/implement/
├── SKILL.md               # main-session workflow
└── orchestrator-brief.md  # prompt template for the orchestrator
```

## License

[MIT](LICENSE)
