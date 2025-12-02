# /speckit.specify - Create Feature Specification

Create a new feature specification from a natural language description.

## Usage

```
/speckit.specify <feature description>
```

## Workflow

### Step 1: Generate Branch Name

Create a semantic branch name (2-4 words, action-noun format):
- Format: `NNN-short-name` (e.g., `001-karaoke-mode`, `002-song-cms`)
- Scan existing `specs/` directories to determine next number

### Step 2: Create Specification Directory

```bash
mkdir -p specs/$BRANCH_NAME
```

### Step 3: Generate spec.md

Use the template structure:

```markdown
# Feature Specification: [Feature Name]

**Branch**: `$BRANCH_NAME`
**Created**: [DATE]
**Status**: Ready for Planning

---

## Summary

[1-2 sentence description of the feature]

---

## User Scenarios & Testing

### US1: [Story Name] (P1/P2/P3)

**Context**: [Why this story matters]

**Story**: As a [role], I want [action] so that [benefit].

**Acceptance Scenarios**:

\`\`\`gherkin
Scenario: [Name]
  Given [context]
  When [action]
  Then [expected result]
\`\`\`

**Edge Cases**:
- [Edge case 1]
- [Edge case 2]

---

## Requirements

### Functional Requirements

| ID | Requirement | User Story |
|----|-------------|------------|
| FR1 | [Requirement] | US1 |

### Non-Functional Requirements

| ID | Requirement | Metric |
|----|-------------|--------|
| NFR1 | [Requirement] | [Measurable target] |

### Key Entities

| Entity | Description |
|--------|-------------|
| [Name] | [Description] |

---

## Success Criteria

| Metric | Target | Measurement |
|--------|--------|-------------|
| [Metric] | [Target] | [How measured] |

---

## Requirement Completeness Checklist

- [ ] All user stories have acceptance scenarios
- [ ] Each user story is independently testable
- [ ] Success criteria are measurable
- [ ] Edge cases are documented
- [ ] Non-functional requirements have specific metrics
- [ ] All `[NEEDS CLARIFICATION]` items resolved
```

## Constraints

1. **No implementation details** - No frameworks, APIs, or languages
2. **Technology-agnostic** - Success criteria don't mention tech
3. **Maximum 3 clarifications** - Mark as `[NEEDS CLARIFICATION: question]`
4. **Priority by impact** - Order: scope > security > UX > technical

## Output

After creating the spec:
1. Report the created file path
2. List any `[NEEDS CLARIFICATION]` items
3. Suggest running `/speckit.plan` next

## Example

```
/speckit.specify Add user authentication with social login
```

Creates `specs/004-user-auth/spec.md` with user stories for login, registration, and social OAuth.
