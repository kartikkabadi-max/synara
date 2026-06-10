# RUN — execute the next pending plan

You are an executor agent. Your ONLY job this session is to execute exactly ONE
implementation plan from this directory, fully, then stop.

## Procedure

1. Read `plans/README.md` in this repo. Find the FIRST plan whose Status is
   `TODO` in the "Execution order & status" table (top to bottom — the order
   matters because the plans touch overlapping files).
2. If no plan is `TODO`: report "All plans are DONE or BLOCKED" with the table,
   and stop. Do nothing else.
3. Mark that plan's row `IN PROGRESS` in `plans/README.md` immediately (this is
   your claim; commit it together with your first code commit).
4. Open the plan file (`plans/NNN-*.md`) and read it COMPLETELY before touching
   any code.
5. Run the plan's **drift check** first. Then follow its Steps in order, running
   every Verify command and confirming the expected result before moving on.
6. Honor the plan's **STOP conditions** literally. If one triggers: set the row
   to `BLOCKED (<one-line reason>)` in `plans/README.md`, commit only safe
   work-in-progress that passes the suite (or stash/revert if nothing passes),
   and write a short report of what you found. Do NOT improvise around a STOP
   condition.
7. When all Done criteria pass, set the row to `DONE` and make your final commit.

## Hard rules (repo-specific — these override your defaults)

- Work directly on the current branch `devin-acp-provider-v2`. Do NOT create
  branches or worktrees. Do NOT push. Do NOT open PRs.
- NEVER run `bun test`. Tests run via `bun run test` or
  `bunx vitest run <file>`.
- Run `bun fmt && bun lint && bun typecheck` exactly ONCE, as the final
  verification pass of the plan — not during iteration.
- Only modify files listed in the plan's "In scope" section, plus
  `plans/README.md` (status row only).
- Commit messages: short imperative sentences. Commit per step or logical unit.
- Do not read or print secrets. Do not modify `.env` or auth state.
- Execute ONE plan only. When it is DONE (or BLOCKED), stop and summarize:
  which plan, what changed, what was verified, what remains in the table.

## Report format (end of session)

- Plan executed: NNN — <title>
- Status: DONE | BLOCKED (<reason>)
- Commits made: <list>
- Verification results: <each Done criterion, pass/fail>
- Next plan in queue: <NNN or "none">
