# Karaoke Cambodia Project Constitution

> Governing principles and development guidelines for the Karaoke Cambodia MVP project.

**Established**: 2024-12-02
**Last Amended**: 2024-12-02
**Status**: Active

---

## Preamble

This constitution establishes the foundational principles, architectural constraints, and development guidelines that govern all specifications and implementations for the Karaoke Cambodia platform. These articles are designed to ensure consistency, maintainability, and quality across the entire codebase.

---

## Article I: Library-First Principle

**Every significant feature MUST be implemented as a reusable package.**

1.1. Core business logic SHALL reside in the `packages/` directory, not within application code.

1.2. Applications (`apps/web`, `apps/admin`) SHALL consume packages as dependencies, maintaining clear separation of concerns.

1.3. The karaoke engine, audio processing, and scoring logic MUST be implemented as `@karaoke/engine` package.

1.4. Shared utilities, types, and validators MUST be in `@karaoke/utils` and `@karaoke/types` packages.

1.5. Database access and tRPC routers MUST be in `@karaoke/db` and `@karaoke/trpc` packages.

---

## Article II: Type Safety Imperative

**All code MUST be fully type-safe with no implicit `any` types.**

2.1. TypeScript strict mode SHALL be enabled in all packages and applications.

2.2. tRPC with Zod validation SHALL be used for all API contracts, ensuring end-to-end type safety.

2.3. Prisma client SHALL be the only database access layer, providing generated types.

2.4. All function parameters and return types MUST be explicitly typed or correctly inferred.

2.5. No `@ts-ignore` or `@ts-expect-error` comments without documented justification.

---

## Article III: Test-First Imperative

**Tests MUST be written before or alongside implementation, never after.**

3.1. Unit tests SHALL cover all utility functions and pure business logic.

3.2. Integration tests SHALL verify tRPC endpoint behavior with real database connections.

3.3. Component tests SHALL use React Testing Library for user interaction flows.

3.4. The karaoke engine MUST have comprehensive tests for pitch detection accuracy and scoring algorithms.

3.5. Minimum code coverage target: 80% for packages, 60% for applications.

---

## Article IV: Client-Side Audio Processing

**All real-time audio processing MUST occur on the client to ensure zero latency.**

4.1. Pitch detection SHALL use the YIN algorithm implemented in JavaScript/TypeScript.

4.2. The Web Audio API SHALL be the foundation for all audio analysis.

4.3. No audio data SHALL be sent to the server during karaoke sessions.

4.4. Scoring calculations MUST complete within 50ms to maintain real-time feedback.

4.5. AudioWorklet SHALL be preferred over deprecated ScriptProcessorNode where browser support allows.

---

## Article V: Simplicity Constraints

**The monorepo SHALL contain no more than necessary packages and applications.**

5.1. Maximum of 2 applications: `apps/web` (user-facing) and `apps/admin` (content management).

5.2. Maximum of 6 core packages without documented justification:
   - `@karaoke/db` - Database client and Prisma schema
   - `@karaoke/trpc` - API routers and procedures
   - `@karaoke/engine` - Karaoke engine (pitch, sync, scoring)
   - `@karaoke/types` - Shared TypeScript types
   - `@karaoke/utils` - Shared utility functions
   - `@karaoke/ui` - Shared UI components (optional)

5.3. Additional packages require documented justification in the Complexity Tracking section.

5.4. No micro-packages; utilities should be grouped logically.

---

## Article VI: Anti-Abstraction Principles

**Use framework and library features directly; avoid unnecessary abstraction layers.**

6.1. Next.js App Router patterns SHALL be used directly without wrapper abstractions.

6.2. tRPC procedures SHALL be defined using standard tRPC patterns, not custom wrappers.

6.3. Prisma client SHALL be used directly for database operations.

6.4. React hooks SHALL follow standard patterns; avoid creating hooks that merely wrap other hooks.

6.5. Any abstraction layer MUST provide documented, measurable value (reduced code, improved testability, or significant DX improvement).

---

## Article VII: Integration-First Testing

**Integration boundaries MUST be tested with real services, not mocks.**

7.1. Database tests SHALL use a real PostgreSQL instance (local or Docker).

7.2. tRPC tests SHALL make actual procedure calls, not mock implementations.

7.3. API contract tests MUST be defined and passing before implementation begins.

7.4. Mocks are permitted only for external services (OAuth providers, R2 storage) with documented justification.

7.5. The test database SHALL be seeded with representative data for all test scenarios.

---

## Article VIII: Progressive Enhancement

**Core functionality MUST work without JavaScript; enhance with client-side features.**

8.1. Song browsing and search MUST be server-rendered for SEO and initial load performance.

8.2. The karaoke mode MAY require JavaScript (Web Audio API is inherently client-side).

8.3. Authentication flows MUST handle no-JavaScript gracefully with server-side redirects.

8.4. Form submissions MUST work with native form behavior as fallback.

8.5. Error states MUST be visible without JavaScript execution.

---

## Article IX: Khmer Language Support

**The platform MUST fully support Khmer script rendering and input.**

9.1. Noto Sans Khmer SHALL be the primary font for Khmer text.

9.2. Text rendering MUST use proper line-height (1.8) and letter-spacing (0.02em) for readability.

9.3. Search functionality MUST handle Khmer input with fuzzy matching via pg_trgm.

9.4. Lyrics display MUST render Khmer script correctly at all zoom levels.

9.5. UI labels SHALL support both Khmer and English (future i18n consideration).

---

## Article X: Security Requirements

**All user data and interactions MUST follow security best practices.**

10.1. Authentication SHALL use NextAuth v5 with OAuth providers (Google, Facebook).

10.2. All API endpoints with user data MUST require authentication.

10.3. Admin routes MUST verify admin role before allowing access.

10.4. File uploads SHALL use presigned URLs; files MUST NOT pass through the application server.

10.5. Environment variables containing secrets MUST NOT be committed to version control.

10.6. SQL queries MUST use parameterized statements (Prisma handles this automatically).

---

## Article XI: Performance Budgets

**All user-facing pages MUST meet performance targets.**

11.1. Largest Contentful Paint (LCP): ≤ 2.5 seconds
11.2. First Input Delay (FID): ≤ 100 milliseconds
11.3. Cumulative Layout Shift (CLS): ≤ 0.1
11.4. Time to Interactive (TTI): ≤ 3.5 seconds
11.5. JavaScript bundle size: ≤ 200KB gzipped for initial load
11.6. Audio processing latency: ≤ 50ms end-to-end

---

## Amendments

Amendments to this constitution require:

1. Documented proposal with rationale
2. Impact analysis on existing specifications
3. Update to all affected specification documents
4. Version increment and dated changelog entry

---

## Complexity Tracking

Any deviation from Articles V or VI MUST be documented here:

| Article | Deviation | Justification | Date |
|---------|-----------|---------------|------|
| - | None | - | - |

---

## Related Documents

- `/specs/` - Feature specifications
- `/memory/research.md` - Technical research and decisions (if needed)
- `/KARAOKE_CAMBODIA_MVP_PLAN.md` - High-level project roadmap
