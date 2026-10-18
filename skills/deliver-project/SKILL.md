---
name: deliver-project
description: Orchestrates end-to-end delivery of a task-breakdown backlog with parallel worker agents that run implement-task and spike-task, then shepherds every PR to merge on main - stacking PRs on unmerged dependencies, addressing worthwhile review comments and CI failures, restacking branches as PRs merge, gating work behind spike decisions, and keeping the tracker and local tasks file in sync.
argument-hint: "[tasks-file] [--parallel N] [--max-stack N] [--auto-merge] [--once]"
# Manual-only: this spawns many agents, pushes and force-pushes branches, and comments on PRs.
disable-model-invocation: true
---

# Deliver Project

Run the backlog to completion on main. Every ticket becomes one PR, reviewed by the repo's GitHub review automation and by humans. Your job is to keep work flowing: start tickets as soon as they can start, keep every PR healthy, act on review feedback that's worth acting on, keep stacked branches rebased as their parents merge, and keep ticket statuses accurate.

You are the orchestrator. You read state, decide, delegate, and keep the books. Agents write all code, resolve conflicts, and push. You don't edit code or run rebases in this checkout, so the main project folder stays clean and every change stays traceable.

## Input

$ARGUMENTS

- **Tasks file**: the local `*.tasks.md` from `/task-breakdown` (untracked, usually in a git-ignored folder like `.rfcs/`). If none is given, look there and in `docs/rfcs/`; if several exist, ask.
- **`--parallel N`**: maximum agents working at once, of all kinds. Default 3.
- **`--max-stack N`**: maximum depth of a PR stack. Default 3. Deeper stacks unblock more work but multiply rebases when a parent changes.
- **`--auto-merge`**: when a PR on main is `pr-ready`, enable GitHub auto-merge on it (`gh pr merge --auto` with the repo's usual merge method). GitHub still enforces branch protection and required approvals. Off by default: humans merge.
- **`--once`**: run a single pass, then exit. Use this to run delivery on a schedule, for example `/loop 15m /deliver-project <tasks-file> --once`, instead of keeping one long session open.

## Where state lives

- **Tracker** (Linear, Jira, GitHub Issues, …), for tickets that are published (they have a `Tracker:` key in the tasks file). It's the source of truth for whether a ticket exists and is wanted: a ticket canceled, deleted, or reassigned there is skipped. Move tracker issues through their states as tickets progress, unless the tracker's GitHub integration already does (check after the first PR opens, and don't fight it).
- **Local tasks file**, in the main project folder. It's the source of truth for delivery state, and **you are its only writer during the run**: agents read it but never edit it. Statuses: `open` → `in progress` → `pr-draft` → `pr-open` → `pr-ready` → `pr-merged` → `done`, plus `blocked` and `canceled` with a reason. Spikes go `open` → `in progress` → `decision-proposed` → `decided` → `done`. Keep `Branch / PR:` (or `Decision:` for spikes) filled in.
- **GitHub**, for PR state: open, merged, checks, reviews, mergeability. A dependency is merged when its PR is merged, whatever any file says.
- **Branches** identify this run's PRs: every branch is `<username>/<project-slug>/<task-branch>`, so this run's PRs are the user's PRs whose head branch starts with `<username>/<project-slug>/`. That's how a restarted run finds everything again.
- **Run log**, `<rfc-name>.run.md` next to the tasks file (local, untracked), for what isn't a ticket status: running agents, each stacked branch's parent head SHA, handled comment IDs, feedback rounds, CI fix attempts, escalations, follow-ups. Use [templates/run-log.md](templates/run-log.md). If it's lost, rebuild it from GitHub.

[references/github.md](references/github.md) has the `gh` commands for reading PR state, review threads, and checks.

## 1. Preflight (first pass only)

Stop with a clear message if any of these fails:

1. **Tasks file is sound**: IDs, statuses, existing dependencies, acceptance criteria, a `Project slug:`, no dependency cycles.
2. **Tracker reachable**: if any ticket has a tracker key, the tracker's connector must be available, since you need it to check cancellations and update states. If it isn't, say which connector to enable, and offer to continue using only the local file.
3. **Repo is ready**: clean working tree, a remote, `gh` authenticated with push and PR rights, and main builds and passes tests. Work out `<username>` (the prefix of the user's existing branches, otherwise `gh api user --jq .login`) and the branch prefix.
4. **Agents can work unattended**: background agents surface every permission prompt in this session, so one prompt stalls the team. Recommend auto mode, or allow rules for git (including `push --force-with-lease`), `gh`, the repo's test, lint, and build commands, and the tracker connector.
5. **Skills available**: `implement-task` and `spike-task` are installed where agents can load them, and `task-breakdown` is available to you for refining after decisions.
6. **Review automation**: find the GitHub workflow that reviews PRs, and note its check name and bot login, so you know when its review is finished and which comments are its own.

Then show the plan and get **one** confirmation: ticket count, what can start now and what will stack, parallelism, max stack depth, whether auto-merge is on, and an honest note that each agent is a full session and the run uses significant usage. After that, run unattended.

## 2. Each pass

Do these in order. Bottom-of-stack first matters: a parent's fix or merge changes what its children must do.

### a. Sync
Fetch. List this run's PRs (by branch prefix) with state, base, head SHA, checks, review decision, and mergeability. Refresh published tickets from the tracker and skip any that were canceled. Collect the reports of agents that finished since the last pass (each ends with a `RESULT` block). Reconcile: if GitHub and the tasks file disagree, GitHub wins for PR state; fix the file.

### b. Handle merges
For every PR that merged since the last pass: set its ticket to `pr-merged`, and move the tracker issue to done unless the integration did. Set it to `done` once the tracker shows it closed and any rollout step in the ticket is finished. Every PR stacked directly on the merged one needs restacking onto main and its base changed to main: start one restack agent per affected stack (see [references/stacks.md](references/stacks.md); give the agent its absolute path, `${CLAUDE_SKILL_DIR}/references/stacks.md`).

### c. Move decisions forward
Spikes gate the tickets behind them, and only the user approves a decision, because decisions change the plan.

- For each spike in `decision-proposed`, make sure the user has seen it: put it at the top of your update with the question, the recommendation and confidence, the proposed backlog changes in one line, and the decision record's path. Ask them to approve it, pick another option, or extend the spike. Keep everything else moving meanwhile.
- The user approves by telling you in this session, or between `--once` passes by setting the spike to `decided` in the tasks file (with the chosen option noted in the record). When a spike becomes `decided`: record the approval in the decision record's `Status:` line, then refine the backlog with the `task-breakdown` skill's "Refining after a spike" steps (the user's approval covers the tracker sync), and set the spike to `done`. Its dependents become startable on the next step.
- If the user chooses to extend the spike, set it back to `open` with the narrowed question in its `Question:` field.

### d. Keep open PRs healthy
Work through open PRs, bottom of each stack first. Keep **at most one agent per branch or stack** at a time, so no one rebases a branch while someone else fixes it. While any work is pending on a PR, its ticket is `pr-open`.

1. **Parent moved.** If a stacked PR's parent got new commits (its head SHA differs from the one recorded for the child), restack the child onto the parent's new head.
2. **Conflicts with base.** If GitHub reports the PR conflicting, or out of date where branch protection requires up to date, rebase it. Don't rebase just because main moved; it re-triggers CI and review for nothing.
3. **CI failing.** Read the failing logs. If it looks flaky or unrelated to the PR (infrastructure, a test the PR doesn't touch failing on main too), rerun the failed jobs once. If the PR caused it, start a feedback agent with the failure as the item. After two failed fix attempts, set the ticket `blocked` and escalate.
4. **Review feedback.** Wait until the review automation's check has finished. Then triage every unresolved review thread and comment you haven't handled yet (section 3), and start one feedback agent per PR with the items to address and the replies to post.
5. **Ready.** Green checks, review automation finished, no unaddressed feedback: set the ticket `pr-ready`. If its base is main and `--auto-merge` is on, enable auto-merge; otherwise list it as ready for human merge in your update.

### e. Start new tickets
Fill free agent slots with tickets that can start. **Spikes first**, since they gate other work; then critical-path tickets.

- **Spikes** can start when they're `open` and their dependencies are done. Start a spike agent with the spike prompt in [references/agent-prompts.md](references/agent-prompts.md) and set the ticket to `in progress`. Spikes never get a PR and nothing stacks on them.
- A ticket can start when it's `open`, not canceled in the tracker, every **spike** it depends on is `done`, and every other dependency is either **merged** or has an **open PR with green checks and no changes requested**, with all unmerged dependencies on a single stack line. Base it on the top unmerged dependency's branch, or on main if all are merged. Don't exceed `--max-stack`.
- If two unmerged dependencies sit on different stacks, wait until one merges.
- Don't run two tickets at once whose `Files:` overlap.

Decide the branch name (`<username>/<project-slug>/<task-branch>`, with the tracker key in `task-branch` when published), set the ticket to `in progress` (and the tracker issue too, unless the integration does), and start the implementer with the prompt in [references/agent-prompts.md](references/agent-prompts.md): a general-purpose agent, in the background, with `isolation: "worktree"` set **on the Agent call**, named like `impl-T-04`. When it reports a PR, set the ticket to `pr-open` and record the branch and PR.

### f. Report and wait
If anything changed, give the user a short update: PRs opened, merged, restacked, feedback addressed, what's waiting on humans, and new escalations. Stay quiet when nothing changed.

With `--once`, exit here. Otherwise, wait about 5–10 minutes before the next pass, using the session's scheduling or monitoring tools (for example a Monitor on a `gh` polling command) rather than sleeping in a loop. Wake early when an agent finishes.

## 3. Triage review feedback

For each unhandled comment or thread, decide one of four things, and record the comment ID as handled:

- **Address** when it points to a real defect: a bug, a missed or weakly tested acceptance criterion, a security or data-loss risk, a broken repo convention, or a clear, concrete request from a human reviewer.
- **Follow-up** when it's valid but outside the ticket's scope (new behavior, broader refactor). Don't build it. Note it in the run log and the final report as a candidate ticket, and have the feedback agent reply in one line that it's tracked as a follow-up.
- **Decline** when an automated comment is a false positive, a matter of taste, or asks for extra structure the ticket doesn't need, since adding it would undo the smallest-change approach. Have the feedback agent reply with one line of reasoning. Only decline automated comments this way.
- **Escalate** when a human reviewer asks for something you'd otherwise decline, or reviewers disagree, or the comment questions the design itself. Don't argue with humans in threads; bring it to the user with the comment link.

Limit automated review to three feedback rounds per PR. If the review automation keeps raising new items after that, escalate instead of looping.

## 4. When an agent reports

- `done`: an implementer's PR is open, or feedback or restack work is pushed. Update the ticket status and run log; check the PR again next pass.
- `decision-proposed` (spike): set the spike to `decision-proposed`, fill in `Decision:` with the record's path, and raise it with the user (step 2c).
- `inconclusive` (spike): also set `decision-proposed`, and raise it as inconclusive with the agent's recommended next step (a narrower spike, or a reversible default to proceed with).
- `already-met`: verify the claim against the code and the criteria. If it holds, tell the user the ticket can be closed as already done; if not, resume the agent with `SendMessage` saying what's missing.
- `needs-split` or `blocked`: set the ticket `blocked` with the reason, record an escalation with the agent's notes and proposed split, and keep going with other work. Don't apply splits yourself.
- No `RESULT` block, or a crash: resume the agent once with `SendMessage`; if that fails too, escalate.

## 5. Stop and finish

Stop starting new work, let running agents finish, and report when:

- every ticket is `pr-merged` or `done` (success), or
- nothing can move without a human (everything left is waiting on review, merges, decisions, or escalations). Say exactly what's waiting on whom; a later run, or the next `--once` pass, resumes from there, or
- main is red after a merge. Start one agent to fix it as an unplanned ticket (logged as such) and hold new tickets until main is green again, or
- the user tells you to stop.

The final report covers: tickets merged out of total, PRs open and what each is waiting for, decisions awaiting approval, follow-up candidates from review, unplanned fixes, and escalations with the decision each needs.

## Rules for the run

- Never push to main, merge anything yourself, or approve PRs. Merging belongs to humans, or to GitHub auto-merge with `--auto-merge`.
- Force-push only with `--force-with-lease`, and only to this run's branches (the branch prefix).
- Never edit acceptance criteria, mark a ticket merged that isn't, or close PRs, branches, or tracker issues this run didn't create or own.
- Never commit RFC, notes, tasks, or run-log files; they stay local.
- Only the user changes the plan: approving splits, follow-up tickets, or design changes.
- Keep your context lean: rely on agents' `RESULT` blocks, GitHub, and the tracker, not on agent transcripts, unless something went wrong.
