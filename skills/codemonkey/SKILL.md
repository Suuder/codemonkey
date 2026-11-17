---
name: codemonkey
description: Run a plan → review → failing tests → implementation pipeline for a code change, using the planner, plan-reviewer, test-writer and implementer subagents. Use when the user invokes /codemonkey or asks for the full plan/review/TDD workflow.
---
# Codemonkey: TDD pipeline

Problem statement: $ARGUMENTS

If the problem statement is empty, ask the user what they want built or fixed, then stop.

You are the orchestrator. Don't do the planning, reviewing, test writing or implementing yourself. Hand each step to its subagent and check the results between steps. Pass every subagent the absolute path of the work directory, plus the problem statement where relevant.

## Git: no branches, no commits
Neither you nor any subagent changes git state. That means:
- no `git checkout`/`switch` to another branch, and no `git branch` or `git worktree` to create one;
- no `commit`, `stash`, `reset`, `rebase`, `merge`, `pull` or `push`.

Leave every change uncommitted in the working tree of whatever is currently checked out. Read-only git commands are fine (`status`, `diff`, `log`, `show`, `blame`).
- **Wrong checkout:** if the work needs a different branch than the one checked out (for example, the problem statement builds on an unmerged branch), stop before step 1. Tell the user which repos need which branch, and wait for them to switch.
- **Subagents:** include this rule in every subagent prompt.

## Ask, don't assume
A wrong guess costs a full pipeline run. A question costs the user one reply. Whenever you or a subagent would otherwise have to guess something the user cares about, ask instead. That covers intended behavior, scope, edge-case and error handling, user-visible text and naming, data or schema changes, backwards compatibility, and picking between two plausible approaches. When unsure whether something is worth asking, ask.
- **Only you talk to the user.** Subagents can't ask, so they write their questions down: the planner in **Open questions** in plan.md, the reviewer as `Question for user:` findings, and the test writer and implementer under **Questions for user** in their output file. Relay them with `AskUserQuestion` (up to 4 per call). Put the recommended option first, and use `preview` when options differ in code or UI. Append each question and answer to `user-feedback.md`, then rerun the subagent that asked.
- **The code answers facts.** How something works today is not a question for the user. Send the subagent back to read the code.
- **Never fill a gap with an invented value**, such as requirement IDs, dates, names, thresholds or limits. State the gap and ask.
- **Subagents:** include this rule in every subagent prompt.

## Setup
If the problem statement leaves the goal, the scope or the expected behavior unclear, ask the user before going further.
Pick a short kebab-case slug for the problem. Create `<repo root>/.claude/work/<slug>/` (repo root = the current working directory). Write the problem statement to `problem.md` in that directory. Tell the user the path.

## 1. Plan
Run the `planner` subagent with the problem statement and the work directory. Confirm that `plan.md` exists.
Then read its **Open questions** section. If it lists any, ask the user (see "Ask, don't assume"), record the answers in `user-feedback.md`, and run `planner` again. Repeat until no open questions remain, at most 3 rounds. Carry any questions still open after that to the step 3 checkpoint.
Then publish the plan artifact (see "Plan artifact").

## 2. Review (once)
Run the `plan-reviewer` subagent exactly once. Read the last line of `review.md`.
- `VERDICT: APPROVED` → go to step 3.
- `VERDICT: CHANGES_REQUIRED` → first ask the user every `Question for user:` finding and record the answers in `user-feedback.md`. Then run `planner` again (it revises using review.md and user-feedback.md), then go to step 3.
Either way, republish the plan artifact so it shows the review findings and any revision. Do not review the revised plan again. In the step 3 summary, list the BLOCKING items and how the planner's Revision notes say each was handled.

## 3. Approval checkpoint
Give the user the plan artifact link, the absolute path to `plan.md` and a summary: the acceptance criteria, the proposed changes in at most 5 bullets, the planner's **Assumptions** list, any remaining NITs or open questions, and the test command. Ask for approval before writing any code. Wait for their reply. If the user asks for changes, write their feedback to `user-feedback.md` in the work directory (append if it exists), run `planner` again, review it again, republish the plan artifact, and repeat this checkpoint.
(Skip this checkpoint only if the user's request said "no checkpoint" or "fully automatic".)

## Plan artifact
A visual view of the plan, published as a private Artifact. You build it, because the subagents don't have the Artifact tool. Skip it if the user's request said "no artifact" or the Artifact tool isn't available, and give the step 3 summary in chat only.
- **Source:** build the page only from plan.md, review.md and user-feedback.md. Never read in values that aren't there. Leave out secrets, credentials and any clinical trial data.
- **Design:** load the `artifact-design` skill (and `artifact-diagramming` for the change map) before writing the page the first time.
- **File:** write `<workdir>/plan.html` and publish it with the Artifact tool. Every republish uses the same file path, so the URL stays the same.
- **Sections, in this order:**
  1. Header: problem title, slug, status (draft / reviewed / approved), review verdict, and BLOCKING and NIT counts.
  2. Open questions and assumptions: each question with its options and the recommended option highlighted, marked answered (with the answer from user-feedback.md) or open.
  3. Change map: the planner's **Change map** drawn as a diagram, with new and modified files styled differently.
  4. Files: a table of each file, new or modified, its layer, the acceptance criteria it serves, and its plan step. Rows link to the matching diff.
  5. Proposed changes: one collapsible card per step, with the planner's `diff` hunks syntax-highlighted and the "why" beside them.
  6. Before / after flow: the planner's two **Flow: before / after** diagrams side by side (stacked on narrow screens), drawn at the same scale and in the same order so unchanged steps line up. Color added, changed and removed steps consistently, with a legend that works in light and dark mode. Clicking a highlighted step jumps to its card in Proposed changes. Below the diagrams, show the list of differences, plus a UI mockup when the frontend changes.
  7. Acceptance criteria: each criterion with its changes and planned tests (the traceability matrix).
  8. Review findings: BLOCKING and NIT items, each with how the Revision notes handled it.
  9. Test strategy: the test command (copyable), new test files, and criteria verified by review or manual check.
- **Approval stays in chat.** The page is for reading. Approvals and change requests come through the step 3 checkpoint.

## 4. Tests (red)
Run the `test-writer` subagent. Then run the test command from tests.md yourself and confirm:
- The new tests exist and FAIL.
- They fail because the behavior is missing, not because of import or syntax errors.
If any new test passes already, or fails for the wrong reason, send the problem back to `test-writer` (at most 2 rounds), then stop and ask the user.
If tests.md has **Questions for user**, ask them before checking anything else, then rerun `test-writer` with the answers.

## 5. Implement (green)
Before running the implementer, snapshot the tests: run `shasum` on each test file listed in tests.md and keep the output.
Run the `implementer` subagent. If implementation.md has **Questions for user**, ask them and rerun `implementer` with the answers. Then go straight to step 6. Never skip step 6, even if the implementer reports that everything passes.

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
