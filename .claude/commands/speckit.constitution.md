# /speckit.constitution - Create or Update Constitution

Create or update the project constitution with governing principles.

## Usage

```
/speckit.constitution [action]
```

Actions:
- `create` - Create new constitution from template
- `update` - Update existing constitution
- `validate` - Check all artifacts for compliance

## Constitution Location

```
memory/constitution.md
```

## Template Structure

```markdown
# [PROJECT_NAME] Project Constitution

> Governing principles and development guidelines for [PROJECT_DESCRIPTION].

**Ratification Date**: [RATIFICATION_DATE]
**Last Amended**: [LAST_AMENDED_DATE]
**Version**: [VERSION]
**Status**: Active

---

## Preamble

[Brief statement of purpose and scope]

---

## Article I: [PRINCIPLE_NAME]

**[Principle statement in imperative form]**

1.1. [Specific rule or constraint]
1.2. [Specific rule or constraint]
1.3. [Specific rule or constraint]

---

## Article II: [PRINCIPLE_NAME]

[Continue pattern for each article...]

---

## Amendments

Amendments to this constitution require:

1. Documented proposal with rationale
2. Impact analysis on existing specifications
3. Update to all affected specification documents
4. Version increment and dated changelog entry

---

## Complexity Tracking

Any deviation from Articles [X] or [Y] MUST be documented here:

| Article | Deviation | Justification | Date |
|---------|-----------|---------------|------|
| - | None | - | - |

---

## Related Documents

- `/specs/` - Feature specifications
- `/memory/research.md` - Technical research
- `/README.md` - Project overview
```

## Version Control

Use semantic versioning:

| Change Type | Version Bump | Example |
|-------------|--------------|---------|
| Remove/redefine principle | MAJOR | 1.0.0 → 2.0.0 |
| Add new principle/section | MINOR | 1.0.0 → 1.1.0 |
| Clarify wording | PATCH | 1.0.0 → 1.0.1 |

## Standard Articles

Common principles to consider:

### Technical Principles

| Article | Name | Purpose |
|---------|------|---------|
| Library-First | Code reuse | Features as packages |
| Type Safety | Quality | Strict TypeScript |
| Test-First | Quality | Tests before code |
| Simplicity | Maintainability | Limit complexity |
| Anti-Abstraction | Clarity | Avoid unnecessary layers |
| Integration-First | Reliability | Test with real services |

### Process Principles

| Article | Name | Purpose |
|---------|------|---------|
| Progressive Enhancement | Accessibility | Core works without JS |
| Security | Protection | Follow OWASP guidelines |
| Performance | UX | Meet Core Web Vitals |
| Documentation | Knowledge | Specs before code |

## Validation Workflow

When running `/speckit.constitution validate`:

1. **Load constitution** from `memory/constitution.md`
2. **Scan all specs** in `specs/*/`
3. **Check each plan.md** for Constitution Check section
4. **Report violations**:

```markdown
## Constitution Compliance Report

### Passing
- specs/001-karaoke-mode/plan.md ✓
- specs/002-song-cms/plan.md ✓

### Violations
- specs/003-admin-panel/plan.md
  - Article V: Missing Simplicity Gate check
  - Article VII: Integration tests not defined

### Recommendations
1. Update 003-admin-panel/plan.md with gate checks
2. Add integration test definitions
```

## Propagation

After updating constitution, update these files:

1. **Plan template** - Constitution checks alignment
2. **Spec template** - Scope/requirements alignment
3. **Tasks template** - Task categorization
4. **README** - Principle summary

## Sync Report

After updates, prepend HTML comment with sync report:

```html
<!--
SYNC REPORT
Version: 1.1.0 → 1.2.0 (MINOR)
Changed: Added Article XII (Accessibility)
Updated Templates: plan-template.md, spec-template.md
Flagged Specs: None require updates
Generated: 2024-12-02
-->
```

## Example Constitution

See `memory/constitution.md` for the Karaoke Cambodia project constitution with 11 articles covering:

- Library-First Principle
- Type Safety Imperative
- Test-First Imperative
- Client-Side Audio Processing
- Simplicity Constraints
- Anti-Abstraction Principles
- Integration-First Testing
- Progressive Enhancement
- Khmer Language Support
- Security Requirements
- Performance Budgets
