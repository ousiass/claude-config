---
name: impl-wt-en
description: Implementation cycle in an isolated git worktree environment with PR creation.
user-invocable: true
---

# impl-wt-en

Worktree-isolated version of `impl`. Implements without affecting the main working tree.

## Prerequisites

- Claude Code environment
- `git`, `gh` CLI

## Arguments

- **Issue number** (e.g., `/impl-wt-en #123`): Fetch requirements from GitHub Issue
- **Issue URL** (e.g., `/impl-wt-en https://github.com/owner/repo/issues/123`): Same as above
- **Text** (e.g., `/impl-wt-en Add user authentication`): Use text as requirements
- **No arguments**: Interview the user for requirements

## Phase 1: Requirements Analysis and Scope Splitting

1. Retrieve requirements from arguments
   - Issue: Read body and comments via `gh issue view`
   - Text: Use as-is
   - No arguments: Interview the user
2. Record the current branch as the base branch (PR merge target)
   - Run `git branch --show-current` and display to user: "Base branch: <branch-name>"
   - Retain this name until PR creation in Phase 3
3. Determine the working branch name (see `references/branch-naming.md`)
4. **Create git worktree** (see `references/worktree-setup.md`)
5. Split requirements into independently implementable and testable units
6. Sort by dependencies and determine implementation order
7. Create tasks with TaskCreate

- Explore and understand the codebase to grasp requirements accurately
- Confirm unclear specs with the user
- Confirm backward compatibility with the user if breaking changes exist

## Phase 2: Implementation Cycle (repeat per scope)

**IMPORTANT: Specify the worktree path as working directory in all subagent calls during Phase 2.**

#### 2-1: Plan
- Create an implementation plan with the `Plan` agent
- Clarify change locations, impact scope, and test requirements
- Include worktree path in prompt

#### 2-2: Develop
- Implement (including tests) with the `develop` agent
- Satisfy requirements with minimal changes
- Include worktree path in prompt

#### 2-3: Review
- Code review with the `review` agent
- Evaluate requirement conformance, code quality, and test sufficiency
- Include worktree path in prompt

#### 2-4: Improvement Cycle
- If review issues found -> fix with `develop` -> re-review -> repeat until no issues

#### 2-5: Format & Lint
- Run format/lint on changed files per project settings
- **Run format/lint commands inside the worktree directory**
- Skip if no settings found

#### 2-6: Commit (mandatory)
- **Commit at each scope completion. Never skip.**
- **Run `git add` / `git commit` inside the worktree directory**
- Follow commit message conventions in CLAUDE.md

## Phase 3: Final Verification and PR Creation

1. Confirm all scopes are implemented
2. **Run full test suite inside the worktree directory**
3. **Run `git push -u origin <working-branch>` inside the worktree directory**
4. Create PR with `gh pr create --base <base-branch>`
   - **Use the base branch recorded in Phase 1. Never fall back to `main` or `master`.**
   - If unclear, check fork point with `git log --oneline --graph HEAD...main`
   - With Issue: Include Issue number in title, add `Closes #<number>` at body start
   - PR body: Change summary + manual checklist (see `templates/pr-checklist.md`)
5. Report implementation summary and **worktree path** to user

Report example:
```
## Done
- PR: <URL>
- Worktree: <path> (run `git worktree remove <path>` after verification)
```

## Rules

- Each scope must be independently implementable and testable
- **Commit at each scope completion.** Never proceed without committing
- Fix review issues at all severity levels
- Track progress with TaskCreate/TaskUpdate
- **All git/file operations must be inside the worktree directory. Never modify the main working tree.**
