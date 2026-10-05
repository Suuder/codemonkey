---
name: codemonkey
description: Run a plan → review → failing tests → implementation pipeline for a code change, using the planner, plan-reviewer, test-writer and implementer subagents. Use when the user invokes /codemonkey or asks for the full plan/review/TDD workflow.
---
# Codemonkey: TDD pipeline

Problem statement: $ARGUMENTS

If the problem statement is empty, ask the user what they want built or fixed, then stop.

You are the orchestrator. Don't do the planning, reviewing, test writing or implementing yourself. Hand each step to its subagent and check the results between steps. Pass every subagent the absolute path of the work directory, plus the problem statement where relevant.

## Setup
Pick a short kebab-case slug for the problem. Create `<repo root>/.claude/work/<slug>/` (repo root = the current working directory). Write the problem statement to `problem.md` in that directory. Tell the user the path.

## 1. Plan
Run the `planner` subagent with the problem statement and the work directory. Confirm that `plan.md` exists.

## 2. Review (once)
Run the `plan-reviewer` subagent exactly once. Read the last line of `review.md`.
- `VERDICT: APPROVED` → go to step 3.
- `VERDICT: CHANGES_REQUIRED` → run `planner` again (it revises using review.md), then go to step 3. Do not review the revised plan again. In the step 3 summary, list the BLOCKING items and how the planner's Revision notes say each was handled.

## 3. Approval checkpoint
Give the user the absolute path to `plan.md` and a summary: the acceptance criteria, the proposed changes in at most 5 bullets, any remaining NITs, and the test command. Ask for approval before writing any code. Wait for their reply. If the user asks for changes, write their feedback to `user-feedback.md` in the work directory (append if it exists), run `planner` again, review it again, and repeat this checkpoint.
(Skip this checkpoint only if the user's request said "no checkpoint" or "fully automatic".)

## 4. Tests (red)
Run the `test-writer` subagent. Then run the test command from tests.md yourself and confirm:
- The new tests exist and FAIL.
- They fail because the behavior is missing, not because of import or syntax errors.
If any new test passes already, or fails for the wrong reason, send the problem back to `test-writer` (at most 2 rounds), then stop and ask the user.

## 5. Implement (green)
Before running the implementer, snapshot the tests: run `shasum` on each test file listed in tests.md and keep the output.
Run the `implementer` subagent, then go straight to step 6. Never skip step 6, even if the implementer reports that everything passes.

## 6. Verify (required, loop with step 5 at most 3 rounds)
The tests written in step 4 are the definition of done. Verify the implementation against them yourself:
1. Check that the test files are unchanged (compare shasums). If any changed, it is a failure: tell the implementer to restore them.
2. Run the new tests with the command from tests.md. Every one must pass.
3. Run the wider test suite. Compare it against the baseline in tests.md. Any test that passed at baseline and now fails is a regression.
If anything fails, run `implementer` again with the exact failure output, then repeat this step. After 3 rounds, stop and report to the user with the failing output.

## 7. Report
Tell the user:
- The work directory path (plan.md, review.md, tests.md, implementation.md)
- Files changed
- Verification results from step 6, run by you (not the subagents' claims): new tests red → green, and the wider suite compared to baseline
- Deviations from the plan and any open concerns

## Later adjustments
If the user asks for adjustments after the report, don't rerun the whole pipeline. Skip the planner, the approval checkpoint and the test writer:
1. Append the request to `user-feedback.md` in the same work directory.
2. Run `plan-reviewer` once. Tell it to review the adjustment in `user-feedback.md` against plan.md and the current code. If it flags BLOCKING items, show them to the user and ask how to proceed.
3. Run `implementer` with the adjustment. Tell it to read `user-feedback.md` along with plan.md.
4. Run step 6 (verify) as usual, then report the changes briefly.
If the adjustment changes behavior that the existing tests check, those tests will fail. Show the user the conflicting tests and ask whether to run `test-writer` to update them.
