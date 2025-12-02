# /speckit.checklist - Create or Validate Checklists

Create validation checklists or verify checklist completion.

## Usage

```
/speckit.checklist [action] [feature-branch]
```

Actions:
- `create` - Create new checklist from spec
- `validate` - Check completion status
- `report` - Generate completion report

## Checklist Location

```
specs/$FEATURE/checklists/
├── pre-implementation.md
├── security.md
├── accessibility.md
├── performance.md
└── release.md
```

## Checklist Types

### Pre-Implementation Checklist

```markdown
# Pre-Implementation Checklist

**Feature**: $FEATURE
**Created**: [DATE]

## Specification Review

- [ ] All user stories have acceptance scenarios
- [ ] Success criteria are measurable
- [ ] Edge cases documented
- [ ] No unresolved `[NEEDS CLARIFICATION]` items

## Technical Review

- [ ] Plan passes all Constitution gates
- [ ] Data model covers all entities from spec
- [ ] API contracts defined for all endpoints
- [ ] File creation order follows test-first

## Dependencies

- [ ] All external dependencies identified
- [ ] Package versions specified
- [ ] Environment variables documented
- [ ] Third-party API access confirmed

## Approval

- [ ] Spec reviewed by stakeholder
- [ ] Plan reviewed by tech lead
- [ ] Ready to proceed with implementation
```

### Security Checklist

```markdown
# Security Checklist

**Feature**: $FEATURE

## Authentication

- [ ] All protected routes require auth
- [ ] Role-based access implemented
- [ ] Session timeout configured

## Data Protection

- [ ] Input validation on all endpoints
- [ ] SQL injection prevented (parameterized queries)
- [ ] XSS prevention in place
- [ ] Sensitive data not logged

## File Handling

- [ ] File type validation
- [ ] File size limits enforced
- [ ] Presigned URLs used (no server passthrough)
- [ ] Malware scanning considered

## Secrets

- [ ] No secrets in code
- [ ] Environment variables used
- [ ] .env files in .gitignore
```

### Accessibility Checklist

```markdown
# Accessibility Checklist

**Feature**: $FEATURE

## Keyboard Navigation

- [ ] All interactive elements focusable
- [ ] Focus order logical
- [ ] Focus visible indicator
- [ ] Keyboard shortcuts documented

## Screen Readers

- [ ] ARIA labels on controls
- [ ] Alt text on images
- [ ] Form labels associated
- [ ] Error messages announced

## Visual

- [ ] Color contrast meets WCAG AA
- [ ] Text resizable to 200%
- [ ] No information conveyed by color alone
- [ ] Animations respect reduced-motion
```

### Performance Checklist

```markdown
# Performance Checklist

**Feature**: $FEATURE

## Core Web Vitals

- [ ] LCP ≤ 2.5s
- [ ] FID ≤ 100ms
- [ ] CLS ≤ 0.1

## Bundle Size

- [ ] JS bundle ≤ 200KB gzipped
- [ ] Code splitting implemented
- [ ] Unused dependencies removed

## Data Fetching

- [ ] Loading states present
- [ ] Error states handled
- [ ] Pagination for large lists
- [ ] Caching strategy defined
```

### Release Checklist

```markdown
# Release Checklist

**Feature**: $FEATURE

## Code Quality

- [ ] All tests passing
- [ ] No TypeScript errors
- [ ] Linting passes
- [ ] Code reviewed

## Documentation

- [ ] README updated
- [ ] API docs current
- [ ] Changelog entry added
- [ ] Migration guide (if breaking)

## Deployment

- [ ] Environment variables set
- [ ] Database migrations ready
- [ ] Feature flag configured
- [ ] Rollback plan documented

## Verification

- [ ] Staging deployment tested
- [ ] Quickstart scenarios pass
- [ ] Performance benchmarked
- [ ] Security scan clean
```

## Validation Output

```markdown
# Checklist Validation Report

**Feature**: 001-karaoke-mode
**Validated**: 2024-12-02

## Summary

| Checklist | Total | Complete | Incomplete |
|-----------|-------|----------|------------|
| pre-implementation.md | 12 | 12 | 0 |
| security.md | 10 | 8 | 2 |
| accessibility.md | 8 | 6 | 2 |
| performance.md | 8 | 8 | 0 |

**Overall**: 38/40 (95%)

## Incomplete Items

### security.md
- [ ] Malware scanning considered
- [ ] Rate limiting implemented

### accessibility.md
- [ ] ARIA labels on controls
- [ ] Focus visible indicator

## Recommendation

⚠ Address security items before release.
Consider accessibility improvements for next iteration.
```

## Integration with Workflow

1. **After /speckit.plan** - Create checklists
2. **Before /speckit.implement** - Validate pre-implementation
3. **During implementation** - Update security/accessibility
4. **Before release** - Complete all checklists
