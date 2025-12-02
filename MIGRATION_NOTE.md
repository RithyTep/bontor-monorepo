# Migration to Spec Kit Structure

**Date**: 2024-12-02

## Summary

The project has been restructured to follow the [GitHub Spec Kit](https://github.com/github/spec-kit) methodology.

## Old Structure (To Be Removed)

The following folders contain the legacy documentation format and should be deleted:

```
decisions/          # Replaced by specs/*/plan.md
features/           # Replaced by specs/*/spec.md
implementation/     # Replaced by specs/*/plan.md + tasks.md
proposals/          # Merged into specs/*/plan.md
```

### Commands to Remove Old Folders

```bash
# Remove legacy documentation folders
rm -rf decisions/
rm -rf features/
rm -rf implementation/
rm -rf proposals/
```

## New Structure

```
memory/
└── constitution.md     # Project governance

specs/
├── 001-karaoke-mode/
│   ├── spec.md         # User stories & requirements
│   ├── plan.md         # Technical implementation
│   ├── data-model.md   # Data structures
│   ├── contracts/      # API contracts
│   ├── quickstart.md   # Validation scenarios
│   └── tasks.md        # Implementation tasks
├── 002-song-cms/
│   ├── spec.md
│   ├── plan.md
│   └── tasks.md
└── 003-admin-panel/
    ├── spec.md
    ├── plan.md
    └── tasks.md
```

## Key Changes

| Old | New |
|-----|-----|
| `decisions/D001_*.md` | `specs/*/plan.md` (technical decisions) |
| `features/F001_*.md` | `specs/*/spec.md` (user stories) |
| `implementation/I001_*.md` | `specs/*/plan.md` + `tasks.md` |
| `proposals/P001_*.md` | Merged into `plan.md` or `memory/` |

## Benefits of Spec Kit

1. **Organized by feature** - All artifacts for a feature in one directory
2. **Clear workflow** - spec → plan → tasks → implement
3. **Parallelizable tasks** - `[P]` markers enable parallel execution
4. **Traceable** - User stories link to tasks via `[Story]` tags
5. **Constitutional governance** - Principles enforced via gates in plans
