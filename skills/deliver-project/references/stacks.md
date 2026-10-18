# Stacked PRs: restacking procedure

A **stacked** PR targets another ticket's branch instead of main, because its ticket depends on work that isn't merged yet. Its diff on GitHub shows only its own changes. When the parent changes or merges, the child has to move with it.

## The one rule that makes this safe

Always rebase with an explicit old base:

```bash
git rebase --onto <new-base> <old-base-sha> <branch>
```

`<old-base-sha>` is the parent commit the child currently sits on. This replays **only the child's own commits**, which matters when the parent was squash-merged: the parent's original commits are not on main, so a plain `git rebase origin/main` would try to replay them too and conflict.

Where to get `<old-base-sha>`:
- The run log records it for each stacked branch.
- Otherwise, the parent's last head SHA. For a merged parent, that's `gh pr view <parent> --json headRefOid`. This is correct as long as the child was restacked every time the parent moved, which the orchestrator ensures.
- Sanity check: `git merge-base <branch> <old-base-sha>` must equal `<old-base-sha>`. If not, stop and report `blocked`.

## Cases

| Situation | Command | Then |
| :--- | :--- | :--- |
| Parent merged into main | `git rebase --onto origin/main <parent-last-head> <child>` | `gh pr edit <child> --base main` |
| Parent got new commits (still open) | `git rebase --onto origin/<parent-branch> <parent-old-head> <child>` | base unchanged |
| PR on main conflicts or must be up to date | `git rebase origin/main <branch>` | base unchanged |

For a stack of several branches, process bottom to top: after restacking one branch, its child's old base is that branch's **previous** head (record it before rebasing), and its new base is the branch's new head. If all branches of the stack are checked out locally, `git rebase --update-refs` can move the whole stack in one command; use it only if you're sure it covers every branch.

If the team uses a stacking tool (Graphite `gt`, git-town, spr), use its restack/sync commands instead of raw git, so its metadata stays correct.

## Per branch

1. `git fetch origin` and check out the branch from `origin/<branch>`.
2. Rebase as above.
3. **Conflicts:** resolve only mechanical ones yourself: lockfiles (regenerate with the package manager) and import ordering. For anything involving logic, run `git rebase --abort` and report `blocked`, naming the files and commits.
4. Run the tests that cover the branch's changes; they must pass.
5. `git push --force-with-lease origin <branch>`. Never plain `--force`, and never push to main.
6. Update the PR base if the table says so.
7. Report each branch's new head SHA. The orchestrator records it as the old base for that branch's children.

Only touch this run's branches, the ones starting with `<username>/<project-slug>/`.
