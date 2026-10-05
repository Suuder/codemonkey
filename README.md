# codemonkey

A Claude Code skill that runs a code change through a TDD pipeline:

1. **planner** explores the codebase and writes `plan.md`
2. **plan-reviewer** checks the plan against the real code (one review round)
3. You approve the plan
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
cp -r skills/codemonkey ~/.claude/skills/
cp agents/*.md ~/.claude/agents/
```

## Usage

```
/codemonkey <describe the bug or feature>
```

Add "no checkpoint" or "fully automatic" to the request to skip the plan approval step.
