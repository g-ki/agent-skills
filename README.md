# agent-skills

A set of [Agent Skills](https://agentskills.io) that take a hard engineering problem from idea to merged PRs.

| Skill | What it does |
| --- | --- |
| [`productive-thinking`](skills/productive-thinking) | Facilitates Tim Hurson's 6-step Productive Thinking Model, grounded in your codebase, and turns the session into a Technical RFC. |
| [`task-breakdown`](skills/task-breakdown) | Splits an RFC or design doc into small, independent 1-point tickets with acceptance criteria and dependencies. |
| [`implement-task`](skills/implement-task) | Implements one ticket test-first as the smallest complete change and opens a PR. |
| [`spike-task`](skills/spike-task) | Runs a time-boxed research spike and produces a decision record. |
| [`deliver-project`](skills/deliver-project) | Orchestrates parallel workers over a whole backlog and shepherds every PR to merge. Manual-only. |

Typical flow: `productive-thinking` → `task-breakdown` → `deliver-project` (which runs `implement-task` / `spike-task`).

## Install

Any agent supported by the [`skills`](https://github.com/vercel-labs/skills) CLI:

```sh
npx skills add g-ki/agent-skills
```

Install a single skill:

```sh
npx skills add g-ki/agent-skills --skill task-breakdown
```

### Manual (Claude Code)

```sh
git clone https://github.com/g-ki/agent-skills.git
cp -R agent-skills/skills/* ~/.claude/skills/        # all projects
# or: cp -R agent-skills/skills/* .claude/skills/    # current project only
```

## License

MIT
