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

Pick one method per agent. Installing both gives you every skill twice.

<details open>
<summary><strong>Claude Code</strong> (plugin, updates itself)</summary>

```sh
claude plugin marketplace add g-ki/agent-skills
claude plugin install agent-skills@g-ki
```

</details>

<details>
<summary><strong>Any other agent, or editable files</strong> (manual updates)</summary>

```sh
npx skills@latest add g-ki/agent-skills -a <agent>   # cursor, opencode, windsurf, amp, ...; omit -a to choose
npx skills@latest add g-ki/agent-skills --skill task-breakdown   # a single skill
```

Update with `npx skills@latest update`.

</details>

<details>
<summary><strong>Manual copy</strong></summary>

```sh
git clone https://github.com/g-ki/agent-skills.git
cp -R agent-skills/skills/* ~/.claude/skills/        # all projects
# or: cp -R agent-skills/skills/* .claude/skills/    # current project only
```

</details>

## Design

- Small, composable skills: each does one step and hands off through a file (RFC, tasks file, decision record), so you can use any of them alone.
- Templates and references live next to the skill that uses them.
- Skills with side effects (`deliver-project`, `productive-thinking`) are manual-only via `disable-model-invocation`.

## License

MIT
