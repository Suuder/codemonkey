# codemonkey

A Claude Code skill that runs a code change through a TDD pipeline:

1. **planner** explores the codebase and writes `plan.md`, including a change map and a before/after diagram of the feature's flow. Anything it would otherwise have to guess goes into Open questions, which you answer before review.
2. **plan-reviewer** checks the plan against the real code (one review round) and turns unlisted assumptions into questions for you
3. You approve the plan, using a visual plan page published as a private claude.ai Artifact
4. **test-writer** writes failing tests (red)
5. **implementer** makes them pass without touching the tests (green)
6. The orchestrator verifies the result itself: test files unchanged, new tests pass, no regressions against the baseline

Working files go in `.claude/work/<slug>/` in your repo.

## Install

As a plugin:

```
/plugin marketplace add <owner>/codemonkey
/plugin install codemonkey@codemonkey
```

Or by hand:

```sh
cp -r skills/codemonkey skills/codemonkey-prompt skills/codemonkey-review ~/.claude/skills/
cp agents/*.md ~/.claude/agents/
```

## Usage

```
/codemonkey <describe the bug or feature>
```

Add "no checkpoint" or "fully automatic" to the request to skip the plan approval step, or "no artifact" to keep the plan summary in chat only.

Subagents never guess intended behavior. They write their questions down, and the orchestrator asks you, saves the answers to `user-feedback.md`, and reruns the subagent.

## Prompt

```
/codemonkey-prompt <rough request>
```

Turns a rough request into a problem statement for `/codemonkey`. It explores the code (read-only), resolves ambiguities with you, and writes a `prompt.md` with the goal, background, decisions, scope and verification.

## Review

```
/codemonkey-review [repo paths...] [base ref] [focus]
```

Reviews the changes in each repo (uncommitted work plus commits since the branch left the default branch, or against the given base ref) and returns a numbered list of issues by severity, each with a fix. Read-only. With no paths, it reviews the current directory.
