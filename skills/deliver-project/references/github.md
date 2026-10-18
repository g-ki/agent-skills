# GitHub commands

`gh api` fills in `{owner}` and `{repo}` for the current repository.

## This run's PRs

This run's PRs are the user's PRs whose head branch starts with the run's prefix, `<username>/<project-slug>/`:

```bash
gh pr list --author "@me" --state all --limit 200 \
  --json number,url,state,isDraft,headRefName,headRefOid,baseRefName,mergedAt,mergeable,mergeStateStatus,reviewDecision,statusCheckRollup \
  --jq '[.[] | select(.headRefName | startswith("<username>/<project-slug>/"))]'
```

Map PRs to tickets by the tracker key or local ID in the branch's last segment.

- `state`: `OPEN`, `MERGED`, `CLOSED`.
- `mergeable: CONFLICTING` means it needs a rebase. `UNKNOWN` means GitHub is still computing; check again next pass.
- `mergeStateStatus: BEHIND` means out of date with its base; rebase only if branch protection requires up-to-date branches (`BLOCKED` with an otherwise green PR is the usual sign).
- `baseRefName` other than main means it's stacked.

## Checks and CI

```bash
gh pr checks <number> --json name,state,bucket,link,workflow
gh run view <run-id> --log-failed        # run id is in the check's link
gh run rerun <run-id> --failed           # once, for flaky or unrelated failures
```

The review automation shows up here as a check; its review is finished when that check has completed.

## Review threads (inline comments)

```bash
gh api graphql -F owner='{owner}' -F name='{repo}' -F number=<number> -f query='
query($owner:String!,$name:String!,$number:Int!){
  repository(owner:$owner,name:$name){
    pullRequest(number:$number){
      reviewThreads(first:100){
        nodes{
          id isResolved isOutdated path line
          comments(first:30){ nodes{ databaseId url createdAt body author{ login __typename } } }
        }
      }
    }
  }
}'
```

- Triage unresolved threads only, and only comments whose ID isn't in the run log's handled list.
- Automated comments: author `__typename` is `Bot`, or the login matches the review automation's bot (often ending in `[bot]`).
- Outdated threads (code since changed) still count if unresolved; check whether the latest code already addresses them.

Reply in a thread (use the `databaseId` of the thread's first comment):

```bash
gh api -X POST repos/{owner}/{repo}/pulls/<number>/comments/<comment-id>/replies -f body='Fixed in abc1234: <what changed>'
```

## Top-level comments and reviews

```bash
gh pr view <number> --json comments,reviews,latestReviews
gh pr comment <number> --body '<text>'
```

A review with state `CHANGES_REQUESTED` from a human blocks merge until that reviewer re-reviews; summarize it to the user once the feedback is addressed.

## Changing PRs

```bash
gh pr edit <number> --base main          # after the parent merged
gh pr merge <number> --auto --squash     # only with --auto-merge; use the repo's usual merge method
```
