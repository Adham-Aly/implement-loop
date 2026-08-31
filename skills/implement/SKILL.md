---
name: implement
description: Delegate an implementation task to one or more implementation-orchestrator subagents so the main session stays lightweight and big-picture. Takes the user's task as its argument.
disable-model-invocation: true
argument-hint: [task to implement + any modifiers (subagent budget, git permissions, orchestrator count/split)]
---

# implement

You (the main session) delegate the user's implementation task to an **implementation orchestrator** — a subagent that owns the actual work. Your job is coordination and quality control, not implementation. The philosophy: keep your own context load minimal so you stay focused on the big picture and on making sure the orchestrator(s) operate properly.

## Step 0 — the task

The user's request is everything they passed when invoking the skill — `$ARGUMENTS`.

- If the argument is empty: stop and ask the user what they want implemented. Never infer the task from conversation history on your own; an explicit task is required before anything is spawned.
- If it references something from the current conversation ("implement the plan we agreed on"), that referenced material is part of the task — you must pass its full substance along, because the orchestrator cannot see this conversation.

## Step 1 — parse modifiers

Scan the user's request for modifiers that change the defaults. The user may override any of these at invocation time:

| Default | User can override with |
|---|---|
| 1 implementation orchestrator | a specific number of orchestrators, and optionally how to split the work and/or which parts run parallel vs sequential |
| Orchestrator may spawn up to 5 subagents | a different maximum, a minimum, or an exact number (applies per orchestrator unless they say otherwise) |
| No state-changing git by anyone | explicit permission for specific git actions (commit, branch, push, ...) |
| No commits/branches/pushes by you | an explicit request for them |

Anything the user asks for that deviates from a default must be woven into the orchestrator brief(s) — the orchestrator only knows what you tell it.

## Step 2 — brief and spawn the orchestrator(s)

Read [orchestrator-brief.md](orchestrator-brief.md) and compose each orchestrator's prompt from that template.

Spawn mechanics:

- Use your environment's subagent-spawning tool. If it takes an agent type, pick the general-purpose one — never a custom agent type, even one with "orchestrator" in its name.
- Spawn the orchestrator at this session's own model/effort level wherever your environment allows it — e.g. don't pass a model override that would downgrade it; inheriting the session's model is typically the default.
- **Single orchestrator (the default):** hand it the entire task.
- **Multiple orchestrators (only when the user asked for them):**
  - Honor the user's split and ordering if they gave one; that always takes precedence.
  - Otherwise split the work yourself into scopes with disjoint file ownership. Run orchestrators in parallel wherever the work is genuinely independent (spawn those in a single message); where a piece genuinely depends on another's output, stage it sequentially after them — mixed topologies are fine (e.g. 3 in parallel, then a 4th that builds on their output).
  - Give a sequential orchestrator a concise summary of what the earlier ones did.
  - Each orchestrator gets its own subagent budget (default: up to 5 each).
  - If several scopes would touch the same context/doc file (AGENTS.md etc.), assign that file's update to exactly one of them, or reconcile it yourself at review.

## Step 3 — while they run

Stay lightweight. Do not implement, explore the codebase in depth, or duplicate the orchestrator's work. Handle anything the user says in the meantime; otherwise wait for the report(s).

## Step 4 — review and close

When an orchestrator reports back:

1. Read the report critically: was the full scope completed? Are the flagged assumptions reasonable? Did lint and the sanity review come back clean?
2. Verify documentation upkeep: spot-check that context files (AGENTS.md / CLAUDE.md / README / affected skills / similar) were actually updated where the code changes made that necessary. The orchestrator is required to have done this completely; if it missed something, fix that omission yourself — this is the one kind of edit you make directly.
3. If something substantial is wrong or incomplete, don't rework it yourself: send the same orchestrator a follow-up with a precise description of what to fix (or spawn a fresh orchestrator briefed on the gap).
4. Report to the user: what was implemented, files touched, assumptions/flags, lint + sanity-review results, doc updates made, and anything left out and why.

Hard boundaries for you in this workflow:

- **No tests.** Don't run them and don't ask anyone to run them. Testing is a separate workflow, entirely outside this skill's scope.
- **No state-changing git** (add/commit/branch/checkout/push/...) unless the user explicitly asked for it in this invocation. Read-only git is fine.
