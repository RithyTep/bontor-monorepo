# /speckit.tasks - Generate Task List

Generate an actionable, dependency-ordered task list from implementation plan.

## Usage

```
/speckit.tasks [feature-branch]
```

## Prerequisites

Required files in `specs/$FEATURE/`:
- `plan.md` (required)
- `spec.md` (required)

Optional supporting files:
- `data-model.md`
- `contracts/*.md`
- `research.md`
- `quickstart.md`

## Task Format

Every task MUST follow this format:

```
- [ ] [ID] [P?] [Story?] Description with file path
```

### Components

| Component | Format | Required | Description |
|-----------|--------|----------|-------------|
| Checkbox | `- [ ]` | Yes | Unchecked markdown checkbox |
| ID | `[T001]` | Yes | Sequential task identifier |
| Parallel | `[P]` | No | Indicates parallelizable task |
| Story | `[US1]` | For story phases | Links to user story |
| Description | Text | Yes | Action with exact file path |

### Valid Examples

```markdown
- [ ] [T001] Initialize project with package.json
- [ ] [T002] [P] [US1] Create user model (`src/models/user.ts`)
- [ ] [T003] [P] [US1] Write user model tests (`tests/user.test.ts`)
- [ ] [T004] [US1] Implement user service (`src/services/user.ts`)
```

### Invalid Examples

```markdown
- [T001] Missing checkbox
- [ ] Missing ID - Create model
- [ ] [T002] [US1] Missing file path
```

## Task Organization

### Phase Structure

```markdown
## Phase 1: Setup

- [ ] [T001] Initialize package
- [ ] [T002] Configure TypeScript
- [ ] [T003] Set up testing framework

**Checkpoint**: Project scaffolded and builds

---

## Phase 2: Foundational (Blocks All User Stories)

- [ ] [T004] [P] Create database schema
- [ ] [T005] [P] Set up authentication
- [ ] [T006] Configure API routing

**Checkpoint**: Core infrastructure ready

---

## Phase 3: User Story 1 - [Name] (P1)

### Goal
[What this story achieves]

### Test Criteria
[How to verify completion]

### Tasks
- [ ] [T007] [P] [US1] Write tests for X (`tests/x.test.ts`)
- [ ] [T008] [US1] Implement X (`src/x.ts`)

**Checkpoint**: US1 complete and testable

---

## Phase 4: User Story 2 - [Name] (P2)

[Continue pattern...]

---

## Phase N: Polish

- [ ] [T0XX] Add loading states
- [ ] [T0XX] Implement error handling
- [ ] [T0XX] Performance optimization

**Checkpoint**: Production-ready
```

## Parallelization Rules

Mark tasks with `[P]` when:
1. They touch **different files**
2. They have **no data dependencies**
3. They can run **simultaneously**

### Parallel Groups

Document which tasks can run together:

```markdown
## Parallel Task Groups

**Group A** (Tests - no dependencies):
- [T002], [T003], [T004]

**Group B** (Models - after schema):
- [T007], [T008], [T009]

**Group C** (Services - after models):
- [T012], [T013]
```

## Dependency Graph

Include a visual dependency flow:

```markdown
## Dependencies

\`\`\`
Phase 1 (Setup)
    │
    ▼
Phase 2 (Foundational)
    │
    ├────────┬────────┐
    ▼        ▼        ▼
Phase 3   Phase 4   Phase 5
(US1)     (US2)     (US3)
    │        │        │
    └────────┴────────┘
             │
             ▼
      Phase N (Polish)
\`\`\`
```

## Effort Estimates

Include estimates per phase:

```markdown
## Estimated Effort

| Phase | Tasks | Parallel? | Estimate |
|-------|-------|-----------|----------|
| Setup | 5 | No | 0.5 days |
| Foundational | 10 | Partial | 2 days |
| US1 | 8 | Partial | 1.5 days |
| US2 | 6 | Partial | 1 day |
| Polish | 8 | Yes | 1 day |

**Total**: ~6 days (with parallelization: ~4 days)
```

## Output Validation

After generating tasks.md, report:

1. **Total task count**
2. **Tasks per user story**
3. **Parallelization opportunities** (% of tasks marked [P])
4. **Estimated total effort**
5. **MVP scope** (which stories are P1)

## Next Step

After tasks are generated, run `/speckit.implement` to execute.
