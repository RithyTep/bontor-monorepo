# /speckit.clarify - Resolve Clarifications

Identify and resolve `[NEEDS CLARIFICATION]` items in specifications.

## Usage

```
/speckit.clarify [feature-branch]
```

## Workflow

### Step 1: Scan for Clarifications

Search all spec files for `[NEEDS CLARIFICATION: ...]` markers:

```bash
grep -r "NEEDS CLARIFICATION" specs/$FEATURE/
```

### Step 2: List Pending Clarifications

```markdown
# Pending Clarifications

**Feature**: 001-karaoke-mode

## From spec.md

### CLR-001: Audio Transposition
> [NEEDS CLARIFICATION: Should we support actual audio key change, or just adjust pitch tolerance for scoring?]

**Context**: US4 - Adjust Key/Transpose
**Impact**: Affects architecture (audio processing vs scoring only)
**Options**:
1. **Tolerance only** - Simpler, scoring adjusts for pitch differences
2. **Audio transposition** - Complex, requires real-time audio processing
3. **Both** - Maximum flexibility, highest complexity

### CLR-002: Reference Pitch Data
> [NEEDS CLARIFICATION: Do we have reference pitch data for songs, or is scoring purely based on pitch stability?]

**Context**: US3 - Pitch Feedback
**Impact**: Affects scoring algorithm design
**Options**:
1. **No reference** - Score based on pitch stability and duration
2. **Reference from MIDI** - Requires MIDI files per song
3. **Reference from vocals** - Extract from original recording

---

## From plan.md

No clarifications pending.

---

## Resolution Needed: 2 items
```

### Step 3: Prioritize by Impact

Order clarifications by impact:
1. **Scope** - Changes feature boundaries
2. **Security** - Affects security model
3. **UX** - Changes user experience
4. **Technical** - Implementation details

### Step 4: Present Options

For each clarification, present structured options:

```markdown
## Resolving CLR-001: Audio Transposition

### Question
Should we support actual audio key change, or just adjust pitch tolerance?

### Options

| Option | Effort | Risk | UX Impact |
|--------|--------|------|-----------|
| Tolerance only | Low | Low | Users may be confused |
| Audio transposition | High | Medium | Better experience |
| Both | Very High | High | Maximum flexibility |

### Recommendation
**Option 1: Tolerance only** for MVP

Rationale:
- Simpler implementation
- Can add transposition post-MVP
- Tolerance adjustment achieves similar goal

### Decision
[USER INPUT REQUIRED]

Select option: ___
```

### Step 5: Update Specifications

After user provides decision:

1. **Remove the marker** from spec/plan
2. **Document the decision** in place
3. **Add to decision log** if significant

Before:
```markdown
[NEEDS CLARIFICATION: Should we support actual audio key change?]
```

After:
```markdown
**Decision**: Tolerance adjustment only for MVP. Audio transposition deferred to post-MVP roadmap.
```

### Step 6: Verification

Confirm no clarifications remain:

```bash
grep -r "NEEDS CLARIFICATION" specs/$FEATURE/
# Should return empty
```

## Maximum Clarifications Rule

Per Spec Kit methodology:
- **Maximum 3 clarifications** per specification
- If more needed, spec is too ambiguous
- Break into smaller features or gather more requirements

## Clarification Format

Standard marker format:
```
[NEEDS CLARIFICATION: <specific question>]
```

Examples:
- `[NEEDS CLARIFICATION: What authentication providers should we support?]`
- `[NEEDS CLARIFICATION: Is offline mode required for MVP?]`
- `[NEEDS CLARIFICATION: What is the maximum file size for uploads?]`

Bad examples (too vague):
- `[NEEDS CLARIFICATION: How should this work?]`
- `[NEEDS CLARIFICATION: Need more info]`
- `[NEEDS CLARIFICATION: ???]`

## Output

After resolution:

```markdown
# Clarification Resolution Report

**Feature**: 001-karaoke-mode
**Resolved**: 2024-12-02

## Resolved Items

| ID | Question | Decision |
|----|----------|----------|
| CLR-001 | Audio transposition | Tolerance only (MVP) |
| CLR-002 | Reference pitch | Stability-based scoring |

## Updated Files

- specs/001-karaoke-mode/spec.md (2 changes)
- specs/001-karaoke-mode/plan.md (0 changes)

## Remaining Clarifications

None - ready for implementation.
```
