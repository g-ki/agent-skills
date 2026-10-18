---
name: spike-task
description: Runs a time-boxed research spike for one ticket of type spike (from the team's tracker such as Linear, or a local tasks file) and produces a decision record with evidence, a recommendation, and proposed backlog changes, without shipping code. Use only when a specific spike ticket (such as ENG-130 or T-03) is named, by the user or by the deliver-project orchestrator.
argument-hint: "<ticket> [--tasks <file>] [--branch <name>] [--unattended]"
---

# Spike Task

Answer one question well enough to make a decision, within the spike's time box. The deliverable is a **decision record**, not merged code.

## Input

$ARGUMENTS

- **Ticket**, required: a tracker key (e.g. `ENG-130`) or a local ID (e.g. `T-03`) of a ticket with type `spike`. If it isn't a spike, stop and point to `/implement-task`.
- **`--tasks <file>`**: the local tasks file. If not given, find the `*.tasks.md` containing the ticket in the project's local, git-ignored RFC folder (e.g. `.rfcs/`), then `docs/rfcs/`.
- **`--branch <name>`**: branch name for a prototype, if one is needed. Otherwise `<username>/<project-slug>/spike-<task-branch>`, following the same naming rule as `/implement-task`.
- **`--unattended`**: started by an orchestrator; no one can answer questions. Never ask; when you would ask, stop and explain in the report. Work in your own worktree.

Read the ticket from the tracker if it's published (the tracker is the source of truth for its content), otherwise from the tasks file. Local RFC and tasks files live only in the main project folder: if you're in a worktree, find it with `git worktree list` (first entry) and read them there by absolute path. Never edit files or run git commands in the main folder, except writing the decision record as described below.

## 1. Frame the question

Before gathering anything, write down:

- **The question**, as one sentence that has an answer ("Can Postgres LISTEN/NOTIFY handle 2k events/s on our instance size?", not "look into eventing").
- **The decision it unblocks**, and which tickets depend on it.
- **The options** being decided between, including "do nothing" or the simplest option when it's plausible.
- **Decision criteria**, taken from the RFC's DRIVE section where possible (restrictions, values, essential outcomes), so evidence is judged against what the team already agreed matters.
- **The time box** from the ticket (default: about a day of effort). Done means the question is answered or the time box is spent, whichever comes first.

If the ticket is too vague to frame a question, stop and report `blocked` with the questions that need answers.

## 2. Gather evidence, cheapest first

Stop as soon as the evidence settles the question; don't research for completeness.

1. **The codebase**: existing patterns, constraints, prior attempts, comments and history around the area (`git log` on relevant paths).
2. **Primary sources**: official docs, changelogs, release notes, issue trackers of the libraries or services involved. Prefer these over blog posts. If a `deep-research` skill is available and the question needs broad external research, use it.
3. **Measure**: when reading can't settle it (performance, compatibility, "does this API really behave like that"), build the **smallest prototype or benchmark** that answers the question, on the spike branch, from the latest main. It exists to produce evidence: no tests beyond what the measurement needs, no polish, and it is never merged. Push the branch so the evidence is reproducible, but don't open a PR.

Keep track of how you know each thing: **measured** (you ran it), **documented** (a primary source says so), or **assumed** (inference). Cite links and file paths.

## 3. Decide what to recommend

- Score each option against the decision criteria, using the evidence.
- Recommend one option, with a **confidence**: high (measured or well documented), medium (documented but not tested in our context), low (mostly inference).
- When confidence is low, prefer the option that's cheapest to reverse, and say what signal would make you switch.
- If the time box ran out before the question was answered, say so plainly: report what's known, and recommend the smallest next step (a narrower follow-up spike, or a reversible default to proceed with).

## 4. Propose backlog changes

The decision usually changes the plan. Propose, don't apply:

- **Tickets to add**, in the same format as the tasks file (title, scope, files, acceptance criteria, dependencies), each 1 story point.
- **Tickets to change** (scope, criteria, or dependencies) and **tickets to remove**, each with the reason.
- **RFC sections** that need updating.

## 5. Write the record and report

1. Write the decision record using [templates/decision.md](templates/decision.md) to `<rfc-name>.<ticket-id>.decision.md` next to the tasks file in the main project folder (the one file you write there). It's a local, untracked file like the RFC.
2. If the ticket is published, post a short summary on the tracker issue: the question, the recommendation and confidence, and the key evidence in three to five lines, clearly marked as **proposed, pending approval**.
3. If you're working directly with the user, set the ticket's status to `decision-proposed` in the tasks file, walk them through the recommendation, and ask them to approve, pick another option, or extend the spike. Unattended, leave statuses to the orchestrator.
4. Finish with the report:

```
RESULT: decision-proposed | inconclusive | blocked
TICKET: <local id> / <tracker key or "unpublished">
RECORD: <absolute path to the decision record>
RECOMMENDATION: <one line> (confidence: high | medium | low)
PROTOTYPE: <branch or "none">
NOTES: <backlog changes in one line: +N tickets, ~N changed, -N removed; or why blocked>
```
