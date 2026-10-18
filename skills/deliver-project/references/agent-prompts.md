# Agent prompts

Fill in the placeholders. Start every agent as a general-purpose agent in the background, with `isolation: "worktree"` set on the Agent call. Worktrees start from the default branch and contain only committed files, so each prompt says which branch to fetch and where the local files are.

Shared context to append to every prompt:

> You are one agent on a delivery team; others are working on other branches at the same time. Touch only what your task needs. Local planning files (RFC, tasks file) live in the main project folder `{MAIN_FOLDER}`: read them there by absolute path, but never edit anything or run git commands in that folder. Don't update ticket statuses in the tasks file or the tracker; the orchestrator does that from your report. No one can answer questions during this run: when you'd ask, stop and explain in your report. Keep your final message to a few lines, ending with the `RESULT` block described below.

---

## Implementer

> Implement one ticket. Invoke the `implement-task` skill with these arguments and follow it:
>
> `{TRACKER_KEY or LOCAL_ID} --tasks {MAIN_FOLDER}/{TASKS_FILE} --base {BASE_BRANCH} --branch {BRANCH_NAME} --unattended`
>
> The ticket is {LOCAL_ID} in the tasks file{IF PUBLISHED: and {TRACKER_KEY} in {TRACKER}; fetch it from there, it's the source of truth for the ticket's content}.
>
> {IF STACKED: Your base `{BASE_BRANCH}` is the branch of PR {PARENT_PR}, which isn't merged yet. Branch from `origin/{BASE_BRANCH}` and open your PR against it.}

## Spike

> Run one research spike. Invoke the `spike-task` skill with these arguments and follow it:
>
> `{TRACKER_KEY or LOCAL_ID} --tasks {MAIN_FOLDER}/{TASKS_FILE} --branch {SPIKE_BRANCH_NAME} --unattended`
>
> The ticket is {LOCAL_ID} in the tasks file{IF PUBLISHED: and {TRACKER_KEY} in {TRACKER}; fetch it from there}. Writing the decision record into the main project folder is the one exception to the read-only rule. Your report ends with the `spike-task` result block (`RESULT: decision-proposed | inconclusive | blocked`).

## Feedback

> Address review feedback on PR {PR_URL} (ticket {LOCAL_ID} / {TRACKER_KEY}). Invoke the `implement-task` skill with:
>
> `{TRACKER_KEY or LOCAL_ID} --tasks {MAIN_FOLDER}/{TASKS_FILE} --feedback {PR_URL} --unattended`
>
> Items to address:
> {for each: - {COMMENT_URL} — {what to do, in one line}}
>
> Items to answer without changing code (reply in the thread with this text, then move on):
> {for each: - {COMMENT_URL} — "{reply text}"}
>
> {IF CI: Failing check: {CHECK_NAME}. Relevant log lines: {EXCERPT}}

## Restack

> Restack PR branches for this run, following the procedure in {STACKS_MD_ABSOLUTE_PATH}. Read it first.
>
> Stack, bottom to top:
> {for each: - PR {PR_URL}, branch `{BRANCH}`: rebase onto `{NEW_BASE}` (old base commit `{OLD_BASE_SHA}`), then set PR base to `{NEW_BASE_BRANCH}`}
>
> Report back with the `RESULT` block (`done` or `blocked`), adding one line per branch with its new head SHA.

The `RESULT` block (implement-task produces it; restack agents write it themselves):

```
RESULT: done | blocked | needs-split | already-met
TICKET: <local id> / <tracker key or "unpublished">, or "stack"
PR: <url or "none">
BRANCH: <branch or "none">
SKIPPED: <items not addressed, or "none">
NOTES: <why blocked; new head SHAs for restacks; follow-ups>
```
