---
name: productive-thinking
description: Facilitates Tim Hurson's 6-step Productive Thinking Model for hard engineering problems (architecture decisions, refactors, feature design), grounded in the actual codebase, and turns the finished session into a Technical RFC. Use when the user wants to think a problem through step by step, run a structured design session, or write an RFC from one.
argument-hint: "[rfc] [problem statement]"
# Manual-only: a 6-step session shouldn't start on its own mid-conversation.
# Remove this line if you want Claude to suggest/start it automatically.
disable-model-invocation: true
---

# Productive Thinking

Facilitate a structured design session using Tim Hurson's Productive Thinking Model, then optionally write it up as an RFC.

## Input

$ARGUMENTS

- If the input starts with `rfc`, run **RFC mode**. Anything after it is an optional title or output path.
- Otherwise run **interactive mode**, treating the input as the problem statement. If it's empty, ask the user to describe the problem in a sentence or two before doing anything else.

## How to facilitate

- **Ground everything in the code.** Before each step, search and read the files, schemas, configs, and docs relevant to it, and cite paths when you rely on them. The value of this session over a whiteboard chat is that conclusions rest on what the system actually does, so verify instead of assuming. For broad sweeps across many files, delegate to an Explore subagent so the session's context stays focused on the discussion.
- **Keep the user's load light.** Ask at most 5 questions per turn, and pair them with your own suggested answers based on what you found, so the user can reply "yes, mostly, but…" instead of starting from a blank page.
- **One step per turn.** Open each step with a header like `Step 2 of 6 — What's success?`, do the step, then stop and wait. Move on only when the user is ready; let them skip, revisit, or jump between steps.
- **Keep a running notes file.** After each step, append the agreed outcome to `<rfc-dir>/<slug>.notes.md` (choose `<rfc-dir>` as described in RFC mode, step 3). Long sessions get compacted or resumed later; the notes file is what lets RFC mode, or a new session, pick up without losing decisions. Tell the user where it is the first time you create it.

## The six steps

### 1. What's going on?
Goal: shared understanding of current reality and the target future.
Explore existing implementations, call sites, data models, and recent git history of the affected areas.
Cover: the pain point or bottleneck (with evidence such as errors, metrics, or code), the impacted modules and services, who is affected, and what "solved" looks like.

### 2. What's success? (DRIVE)
Goal: criteria every option will be judged against. Propose a draft from Step 1 and let the user correct it.
- **Do**: what the solution must do
- **Restrictions**: hard constraints (backward compatibility, performance budgets, stack, compliance)
- **Investment**: time, people, and complexity the team is willing to spend
- **Values**: engineering principles that break ties (simplicity, testability, …)
- **Essential outcomes**: non-negotiable results, measurable where possible

Also capture explicit non-goals.

### 3. What's the question?
Propose 2–3 "How might we…" questions that frame the challenge differently, for example one narrow and one that challenges an assumption from Step 1. The user picks or rewrites one; it becomes the problem statement for the rest of the session.

### 4. Generate answers
Propose 2–3 genuinely distinct approaches grounded in the codebase, not variations of one idea. For each: a short sketch, the files or modules it touches, and its main risk. Hold back judgment in this step and invite the user's own ideas.

### 5. Forge the solution
Score each approach against DRIVE (a small table works well), recommend one, and name the trade-off you're accepting. Then harden it: failure modes, edge cases, migration concerns, open questions. Record why the other options lost; that becomes "Alternatives considered".

### 6. Align resources
Agree on the delivery shape, not individual tickets: phases or milestones and what each makes demonstrable, the riskiest part to prove first, feature-flag and migration strategy, testing strategy, rollout and rollback, and what to monitor. The detailed ticket breakdown comes later from `/task-breakdown`, which works from the RFC. End by offering to generate the RFC.

## RFC mode

1. Find the session: use the conversation so far plus the notes file, if one exists. If there is no session at all, say so and offer to start one: ask for the problem statement and begin Step 1.
2. If steps are missing, list which ones and ask whether to fill them in now or draft the RFC with those sections marked **TBD**. Don't invent decisions the user didn't make.
3. Choose the output location. RFCs, notes, and tasks files are the developer's local working documents: **untracked by default**, not committed with the code. Look for an existing local, git-ignored folder for them (check `.gitignore` and `.git/info/exclude`) and follow its naming. If there isn't one, propose `.rfcs/YYYY-MM-DD-<slug>.md` and add `.rfcs/` to `.git/info/exclude`, which ignores it locally without changing the repo's `.gitignore`. Use a tracked folder such as `docs/rfcs/` only if the user says the team commits RFCs.
4. Write the RFC using [templates/rfc.md](templates/rfc.md), filling each section from the session and tying codebase claims to file paths.
5. Tell the user the path, and list any TBD sections and open questions.
6. Offer the next step: `/task-breakdown <rfc-path>` to turn the RFC into 1-story-point tickets the team can pick up.
