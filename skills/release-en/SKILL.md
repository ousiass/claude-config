---
name: release-en
description: Generate changelog, version tag, and GitHub release notes.
user-invocable: true
---

**Language: Always interact with the user in 日本語.**

# release-en

## Prerequisites

- Claude Code environment
- `git`, `gh` CLI

## Arguments

- **Version** (e.g., `/release-en v1.2.0`): Release with specified version
- **Version type** (e.g., `/release-en patch`): Auto-determine from current version
- **No arguments**: Suggest version type based on changes and confirm with user

## Phase 0: Release Options

Confirm with `AskUserQuestion`:

1. **Branch merge**: Merge a branch before release? (confirm source and target)
2. **Binary build**: Attach build artifacts? (confirm build command and artifact path)

## Phase 1: Merge (if applicable)

1. Check if source branch is up to date (`git fetch && git log`)
2. Checkout target -> `git merge --no-ff <source>`
3. If conflicts exist, report to user and abort

## Phase 2: Collect Changes

1. Get latest release tag (`gh release list` / `git tag`)
2. Collect commits and merged PRs since last release
3. Classify by change category (see `templates/changelog.md`)

## Phase 3: Version Decision

1. If version specified in arguments -> use as-is
2. If not specified -> suggest based on changes:
   - Breaking changes -> **major**
   - New features -> **minor**
   - Bug fixes/improvements only -> **patch**
3. Confirm with user for agreement

## Phase 4: Changelog Generation

1. Check if `CHANGELOG.md` exists (create new if not, prepend if exists)
2. Add entries in the format from `templates/changelog.md`
3. Each entry in `English / Japanese` bilingual format. Generate from commit messages

## Phase 5: Build (if applicable)

1. Run build command
2. Verify artifacts exist
3. If failed, report to user and abort

## Phase 6: Release Creation

1. Commit changelog (`docs: add changelog for v<version>`)
2. Create tag (`git tag v<version>`)
3. Push (`git push && git push --tags`)
4. Create release with `gh release create v<version>` (attach artifacts if any)
5. Report release URL to user

## Rules

- Always get user agreement before finalizing version
- Changelog must be fact-based. Use commit messages and PR titles as source
- Always mark breaking changes in the warning section
- Confirm with user before pushing to remote and creating release
- Track progress with TaskCreate/TaskUpdate
