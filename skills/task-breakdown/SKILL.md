---
name: task-breakdown
description: Breaks an RFC, design doc, or finished productive-thinking session into small 1-story-point tickets a team can pick up independently, with acceptance criteria, files to touch, and dependencies. Use when the user wants to split a design or feature into tickets, stories, or tasks, plan a sprint from an RFC, or asks "how do we deliver this".
argument-hint: "[path to RFC or design doc]"
---

# Task Breakdown

Turn a design into a backlog of 1-story-point tickets that any engineer on the team can pick up, finish, and merge on their own.

## Input

$ARGUMENTS

- If a path is given, read that document.
- Otherwise, use the most recent RFC or `*.notes.md` from a productive-thinking session in this conversation or on disk. These are usually local, untracked files (look in git-ignored folders such as `.rfcs/`, then `docs/rfcs/`, `docs/adr/`, `rfcs/`). If you find none, ask which document or design to break down.
- If the source has gaps that block slicing (no chosen design, no rollout approach), list them and ask before inventing answers. Small gaps can become spike tickets instead.

## What "1 story point" means here

A ticket is 1 point when **all** of these hold. If one fails, split it:

1. **One person, about a day or less**, including tests and review fixes.
2. **One reviewable PR** with a single purpose. A reviewer can understand it without reading other open PRs.
3. **Mergeable on its own.** Main stays green and shippable after it lands, which usually means new behavior sits behind a flag or isn't wired up yet.
4. **Testable.** It has 1–5 acceptance criteria that someone other than the author can check.
5. **No hidden unknowns.** If the implementer would have to research before knowing what to do, it's a spike, not a story.

If the team has its own definition of a point, follow theirs and note it at the top of the output.

## How to slice

Read the code before slicing: confirm the files, modules, schemas, and tests each ticket touches actually exist, and name them in the ticket. Tickets that cite real paths get picked up; vague ones sit in the backlog.

Start with a **walking skeleton**: the thinnest end-to-end slice that proves the design works in production (often behind a flag), then thicken it. This front-loads risk instead of discovering integration problems in the last ticket.

When a ticket is too big, these splits usually work:

- **Prepare, then change.** Pull out a pure refactor first ("make the change easy, then make the easy change"), with tests pinning current behavior before the refactor.
- **Expand / migrate / contract** for schema and API changes: add the new column or endpoint, backfill or dual-write, switch reads, then remove the old path. Each phase is its own ticket.
- **By layer behind a flag**: data model → service logic → API → UI, each mergeable because the flag is off.
- **Read path before write path**, and **happy path before edge cases and error handling**.
- **By variant**: one ticket per provider, file type, or client, after the first establishes the pattern.
- **Spikes for unknowns**: time-boxed (≤ 1 day), and the output is a decision plus follow-up tickets, not production code. Give each spike a `Question:` that has an answer, and a time box. Tickets that depend on the decision get sliced only as far as is safe, with a note that they'll be refined once the spike is decided. Put spikes early: they gate everything behind them.

Don't forget the work that isn't feature code: feature-flag setup and removal, metrics/alerts/dashboards, runbook or docs updates, data backfills, staged rollout steps, and cleanup of the old path. These are usually where "almost done" projects stall.

Don't split for the sake of it: a ticket that only makes sense merged together with another should be one ticket, or the pair should be re-cut along a different seam.

## Output

1. Write the backlog using [templates/tasks.md](templates/tasks.md) to `<rfc-name>.tasks.md` next to the source RFC. Like the RFC, it's a local working file, untracked by default; if its folder isn't git-ignored yet, add it to `.git/info/exclude`. Number tickets `T-01`, `T-02`, … so dependencies can reference them, set every status to `open`, and fill in the project slug used for branch names. Keep the field names exactly as in the template, since `/implement-task` and `/deliver-project` read them.
2. Group tickets into milestones that match the RFC's phases, each ending in something demonstrable or shippable.
3. Include the dependency picture: which tickets can run in parallel, and the critical path. A small Mermaid graph works well when there are more than ~8 tickets.
4. In chat, give a short summary: ticket count per milestone, how many can start immediately, the critical path, spikes, and any ticket you're unsure is really 1 point. Ask the user to flag tickets that look too big, too small, or wrongly ordered, and revise.
5. **Publish to the team's tracker.** Assume the team uses one (Linear, Jira, GitHub Issues, …). Once the user is happy with the breakdown, offer to publish it through the connected tracker tool; if none is connected, say which connector would help, and offer a CSV with columns `ID, Title, Description, Acceptance Criteria, Story Points, Milestone, Depends On, Labels` as a fallback. Publish only after the user confirms, since it's visible to the whole team. For each ticket, create an issue with the title, a description that stands alone (context, scope, files, out of scope, acceptance criteria), estimate 1, the milestone, and blocked-by relations for its dependencies where the tracker supports them. Then write each issue's key and link into the ticket's `Tracker:` field in the local file.
6. Point to the next step: `/implement-task <ticket-id>` for a single ticket, `/spike-task <ticket-id>` for a spike, or `/deliver-project <tasks-file>` to run the whole backlog with a team of agents.

If the source RFC has an execution-plan section, update it to link the tasks file instead of duplicating the list.

## Refining after a spike

When given a decided spike's decision record (status `decided`), update the backlog instead of starting over:

1. Apply the record's approved backlog changes: add tickets (continue the `T-NN` numbering, never reuse IDs), change scope, criteria, or dependencies of affected tickets, and mark removed tickets `canceled` with the reason rather than deleting them.
2. Re-check every changed or new ticket against the 1-point test, and update the summary, dependency graph, and critical path.
3. Update the RFC sections the record lists.
4. Sync the tracker: create the new issues, update changed ones, and cancel removed ones, writing new keys into the tasks file. If the user approved the decision and its backlog changes together, that approval covers this sync; otherwise confirm first.
5. Set the spike to `done` and summarize what changed in a few lines.
