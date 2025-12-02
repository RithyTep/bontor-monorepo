# /speckit.analyze - Analyze Specifications

Analyze specifications for consistency, completeness, and compliance.

## Usage

```
/speckit.analyze [target] [options]
```

Targets:
- `all` - Analyze all specs
- `001-karaoke-mode` - Analyze specific feature
- `constitution` - Analyze constitution compliance

Options:
- `--fix` - Suggest fixes for issues
- `--report` - Generate detailed report

## Analysis Checks

### 1. Specification Completeness

For each `spec.md`:

| Check | Criteria |
|-------|----------|
| Summary | Non-empty, 1-3 sentences |
| User Stories | At least 1 story with P1 priority |
| Acceptance Scenarios | Given-When-Then format |
| Requirements | FR and NFR tables present |
| Success Criteria | Measurable metrics defined |
| Checklist | All items checked or explained |

### 2. Plan Completeness

For each `plan.md`:

| Check | Criteria |
|-------|----------|
| Technical Context | All aspects defined |
| Constitution Check | All gates addressed |
| Project Structure | Directories documented |
| File Creation Order | Tests-first ordering |
| Complexity Tracking | Table present |

### 3. Task Validity

For each `tasks.md`:

| Check | Criteria |
|-------|----------|
| Format | All tasks follow `[ID] [P?] [Story?] Description` |
| IDs | Sequential, no duplicates |
| Checkpoints | Present after each phase |
| Dependencies | Valid phase ordering |
| Estimates | Effort table included |

### 4. Cross-Reference Consistency

| Check | Criteria |
|-------|----------|
| Story Coverage | All spec stories in tasks |
| Entity Coverage | All data-model entities implemented |
| Contract Coverage | All contracts have implementations |
| Requirement Traceability | FR/NFR mapped to stories |

### 5. Constitution Compliance

| Check | Criteria |
|-------|----------|
| Gate Checks | All plans have gates |
| Complexity | Deviations documented |
| Article Adherence | No untracked violations |

## Output Format

```markdown
# Specification Analysis Report

**Generated**: 2024-12-02
**Scope**: All specifications

---

## Summary

| Metric | Value |
|--------|-------|
| Features Analyzed | 3 |
| Total Issues | 5 |
| Critical | 1 |
| Warnings | 3 |
| Info | 1 |

---

## Issues by Feature

### 001-karaoke-mode

#### ✓ spec.md - PASS
All checks passed.

#### ⚠ plan.md - WARNING
- Line 45: Constitution Check missing Integration-First gate status

#### ✓ tasks.md - PASS
All checks passed.

### 002-song-cms

#### ❌ spec.md - CRITICAL
- Missing acceptance scenarios for US3
- Success criteria not measurable

#### ⚠ plan.md - WARNING
- File creation order missing test files

---

## Cross-Reference Issues

| Source | Reference | Issue |
|--------|-----------|-------|
| spec.md US4 | tasks.md | No tasks found for US4 |
| data-model.md Artist | contracts/ | No API contract |

---

## Recommendations

1. **Critical**: Add acceptance scenarios to 002-song-cms/spec.md US3
2. **Warning**: Update 001-karaoke-mode/plan.md Integration-First gate
3. **Info**: Consider adding Artist CRUD contract

---

## Constitution Compliance

| Article | Status | Violations |
|---------|--------|------------|
| I. Library-First | ✓ | 0 |
| II. Type Safety | ✓ | 0 |
| III. Test-First | ⚠ | 1 (missing test order) |
| IV. Client Audio | ✓ | 0 |
| V. Simplicity | ✓ | 0 |
```

## Issue Severity

| Level | Description | Action |
|-------|-------------|--------|
| Critical | Blocks implementation | Must fix before /speckit.implement |
| Warning | May cause issues | Should fix before implementation |
| Info | Best practice | Consider fixing |

## Auto-Fix Suggestions

When using `--fix`:

```markdown
## Suggested Fixes

### 002-song-cms/spec.md

**Issue**: Missing acceptance scenarios for US3

**Suggested Addition**:
\`\`\`gherkin
Scenario: Publish a complete song
  Given a song has title, artist, audio, and lyrics
  When I click "Publish"
  Then the song should be visible to users
  And the published timestamp should be recorded
\`\`\`

**Apply fix?** [Y/n]
```

## Integration with Workflow

Run analysis at key points:

1. **After /speckit.specify** - Validate spec completeness
2. **After /speckit.plan** - Check constitution compliance
3. **After /speckit.tasks** - Verify coverage
4. **Before /speckit.implement** - Full analysis
