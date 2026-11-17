---
name: codemonkey-prompt
description: Turn a rough user request into a well-grounded problem statement for /codemonkey. Explores the codebase (read-only), resolves ambiguities with the user, and writes a prompt.md with goal, background, decisions, scope and verification. Use when the user invokes /codemonkey-prompt or asks to "write a prompt/brief for codemonkey".
---
# Codemonkey prompt writer

User request: $ARGUMENTS

If the request is empty, ask the user what they want built or fixed, then stop.

Your output is a problem statement that the `/codemonkey` pipeline will consume. Its `planner` subagent will explore the code again and write the actual plan, so your job is not to plan the implementation. Your job is to make sure the planner starts from the right understanding of the problem: what is wanted, why, where it lives, what has already been decided, and how to tell when it is done.

You are read-only. Never edit source or test files. The only file you write is the prompt.

## 1. Understand the request
Restate the request to yourself in one or two sentences. Note the nouns (services, endpoints, tools, tables, lambdas, UI screens) you need to locate in the code.

## 2. Explore the codebase
Find the code the request touches. For each relevant piece, learn how it works today:
- Entry points, the call chain, and data flow across services (a repo may hold several, e.g. ai-be, ai-service, virtual_isf_be, virtual_isf_fe). Follow calls across service boundaries when the request crosses them.
- Other consumers of anything that would change (callers, other lambdas, frontends, tests). These are where regressions hide.
- Existing precedent: a similar feature, pattern or helper the change should follow.
- How the affected modules run their tests.
- Prior work in `.claude/work/*/` that touches the same area (problem.md, plan.md, user-feedback.md). Earlier user feedback there often records preferences that still apply.

Use Grep/Glob/Read directly. For broad sweeps across many directories, use the `Explore` agent and ask it for conclusions with `path:line` references. Don't guess: every file, function and line you cite in the prompt must be one you (or Explore) actually saw.

## 3. Resolve ambiguities
List the decisions the request leaves open that would change what gets built: scope (which services/consumers), behavior on edge cases and errors, backward compatibility, naming visible to users or other services, and anything where the code shows two reasonable options.
- Settle what the code or sensible defaults settle. Record them as assumptions.
- Ask the user about the rest with AskUserQuestion (one round, at most 4 questions, recommended option first). Skip this if nothing is genuinely open.
- If an approach was chosen over clear alternatives, keep the reasons; the planner should not reopen the decision without flagging it.

## 4. Write the prompt
Pick a short kebab-case slug. Write `<repo root>/.claude/work/<slug>/prompt.md` (repo root = current working directory). Use these sections, dropping any that would be empty:

```markdown
# <Imperative title>

## Goal
What should be true after the change, in 2-5 sentences. User-visible behavior first.

## Background
Why this is needed and how the code works today. Cite `path:line` (paths relative to repo root).
Describe the current call chain / data flow that the change touches, and name other consumers
that must keep working.

## Decisions
Decisions already made (by the user or settled from the code), each with a one-line reason.
Alternatives that were rejected and why, as a short table if there are several.

## Requirements
Numbered, concrete, checkable behaviors. Each should map to at least one test.
Include edge cases and error behavior.

## Out of scope
What must not change. "If you believe one of these must change, stop and explain why."

## Verification
Test commands per affected module, which existing tests must keep passing, and any manual
checks (described, not run, if they need environments you don't have).

## Assumptions & open questions
Anything you assumed without confirmation, so the approval checkpoint can catch it.
```

Guidelines:
- Describe behavior and constraints, not the line-by-line implementation. Name files and precedents as pointers, not as a plan. Prescribe an approach only when it was a deliberate decision.
- Be specific: real endpoint paths, class names, field names, statuses, config keys.
- Keep it as short as it can be while being complete. A small change gets a short prompt.
- Do not add "no checkpoint" or "fully automatic" unless the user asked for it.

## 5. Hand off
Tell the user:
- The absolute path of `prompt.md`.
- A 3-5 bullet summary: goal, key decisions, scope, and any assumptions they should check.
- How to run it, ready to copy:
  `/codemonkey Implement the problem described in <absolute path to prompt.md>. Read that file first; it is the full problem statement.`

If the user asks for changes to the prompt, edit `prompt.md` in place and show only what changed.
