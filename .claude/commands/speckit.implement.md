# /speckit.implement - Execute Implementation

Execute all tasks to build the feature according to the plan.

## Usage

```
/speckit.implement [feature-branch]
```

## Prerequisites

Required files in `specs/$FEATURE/`:
- `tasks.md` (required)
- `plan.md` (required)

Optional supporting files:
- `data-model.md`
- `contracts/*.md`
- `research.md`
- `quickstart.md`
- `checklists/*.md`

## Workflow

### Step 1: Checklist Verification

If `checklists/` directory exists:
1. Scan all checklist files
2. Count completed vs incomplete items
3. Display status table
4. Pause if incomplete items exist
5. Proceed only with user confirmation

```markdown
| Checklist | Total | Done | Incomplete |
|-----------|-------|------|------------|
| pre-impl.md | 10 | 8 | 2 |
| security.md | 5 | 5 | 0 |
```

### Step 2: Load Context

Read all available documentation:
- `tasks.md` - Task list with dependencies
- `plan.md` - Technical decisions
- `data-model.md` - Entity definitions
- `contracts/*.md` - API specifications
- `research.md` - Technical research
- `quickstart.md` - Validation scenarios

### Step 3: Project Setup

1. **Verify git repository**
   ```bash
   git status || git init
   ```

2. **Create/update .gitignore**
   - Apply technology-specific patterns
   - Include node_modules, .env, build outputs, etc.

3. **Generate tool-specific ignore files**
   - `.dockerignore`
   - `.eslintignore`
   - `.prettierignore`

### Step 4: Parse Task Structure

Extract from `tasks.md`:
- Phases and their tasks
- Dependencies between phases
- Parallel markers `[P]`
- User story associations `[US1]`

Build execution graph:
```
Phase 1 → Phase 2 → [Phase 3, Phase 4, Phase 5] → Phase N
                    (parallel)
```

### Step 5: Execute Tasks

For each phase:

1. **Display phase header**
   ```
   ## Executing Phase 2: Foundational
   ```

2. **Identify parallel groups**
   - Tasks marked `[P]` with same dependencies

3. **Execute with TDD priority**
   - Tests first (if following test-first)
   - Implementation second
   - Integration third

4. **Mark completion**
   ```markdown
   - [x] [T004] [P] Create database schema (`prisma/schema.prisma`)
   ```

5. **Verify checkpoint**
   - Run tests for completed tasks
   - Confirm functionality works

### Step 6: Progress Tracking

Update `tasks.md` in real-time:
- Mark completed tasks with `[x]`
- Add completion timestamps (optional)
- Note any blockers or changes

```markdown
- [x] [T001] Initialize project ✓ 2024-12-02
- [x] [T002] Configure TypeScript ✓ 2024-12-02
- [ ] [T003] Set up testing framework ← Current
- [ ] [T004] Create database schema
```

### Step 7: Validation

After all tasks complete:

1. **Run full test suite**
   ```bash
   pnpm test
   ```

2. **Verify against quickstart.md**
   - Execute validation scenarios
   - Confirm acceptance criteria met

3. **Check spec compliance**
   - All user stories implemented
   - Success criteria achievable
   - Edge cases handled

### Step 8: Completion Report

Generate summary:

```markdown
## Implementation Complete

**Feature**: 001-karaoke-mode
**Duration**: 3 days
**Tasks**: 42/42 completed

### Phase Summary
| Phase | Tasks | Status |
|-------|-------|--------|
| Setup | 5 | ✓ Complete |
| Foundational | 10 | ✓ Complete |
| US1 | 12 | ✓ Complete |
| US2 | 8 | ✓ Complete |
| Polish | 7 | ✓ Complete |

### Test Results
- Unit tests: 45 passed
- Integration tests: 12 passed
- E2E tests: 8 passed

### Files Created
- 23 source files
- 15 test files
- 3 configuration files

### Next Steps
1. Run `/speckit.checklist` for final review
2. Create PR with `/speckit.taskstoissues`
```

## Error Handling

If a task fails:
1. **Log the error** with context
2. **Check dependencies** - was something missed?
3. **Offer options**:
   - Retry the task
   - Skip and continue
   - Pause for manual intervention
4. **Update tasks.md** with failure note

```markdown
- [ ] [T015] [US2] Implement search ❌ BLOCKED: Missing index
```

## Interruption Recovery

If implementation is interrupted:
1. Read `tasks.md` for current state
2. Find last completed task
3. Resume from next uncompleted task
4. Verify previous work still valid

## Best Practices

1. **Commit frequently** - After each phase or significant task
2. **Run tests continuously** - Don't let failures accumulate
3. **Update docs** - Keep plan.md current if decisions change
4. **Communicate blockers** - Flag issues early
