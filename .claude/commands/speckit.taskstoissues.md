# /speckit.taskstoissues - Convert Tasks to GitHub Issues

Convert tasks.md into GitHub issues for project management.

## Usage

```
/speckit.taskstoissues [feature-branch] [options]
```

Options:
- `--dry-run` - Preview without creating issues
- `--milestone` - Assign to milestone
- `--labels` - Add labels (comma-separated)
- `--assignee` - Assign to user

## Prerequisites

- GitHub CLI (`gh`) installed and authenticated
- Repository has GitHub remote configured
- `specs/$FEATURE/tasks.md` exists

## Workflow

### Step 1: Parse Tasks

Read `tasks.md` and extract:
- Phase names
- Task IDs, descriptions, story associations
- Parallel markers
- Checkpoints

### Step 2: Create Issue Structure

Map tasks to issues:

| Task Element | Issue Field |
|--------------|-------------|
| Phase | Milestone or Label |
| Task ID | Issue title prefix |
| Description | Issue title |
| Story [US1] | Label |
| [P] marker | Label: `parallelizable` |
| Checkpoint | Issue with checklist |

### Step 3: Generate Issues

For each task:

```markdown
## Issue Template

**Title**: [T001] Initialize project with package.json

**Labels**:
- `feature: 001-karaoke-mode`
- `phase: setup`
- `story: US1` (if applicable)
- `parallelizable` (if [P] marker)

**Body**:
## Task
Initialize project with package.json

## Feature
001-karaoke-mode

## Phase
Phase 1: Setup

## Acceptance
- [ ] package.json created
- [ ] Dependencies defined
- [ ] Scripts configured

## Related
- Spec: specs/001-karaoke-mode/spec.md
- Plan: specs/001-karaoke-mode/plan.md
```

### Step 4: Create Checkpoint Issues

For each phase checkpoint:

```markdown
**Title**: [CHECKPOINT] Phase 1: Setup Complete

**Labels**:
- `feature: 001-karaoke-mode`
- `checkpoint`

**Body**:
## Checkpoint
Phase 1: Setup

## Verification
- [ ] All Phase 1 tasks complete
- [ ] Project builds successfully
- [ ] Tests pass

## Blocking Tasks
- #1 [T001] Initialize project
- #2 [T002] Configure TypeScript
- #3 [T003] Set up testing framework

## Next Phase
Phase 2: Foundational
```

### Step 5: Link Dependencies

Create issue dependencies where supported:

```markdown
## Dependencies

**Blocks**: #10, #11, #12
**Blocked by**: #5, #6
```

## Dry Run Output

```markdown
# GitHub Issues Preview

**Feature**: 001-karaoke-mode
**Total Issues**: 45

## Phase 1: Setup (5 issues)

| # | Title | Labels |
|---|-------|--------|
| 1 | [T001] Initialize project | setup |
| 2 | [T002] Configure TypeScript | setup |
| 3 | [T003] Set up testing | setup |
| 4 | [T004] Add dependencies | setup |
| 5 | [CHECKPOINT] Phase 1 Complete | checkpoint |

## Phase 2: Foundational (12 issues)

| # | Title | Labels |
|---|-------|--------|
| 6 | [T005] Write pitch detector tests | foundational, parallelizable |
| 7 | [T006] Implement pitch detector | foundational |
| ... | ... | ... |

## Summary

- Setup: 5 issues
- Foundational: 12 issues
- US1: 10 issues
- US2: 8 issues
- US3: 6 issues
- Polish: 4 issues

**Create these issues?** [Y/n]
```

## GitHub CLI Commands

Generated commands for manual execution:

```bash
# Create milestone
gh api repos/:owner/:repo/milestones \
  -f title="001-karaoke-mode" \
  -f description="Karaoke mode feature implementation"

# Create labels
gh label create "feature: 001-karaoke-mode" --color "#0366d6"
gh label create "phase: setup" --color "#fbca04"
gh label create "story: US1" --color "#d73a4a"
gh label create "parallelizable" --color "#0e8a16"
gh label create "checkpoint" --color "#5319e7"

# Create issues
gh issue create \
  --title "[T001] Initialize project with package.json" \
  --body "..." \
  --label "feature: 001-karaoke-mode,phase: setup"

gh issue create \
  --title "[T002] Configure TypeScript" \
  --body "..." \
  --label "feature: 001-karaoke-mode,phase: setup"
```

## Project Board Integration

Optionally create GitHub Project board:

```bash
# Create project
gh project create \
  --title "001-karaoke-mode" \
  --owner @me

# Add columns
# - Backlog
# - In Progress
# - Review
# - Done

# Add issues to project
gh project item-add ... --owner @me --url <issue-url>
```

## Issue Templates

Create reusable templates in `.github/ISSUE_TEMPLATE/`:

```yaml
# .github/ISSUE_TEMPLATE/speckit-task.yml
name: Spec Kit Task
description: Task generated from specs
labels: ["speckit"]
body:
  - type: input
    id: task-id
    attributes:
      label: Task ID
      placeholder: T001
  - type: textarea
    id: description
    attributes:
      label: Description
  - type: input
    id: feature
    attributes:
      label: Feature Branch
  - type: checkboxes
    id: acceptance
    attributes:
      label: Acceptance Criteria
      options:
        - label: Implementation complete
        - label: Tests passing
        - label: Reviewed
```

## Sync Back to Tasks

After issues created, update `tasks.md` with issue links:

```markdown
- [ ] [T001] Initialize project (#1)
- [ ] [T002] Configure TypeScript (#2)
- [x] [T003] Set up testing (#3) ✓
```
