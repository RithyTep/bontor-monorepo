# D003: API Layer Architecture

## Status
- [ ] Proposed
- [x] Approved
- [ ] Superseded by D###

## Context

Karaoke Cambodia needs an API layer for:
- Song listing, search, and detail fetching
- User authentication and profile management
- Favorites management
- Score submission and history
- Admin operations (song CRUD)

The API design affects type safety, developer velocity, and client-server contract maintenance.

## Alternatives Considered

### Option A: REST API

**Pros:**
- Industry standard, well understood
- HTTP caching built-in
- Easy to debug with curl/Postman
- OpenAPI spec for documentation

**Cons:**
- No built-in type safety across FE/BE
- Over-fetching or under-fetching data
- Multiple endpoints for related data
- Manual type synchronization

### Option B: GraphQL

**Pros:**
- Single endpoint, query what you need
- Strong typing with schema
- Great developer tools (Apollo)
- Efficient data fetching

**Cons:**
- Over-engineering for our use case
- Learning curve for team
- Caching more complex
- More infrastructure (Apollo Server)

### Option C: tRPC

**Pros:**
- Full type safety FE to BE (zero codegen)
- Monorepo synergy (shared types)
- Simple RPC-style calls
- Excellent Next.js integration
- Zod validation built-in
- Minimal boilerplate

**Cons:**
- TypeScript-only (not a con for us)
- Less ecosystem than REST/GraphQL
- Debugging less intuitive than REST

## Decision

**tRPC for all API communication**

Use tRPC v10 for type-safe client-server communication, deployed as Next.js API routes (Vercel Serverless).

## Rationale

1. **Type Safety:** tRPC provides end-to-end type safety without code generation. Changes in backend types immediately reflect in frontend.

2. **Monorepo Synergy:** With shared `@karaoke/trpc` package, router definitions are shared between apps.

3. **Developer Velocity:** No API documentation to maintain, IDE autocompletion for all procedures, Zod validation.

4. **Next.js Integration:** tRPC adapters work seamlessly with Next.js App Router and RSC.

5. **Simplicity:** RPC model is intuitive for our CRUD-heavy operations.

## Impact

### Router Structure

```typescript
appRouter
├── song
│   ├── list      // paginated, filterable
│   ├── get       // by ID
│   └── search    // fuzzy search
├── user
│   ├── me        // current user profile
│   └── update    // update profile
├── auth
│   ├── session   // get session
│   └── signOut   // logout
├── favorite
│   ├── list      // user's favorites
│   ├── add       // add favorite
│   └── remove    // remove favorite
├── artist
│   ├── list
│   └── get
├── category
│   └── list
└── score
    ├── submit    // save score
    └── history   // user's score history
```

### Validation

All inputs validated with Zod schemas:

```typescript
const SongListInput = z.object({
  q: z.string().optional(),
  categoryId: z.string().uuid().optional(),
  skip: z.number().int().min(0).default(0),
  take: z.number().int().min(1).max(50).default(20),
})
```

### Error Handling

Standard tRPC error codes:
- `NOT_FOUND` - Resource doesn't exist
- `UNAUTHORIZED` - Not authenticated
- `FORBIDDEN` - Not authorized
- `BAD_REQUEST` - Invalid input
- `INTERNAL_SERVER_ERROR` - Unexpected error

## Related

- Features: All features use tRPC
- Decisions: D002_DBSchema
- Implementation: I002_BackendAPI
