---
name: spec-gen-en
description: Interactively create design documents for new projects. Also supports appending to existing specs.
user-invocable: true
---

# spec-gen-en

Actively discuss with the user to resolve unknowns, presenting reference ideas to solidify specifications.

## Prerequisites

- Claude Code environment
- `gh` CLI (for GitHub Issue mode)

## Arguments

- No arguments: Create a full set of design documents for a new project
- Path specified: Append/update to an existing specification document

## Documents to Generate

| Document | File | Content |
|----------|------|---------|
| **Functional requirements** | `docs/requirements/functional.md` | Use cases, feature list, screen/operation flows |
| **Non-functional requirements** | `docs/requirements/non-functional.md` | Performance, security, availability, scalability |
| **Architecture design** | `docs/architecture/overview.md` | Overall structure, tech stack, layer architecture, communication flows |
| **ER / Data model** | `docs/architecture/er.md` | Overall ER diagram, per-table ER diagrams, table definitions, constraints, indexes |
| **API specification** | `docs/api/endpoints.md` | Endpoints, request/response, auth, errors |
| **Component design** | `docs/components/overview.md` | Component breakdown, responsibilities, dependencies, interfaces |

- Follow project conventions if they exist. Match existing spec structure when specs already exist

## Phase 1: Assess Current State and Decide Direction

#### Output Destination

Confirm with `AskUserQuestion`:
- **GitHub Issue + branch** (recommended): Create feat Issue and work on a branch. Optimal for linking with impl skill
- **Local only**: Work on current branch without creating Issues or branches

#### For New Projects
1. Interview about overview, purpose, and background
2. Present similar projects and tech stack direction
3. Agree on document scope
4. **GitHub Issue mode**: Record base branch -> create feat Issue -> create `feat/#<issue-number>` branch -> add branch name to Issue
5. Create tasks with TaskCreate

#### For Appending to Existing Specs
1. Read existing specs and understand current specifications
2. Interview about additions
3. Present existing feature reuse/change proposals
4. Agree on direction
5. **GitHub Issue mode**: Same as above
6. Create tasks with TaskCreate

## Phase 2: Document Creation Cycle (repeat per document)

#### 2-1: Interview
- Gather information needed for the document in **a single `AskUserQuestion` call**
- Include pros/cons and reference ideas in option `description` fields

#### 2-2: Write
- Create based on interview results. Use Mermaid for diagrams (see `references/mermaid-guide.md`)
- Include revision history (see format at end of `references/mermaid-guide.md`)

#### 2-3: Review
- Present to user and confirm with `AskUserQuestion` (include "No issues" option)

#### 2-4: Improvement Cycle
- If feedback -> revise -> update revision history -> re-confirm -> repeat until satisfied

#### 2-5: Format & Lint
- Run if settings exist. Skip otherwise

#### 2-6: Commit
- Commit following CLAUDE.md conventions

## Phase 3: Finalize and Push

1. Confirm all documents are complete
2. Cross-document consistency check (ER vs API, components vs functional requirements, etc.)
3. **GitHub Issue mode**: Push -> add spec links to Issue -> report summary (including Issue URL)
4. **Local mode**: Report summary (file path list)

## Rules

- Never guess unknowns; always confirm with user
- **Always use `AskUserQuestion` with options for questions.** Never ask via text alone
- Max 4 questions per round
- Include reference ideas in each option's `description`
- When appending to existing specs, prioritize reuse/modification of existing features
- Always add revision history when updating documents
- Track progress with TaskCreate/TaskUpdate
