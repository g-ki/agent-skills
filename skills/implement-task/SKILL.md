---
name: implement-task
description: Implements one ticket (from the team's tracker such as Linear, or a local tasks file) as the smallest complete change, test-first from its acceptance criteria, and opens a PR for review; can also address review feedback on that PR. Use only when a specific ticket (such as ENG-123 or T-04) is named for implementation, by the user or by the deliver-project orchestrator.
argument-hint: "<ticket> [--tasks <file>] [--base <branch>] [--branch <name>] [--feedback <pr>] [--unattended]"
---

# Implement Task

Deliver one 1-point ticket as a PR for review, containing exactly what the ticket asks for and nothing more.

## Input

$ARGUMENTS

- **Ticket**, required: a tracker key (e.g. `ENG-123`) or a local ID (e.g. `T-04`). If missing, list the ready tickets (status `open`, dependencies merged), critical-path first, and ask which to take. In unattended mode, report `blocked` instead.
- **`--tasks <file>`**: the local tasks file. If not given, look for the `*.tasks.md` containing the ticket in the project's local, git-ignored RFC folder (e.g. `.rfcs/`), then `docs/rfcs/`.
- **`--base <branch>`**: the branch to start from and target with the PR. Defaults to main. A non-main base means the PR is **stacked** on another ticket's unmerged PR.
- **`--branch <name>`**: the branch name to use. Otherwise follow the naming rule in step 1.
- **`--feedback <pr>`**: don't implement; address review feedback on an existing PR (see the last section).
- **`--unattended`**: you were started by an orchestrator and no one can answer questions. Never ask; when you would ask, stop and explain in the report. Do your work in your own worktree.

## Where the ticket and local files live

- **Tracker first.** If the ticket has a tracker key (as the argument, or in the tasks file's `Tracker:` field), fetch it through the connected tracker tool (Linear, Jira, GitHub Issues, …). When published, the tracker is the source of truth for the ticket's content; if it's canceled or done there, stop and report.
- **Local tasks file otherwise**, and for fields the tracker issue lacks (files, out of scope). RFCs and tasks files are the developer's local, untracked documents, so they exist only in the **main project folder**, not in a worktree. If you're in a worktree, find the main folder with `git worktree list` (the first entry) and read the files there by absolute path. Treat the main folder as **read-only**: never edit, check out, or run git commands in it.
- Never commit RFC, notes, or tasks files. Stage files by explicit path, not `git add -A`.

## 1. Pick the ticket

- Read the ticket (see above), including its acceptance criteria and dependencies. If its type is `spike`, stop: spikes produce a decision, not a PR; point to `/spike-task` (unattended: report `blocked`).
- A dependency is satisfied when its PR is merged into main (check the PR state with `gh`, or the dependency's status `pr-merged`/`done`), or when the base branch is that dependency's own branch (stacking). A spike dependency is satisfied only when it's `done`, meaning its decision was approved and the backlog refined. If any dependency is unsatisfied, stop and say which.
- Make sure the working tree is clean, fetch, and create the branch from the latest `origin/<base>`.
- **Branch name** (unless `--branch` was given): `<username>/<project-slug>/<task-branch>`.
  - `username`: the prefix of the user's existing branches if they follow this pattern, otherwise their GitHub login (`gh api user --jq .login`).
  - `project-slug`: from the tasks file's `Project slug:` header, or the RFC name in kebab case.
  - `task-branch`: the tracker key plus a short slug, e.g. `eng-123-add-v2-flag` (the key lets tracker integrations link the PR), or `t-04-add-v2-flag` if unpublished.
- If you're working directly with the user (not unattended), set the ticket's status to `in progress` in the tasks file, and in the tracker if its GitHub integration won't do it.

## 2. Load only what the ticket needs

Read the ticket, the RFC section it relies on, and the files it names. Then trace the real flow: callers, existing helpers, and the tests next to the code you'll change. Read enough to understand the problem fully, but don't load the whole RFC or session history; the ticket was written to stand on its own. If it doesn't, that's a gap to report, not to guess around.

## 3. Write the tests first

Turn each acceptance criterion into a test, using the repo's existing test framework, location, and style. Run them and confirm they fail for the right reason. One focused test per criterion is enough: no new test utilities, fixtures, or helpers unless an existing pattern requires them. Purely mechanical tickets (e.g. adding a flag) may need no new test if the criterion is checked by existing tests or the type checker; say so in the PR.

## 4. Implement the smallest complete change

"Complete" means every acceptance criterion passes. "Smallest" means nothing beyond that. This follows the [ponytail](https://github.com/DietrichGebert/ponytail) approach; if a `ponytail` skill is installed, load it now at its default level and let it govern this step.

Otherwise, take the first option that works:

1. Is the need already met? If existing code already satisfies the criteria, stop and propose closing the ticket.
2. Reuse what the codebase already has: helpers, types, patterns.
3. Use the standard library, then native platform features (DB constraints, built-in HTML inputs, CSS before JS), then dependencies already installed.
4. Only then write new code, as little as works.

While writing it:

- **No speculative structure.** No interfaces with one implementation, no factories for one product, no config for values that never change, no "for later" scaffolding. Prefer editing an existing file to creating one.
- **Match the surrounding code**: naming, error handling, structure. The diff should look like the codebase wrote it.
- **Comments only where the code can't speak.** Don't narrate what the code does, restate the ticket, or leave commented-out code or debug output. The one comment worth adding is at a deliberate simplification: name the limit and when to upgrade it, so nobody "fixes" it by accident.
- **Fix bugs at the root.** Check every caller before changing a shared function.
- **Never minimize safety.** Input validation at trust boundaries, error handling that prevents data loss, security, and accessibility basics are always done properly, even if the ticket doesn't spell them out.

## 5. Guard the scope

If the ticket turns out bigger than 1 point (more than about a day, a second concern appears, or a criterion can't be met without touching unrelated areas), stop. Commit nothing beyond what's clean, explain what you found, and propose a split as new tickets for approval: write them into the tasks file when working with the user, or into the report when unattended. Unrelated problems you notice go in the PR's notes, not into the diff. Never weaken or reinterpret an acceptance criterion to make it pass.

## 6. Verify and open the PR

1. Run the tests, linter, and type checker the way the repo does. If the change has runtime behavior and you can run `/verify`, do; otherwise exercise the behavior with the project's own run or check commands.
2. Re-read the full diff yourself: every changed line should trace to an acceptance criterion. Remove unrelated formatting, renames, and stray files, and make sure no local RFC or tasks file slipped in.
3. Commit following the repo's commit style, push, and open the PR **ready for review** (not draft, so review automation runs, unless the user asked for a draft) against the base branch, using [templates/pr.md](templates/pr.md). Put the tracker key in the title and link the tracker issue in the description. If the base isn't main, say which PR it's stacked on and that that one merges first. If you can't open PRs from here, push the branch and include the PR text in your reply.
4. If you're working directly with the user, set the ticket's status to `pr-open` (or `pr-draft`) and fill in `Branch / PR:` in the tasks file, and move the tracker issue to its review state if its GitHub integration won't. Unattended, leave statuses to the orchestrator.
5. Finish with the report below. With a user present, a two-or-three-line version is enough.

## Addressing PR feedback (`--feedback`)

You get a PR and a list of review items to address (comment links plus what to do).

1. Fetch and check out the PR's branch in your worktree. Read the ticket and the items; read the current diff.
2. Address each item with the smallest change that resolves it, under the same rules as step 4. Stay within the ticket's scope: if an item asks for something beyond it, don't build it; report it as a follow-up.
3. Add **new commits** rather than amending or force-pushing, so reviewers can see what changed since their review. Run the tests, then push.
4. Reply on each addressed comment thread with one line saying what changed and the short commit SHA. For items you were told to answer without changing code, reply with the given text. If you conclude an item shouldn't be changed, don't argue in the thread; leave it and explain in your report.
5. Finish with the report, using `done` if every item was handled, `blocked` otherwise.

## Report

End your final message with this block exactly, so an orchestrator can parse it:

```
RESULT: done | blocked | needs-split | already-met
TICKET: <local id> / <tracker key or "unpublished">
PR: <url or "none">
BRANCH: <branch or "none">
SKIPPED: <deliberate limits or items not addressed, or "none">
NOTES: <one or two lines: why blocked, what to split, follow-ups, unrelated issues found>
```
