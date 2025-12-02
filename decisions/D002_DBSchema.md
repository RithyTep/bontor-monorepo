# D002: Database Schema Design

## Status
- [ ] Proposed
- [x] Approved
- [ ] Superseded by D###

## Context

Karaoke Cambodia needs a database to store:
- User accounts and authentication data
- Songs with metadata, audio URLs, and lyrics
- Artists and categories for organization
- User favorites and score history

The database choice affects performance, developer experience, and operational costs.

## Alternatives Considered

### Option A: MongoDB (Document Store)

**Pros:**
- Flexible schema for varied song metadata
- Easy to store nested lyrics JSON
- Horizontal scaling built-in
- Good for rapid prototyping

**Cons:**
- Weaker relational queries (favorites, joins)
- No ACID transactions by default
- Prisma MongoDB support is less mature
- Over-engineered for our data model

### Option B: PostgreSQL (Relational)

**Pros:**
- Strong relational integrity (user-favorites-songs)
- ACID compliance
- Excellent Prisma support
- pg_trgm for fuzzy search (Khmer + English)
- JSON/JSONB columns for lyrics
- Neon offers serverless PostgreSQL

**Cons:**
- Schema migrations required
- Horizontal scaling more complex (not needed for MVP)

### Option C: PlanetScale (MySQL-compatible)

**Pros:**
- Serverless MySQL
- Branching for schema changes
- Good Prisma support

**Cons:**
- No foreign key constraints (by design)
- Less flexible JSON support than PostgreSQL
- No pg_trgm equivalent for fuzzy search

## Decision

**PostgreSQL with Neon (Serverless)**

Use PostgreSQL hosted on Neon with Prisma ORM for all data persistence.

## Rationale

1. **Relational Model Fit:** Our data is inherently relational (users have favorites, songs belong to artists/categories). PostgreSQL excels here.

2. **Prisma Excellence:** Prisma has first-class PostgreSQL support with excellent type generation and migration tools.

3. **Search Capability:** pg_trgm extension enables fuzzy search across song titles and artist names without external search service.

4. **JSONB for Lyrics:** PostgreSQL JSONB columns efficiently store parsed lyrics while maintaining relational integrity for metadata.

5. **Serverless Fit:** Neon provides serverless PostgreSQL with auto-scaling and connection pooling, perfect for Vercel deployment.

6. **Cost Effective:** Neon's free tier is generous; paid tiers are reasonable for MVP scale.

## Impact

### Schema Design

```prisma
// Core Models
- User (auth, profile)
- Account, Session, VerificationToken (NextAuth)
- Artist (name, slug, songs[])
- Category (name, slug, songs[])
- Song (title, audioUrl, lyricsJson, artist, category)
- Favorite (user, song, many-to-many)
- Score (user, song, totalScore, breakdown)
```

### Indexing Strategy

| Table | Index | Purpose |
|-------|-------|---------|
| song | title (B-tree) | Exact match |
| song | title (GIN pg_trgm) | Fuzzy search |
| song | artistId, categoryId | Joins |
| user | email | Auth lookup |
| favorite | (userId, songId) | Unique constraint |

### Operations

- Migrations via `prisma migrate`
- Neon handles backups automatically
- Connection pooling via Neon's proxy

## Related

- Features: F002_SongCMS, F007_Search, F008_Favorites
- Decisions: D012_SearchImplementation
- Implementation: I002_BackendAPI
- Proposals: P003_DBIndexing
