---
name: implement-loop
description: Run a task through three delegated phases — planning, implementation, review — each owned by its own orchestrator subagent, with all shared context kept in a temporary .implement-loop/ folder so the main session stays lightweight. Takes the task as its argument; "review" reruns the review phase.
disable-model-invocation: true
argument-hint: "[task + any modifiers (grill me, subagent budgets, models/effort, nesting, git)] | review [modifiers]"
---

# implement-loop

You (the main session) run the user's task through three sequential phases — **planning → implementation → review** — each owned by a fresh orchestrator subagent. You never plan, implement, test, or review anything yourself. You create the branch and the working folder, brief and spawn one orchestrator per phase, gate each phase's report, and talk to the user. Everything the orchestrators need from one another lives in `.implement-loop/`, not in your context.

## Step 0 — the task

The user's request is everything passed with the invocation — `$ARGUMENTS`.

- Empty → stop and ask what to implement. Never infer the task from conversation history; an explicit task is required before anything is spawned.
- `review` (or an equivalent request to run the review phase again, whether in the invocation or later in the conversation) → go to **Review rerun** below.
- If it references something from the conversation ("implement the plan we discussed"), that material is part of the task and must go into `task.md` in full — orchestrators cannot see this conversation.

## Step 1 — modifiers

The defaults below hold unless the user's request changes them. Each override is written into exactly the brief(s) it concerns; an orchestrator knows only what its brief says. Expect the defaults; do not wait for or assume modifiers the user did not give.

| Default | The user may override with |
|---|---|
| Three phases, exactly one orchestrator each, run in order and flowing automatically | a pause after planning for their approval; extra instructions for any phase |
| The planning orchestrator resolves ambiguity by assumption and records it | "grill me" (or any request to be asked questions before the plan is made): grilling is **on** — the planning orchestrator stops to question the user, as often as it needs, and you relay verbatim (Step 4, **Grilling**). Off unless asked for |
| Each orchestrator may spawn up to 4 subagents and decides count and ordering itself | a maximum, minimum, or exact count and/or ordering instructions — for one orchestrator, several, or all, each possibly different |
| Subagents may not spawn subagents (nesting ends at main session → orchestrator → subagent) | permission for specific subagents of specific orchestrators to nest further, with whatever limits they state |
| Every orchestrator and subagent inherits its parent's model and effort | model/effort per orchestrator or per subagent group, in any mix; anything unspecified inherits |
| Git: you create the task branch at the start; no other state-changing git by anyone; commit + push is only offered at the end | a different branch policy; permission to commit without asking; git permissions for an orchestrator |

## Step 2 — branch

If the workspace is a git repository:

1. Pick a short kebab-case name that describes the task itself (e.g. `csv-export`, `fix-login-redirect`). No `implement-loop/` or similar prefix.
2. Create it from the current HEAD and switch to it: `git switch -c <name>`.
3. If a remote exists, publish it: `git push -u origin <name>`. If that fails, continue locally and mention it in your final report.

Do not stash, reset, or discard uncommitted changes. If the user named a branch or asked to stay on the current one, do that instead. No git repository → skip this step and say so at the end.

## Step 3 — working folder

Create `.implement-loop/` at the workspace root. It is the run's shared memory and it is **temporary: it must be deleted before anything is committed** (Step 5). Write two files into it:

- `.gitignore` containing the single line `*`, so the folder can never be committed by accident.
- `task.md` — the task in full (the user's words plus every referenced plan, constraint, preference, and file pointer; do not thin it into a summary), followed by a short **Run constraints** list: the branch name and any user modifier that applies to every phase (e.g. git permissions, "don't touch the auth module").

The orchestrators add the rest:

| File | Written by | Contents |
|---|---|---|
| `plan.md` | planning orchestrator | the plan |
| `planning.md` | planning orchestrator | phase context |
| `implementation.md` | implementation orchestrator | phase context |
| `review-N.md` | review orchestrator N (1, 2, …) | phase context — one new file per review run, never overwritten |

Every context file follows the same contract: written for the next orchestrator, bullets only, strictly concise.

## Step 4 — run the phases

For each phase in order — planning, implementation, review — read the template, compose the prompt, spawn the orchestrator, wait for its report, gate it.

| Phase | Template |
|---|---|
| Planning | [planning-orchestrator.md](planning-orchestrator.md) |
| Implementation | [implementation-orchestrator.md](implementation-orchestrator.md) |
| Review | [review-orchestrator.md](review-orchestrator.md) |

**Composing a prompt:** copy the template's text below its `---` line verbatim. Fill every required `{{PLACEHOLDER}}` (the planning template's `{{GRILL}}` is `on` only when the user asked to be grilled, otherwise `off`). `{{IF_… — …}}` placeholders exist only for user modifiers: when the modifier applies, replace the placeholder with the concrete instruction; when it doesn't, delete the placeholder — the surrounding text already states the default. Never edit a numbered rule beyond what a modifier requires.

