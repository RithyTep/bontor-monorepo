# /speckit.plan - Create Implementation Plan

Generate a technical implementation plan from a feature specification.

## Usage

```
/speckit.plan [feature-branch]
```

If no branch specified, uses the current feature context or prompts for selection.

## Prerequisites

- `specs/$FEATURE/spec.md` must exist
- `memory/constitution.md` must exist

## Workflow

### Phase 0: Research & Context

1. **Load specification** from `specs/$FEATURE/spec.md`
2. **Load constitution** from `memory/constitution.md`
3. **Identify unknowns** - Items marked `[NEEDS CLARIFICATION]`
4. **Research dependencies** - Technology choices, integrations
5. **Generate `research.md`** (if needed)

### Phase 1: Design & Contracts

1. **Extract entities** from spec → generate `data-model.md`
2. **Define API contracts** → generate `contracts/*.md`
3. **Create validation scenarios** → generate `quickstart.md`

### Phase 2: Implementation Plan

Generate `plan.md` with this structure:

```markdown
# Implementation Plan: [Feature Name]

**Branch**: `$BRANCH_NAME`
**Created**: [DATE]
**Spec**: `specs/$FEATURE/spec.md`

---

## Summary

[Technical summary of implementation approach]

---

## Technical Context

| Aspect | Decision |
|--------|----------|
| Language | [e.g., TypeScript 5.x] |
| Framework | [e.g., Next.js 16] |
| Database | [e.g., PostgreSQL] |
| [Other] | [Decision] |

---

## Constitution Check

### Phase -1: Pre-Implementation Gates

#### Simplicity Gate (Article V)
- [ ] Implementation uses existing packages only
- [ ] No new packages beyond approved list
- [ ] Justification for any additions

#### Anti-Abstraction Gate (Article VI)
- [ ] Using framework features directly
- [ ] No unnecessary wrapper abstractions
- [ ] Standard patterns followed

#### Integration-First Gate (Article VII)
- [ ] Contract tests defined
- [ ] Real service integration tested
- [ ] Mocks documented and justified

---

## Project Structure

### Specification Directory
\`\`\`
specs/$FEATURE/
├── spec.md
├── plan.md
├── data-model.md
├── contracts/
├── quickstart.md
└── tasks.md
\`\`\`

### Source Code Structure
\`\`\`
[Package/app structure for implementation]
\`\`\`

---

## Technical Design

[Architecture diagrams, component relationships, data flow]

---

## File Creation Order

[Ordered list with tests-first approach]

---

## Complexity Tracking

| Article | Deviation | Justification | Date |
|---------|-----------|---------------|------|
| - | None | - | - |

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| [Risk] | [Mitigation strategy] |
```

## Constitution Compliance

The plan MUST pass all gates before proceeding:

1. **Simplicity Gate** - Verify package count within limits
2. **Anti-Abstraction Gate** - No unnecessary layers
3. **Integration-First Gate** - Contracts before code

If gates fail, document violations in Complexity Tracking with justification.

## Output Artifacts

| File | Purpose |
|------|---------|
| `plan.md` | Technical implementation plan |
| `data-model.md` | Entity definitions |
| `contracts/*.md` | API contracts |
| `quickstart.md` | Validation scenarios |
| `research.md` | Technical research (if needed) |

## Next Step

After plan is complete, run `/speckit.tasks` to generate task list.