**Spawning:** use your environment's subagent-spawning tool with its general-purpose agent type — never a custom agent type, even one with "orchestrator" in its name. Spawn at this session's own model/effort level (don't pass an override that would downgrade it) unless the user assigned that orchestrator a model/effort. One fresh orchestrator per phase; never reuse one across phases.

**While it runs:** stay lightweight — don't explore the codebase, read the diff, or duplicate its work. Handle anything the user says in the meantime.

**Gating a report** (each is a few lines: status, flags needing the user's decision, context file path):

- Status *complete*, no flags → start the next phase.
- Status *questions* (planning, grilling on) → follow **Grilling** below; the phase is not over.
- A flag only the user can settle (a decision that would materially change the work) → ask the user, append their answer to `task.md` under Run constraints, continue.
- Status *partial*, or the report is missing or incoherent → do not fill the gap yourself. Spawn a fresh orchestrator of the same phase briefed on the gap; it reads the phase's existing context file and rewrites it to reflect the combined result. (For the review phase this is simply another numbered review run.)
- After planning, if the user asked to approve the plan first: point them to `.implement-loop/plan.md` and wait.

### Grilling

When grilling is on, the planning orchestrator may end its turn with status *questions*: a numbered list of questions, each with a header, a multi-select flag, and options. It has stopped completely and resumes only when the answers reach it. You are a relay, not a participant:

1. **Put the questions to the user exactly as written** — same wording, same options, same order, all of them. Add nothing, drop nothing, merge nothing, reword nothing, and answer nothing on the user's behalf. Do not think about what the questions mean, look into the codebase to understand them, or form a view on the answers; that is the orchestrator's work and it has already done it.
2. **Use a structured question tool if your environment has one** — a tool that presents a question with selectable options to the user. Map each question's text, header, options, and multi-select flag straight onto it; if the tool takes fewer questions per call than the orchestrator asked, ask them across consecutive calls in the given order. If no such tool exists, print the questions and their options in the chat, verbatim, and ask the user to reply with their answers.
3. **Record the answers** by appending them verbatim to `task.md` under a `## Grilling — questions and answers` heading (create it on the first round; add to it on later rounds): each question number, the question, and the answer — whether a chosen option or free text. Later phases read their decisions from there.
4. **Send the answers back to the same planning orchestrator** as one message, numbered like its questions, verbatim, using your environment's mechanism for continuing a previously spawned subagent with its context intact. Then wait for its next report and gate it as usual: it may grill again — repeat this procedure every time — or finish.
5. **If your environment cannot resume a subagent** that has ended its turn, spawn a fresh planning orchestrator with the same brief instead: `task.md` now carries the answers, and the template tells it to treat them as settled and to continue from the partial `planning.md`.

## Step 5 — close

When the review phase reports, read `implementation.md` and the latest `review-N.md` (concise by contract) and tell the user:

- what was built, and anything left out and why
- what verification the review ran and its results; findings fixed and findings left open
- assumptions and flags from all phases
- the branch name and whether it was pushed

Then **offer** to commit and push. Do not commit, push, or change git state in any other way until the user says yes (unless they pre-authorized it in the invocation). When you do, **delete `.implement-loop/` first** — its job ends when the work is committed. Mention that they can instead ask for another review pass.

## Review rerun

The user may rerun the review phase as often as they like before committing — from this session or a later one, as long as `.implement-loop/` still exists.

1. Confirm `.implement-loop/implementation.md` exists; if not, tell the user there is no run to review.
2. Set N = 1 + the highest existing `review-N.md` number.
3. Compose the review brief with that N and whatever modifiers the user gave for this rerun, spawn a **new** review orchestrator (never re-message a previous one), gate it, and close as in Step 5.

## Hard boundaries for you

- **You never plan, implement, test, or review** — not even a small fix or a doc update. Anything wrong goes back to an orchestrator.
- **Grilling questions pass through you untouched**, in both directions. You never rephrase, filter, explain, or answer them, and you never reason about them.
- **Git:** the branch in Step 2 and a commit/push the user explicitly approved are your only state-changing git actions; read-only git is fine. If the user asks to commit mid-run, warn that `.implement-loop/` will be deleted and later phases lose their context, and proceed only on confirmation.
- **Keep your context small:** orchestrator reports and the context files are all you read from the run.
