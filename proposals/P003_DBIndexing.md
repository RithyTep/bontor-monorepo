# P003: Database Indexing Strategy

## Status: Approved

## Context

The Karaoke Cambodia application needs efficient database queries for:
- Song search (title, artist name) in both Khmer and English
- Listing songs by category with pagination
- User favorites retrieval
- Leaderboard/scoring queries

Without proper indexing, query performance will degrade as the catalog grows beyond 1,000 songs.

## Proposal

Implement a multi-layered indexing strategy using PostgreSQL's native capabilities, starting with pg_trgm for fuzzy search and B-tree indexes for common lookups.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DATABASE INDEXING STRATEGY                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  LAYER 1: B-TREE INDEXES (Primary/Foreign Keys)                         │
│  ├── Song.id (PRIMARY)                                                  │
│  ├── Song.artistId → Artist.id (FK)                                     │
│  ├── Song.categoryId → Category.id (FK)                                 │
│  ├── Favorite.userId → User.id (FK)                                     │
│  └── Score.songId → Song.id (FK)                                        │
│                                                                          │
│  LAYER 2: GiST INDEXES (Fuzzy Search with pg_trgm)                      │
│  ├── Song.title (trigram similarity)                                    │
│  ├── Artist.name (trigram similarity)                                   │
│  └── Song.slug (prefix matching)                                        │
│                                                                          │
│  LAYER 3: COMPOSITE INDEXES (Common Query Patterns)                     │
│  ├── Song(categoryId, isPublished, createdAt)                          │
│  ├── Favorite(userId, createdAt)                                        │
│  └── Score(songId, totalScore DESC)                                     │
│                                                                          │
│  LAYER 4: PARTIAL INDEXES (Filtered Queries)                            │
│  └── Song WHERE isPublished = true                                      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## Technical Specifications

### 1. Enable pg_trgm Extension

```sql
-- Migration: 001_enable_extensions.sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

### 2. Primary B-Tree Indexes (Automatic via Prisma)

```prisma
// schema.prisma - Primary keys create B-tree indexes automatically
model Song {
  id          String   @id @default(uuid())
  artistId    String
  categoryId  String

  // Foreign key indexes (created automatically by Prisma)
  artist      Artist   @relation(fields: [artistId], references: [id])
  category    Category @relation(fields: [categoryId], references: [id])
}
```

### 3. GiST Indexes for Fuzzy Search

```sql
-- Migration: 002_fuzzy_search_indexes.sql

-- Song title fuzzy search (Khmer + English)
CREATE INDEX idx_song_title_trgm ON "Song"
  USING GiST (title gist_trgm_ops);

-- Artist name fuzzy search
CREATE INDEX idx_artist_name_trgm ON "Artist"
  USING GiST (name gist_trgm_ops);

-- Song slug for URL lookups
CREATE INDEX idx_song_slug ON "Song" (slug);
```

### 4. Composite Indexes for Common Queries

```sql
-- Migration: 003_composite_indexes.sql

-- Songs by category (with published filter and ordering)
CREATE INDEX idx_song_category_published ON "Song" (
  "categoryId",
  "isPublished",
  "createdAt" DESC
) WHERE "isPublished" = true;

-- User favorites (ordered by most recent)
CREATE INDEX idx_favorite_user_created ON "Favorite" (
  "userId",
  "createdAt" DESC
);

-- Song scores for leaderboard
CREATE INDEX idx_score_song_score ON "Score" (
  "songId",
  "totalScore" DESC
);

-- User scores for history
CREATE INDEX idx_score_user_song ON "Score" (
  "userId",
  "songId",
  "createdAt" DESC
);
```

### 5. Partial Indexes

```sql
-- Migration: 004_partial_indexes.sql

-- Only index published songs (most common query)
CREATE INDEX idx_song_published ON "Song" ("createdAt" DESC)
  WHERE "isPublished" = true;

-- Recently added songs (homepage carousel)
CREATE INDEX idx_song_recent_published ON "Song" ("createdAt" DESC)
  WHERE "isPublished" = true AND "createdAt" > (NOW() - INTERVAL '30 days');
```

### Query Examples with Index Usage

#### Fuzzy Search Query

```typescript
// Using pg_trgm similarity operator (%)
const songs = await prisma.$queryRaw`
  SELECT s.*, similarity(s.title, ${query}) AS title_sim
  FROM "Song" s
  LEFT JOIN "Artist" a ON s."artistId" = a.id
  WHERE s."isPublished" = true
    AND (
      s.title % ${query}
      OR a.name % ${query}
    )
  ORDER BY
    GREATEST(
      similarity(s.title, ${query}),
      similarity(a.name, ${query})
    ) DESC
  LIMIT ${take}
`
```

#### Category Listing Query

```typescript
// Uses idx_song_category_published composite index
const songs = await prisma.song.findMany({
  where: {
    categoryId: categoryId,
    isPublished: true,
  },
  orderBy: { createdAt: 'desc' },
  take: 20,
  skip: 0,
})
```

#### User Favorites Query

```typescript
// Uses idx_favorite_user_created composite index
const favorites = await prisma.favorite.findMany({
  where: { userId: userId },
  include: { song: { include: { artist: true } } },
  orderBy: { createdAt: 'desc' },
})
```

### Index Monitoring

```sql
-- Check index usage statistics
SELECT
  schemaname,
  tablename,
  indexname,
  idx_scan AS times_used,
  idx_tup_read AS tuples_read,
  idx_tup_fetch AS tuples_fetched
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan DESC;

-- Find unused indexes (candidates for removal)
SELECT
  indexname,
  idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND schemaname = 'public';

-- Check table size vs index size
SELECT
  relname AS table_name,
  pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
  pg_size_pretty(pg_relation_size(relid)) AS table_size,
  pg_size_pretty(pg_total_relation_size(relid) - pg_relation_size(relid)) AS index_size
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(relid) DESC;
```

### EXPLAIN ANALYZE Examples

```sql
-- Test fuzzy search performance
EXPLAIN ANALYZE
SELECT * FROM "Song"
WHERE title % 'cambodia'
  AND "isPublished" = true
LIMIT 10;

-- Expected: Index Scan using idx_song_title_trgm

-- Test category listing performance
EXPLAIN ANALYZE
SELECT * FROM "Song"
WHERE "categoryId" = 'uuid-here'
  AND "isPublished" = true
ORDER BY "createdAt" DESC
LIMIT 20;

-- Expected: Index Scan using idx_song_category_published
```

## Prisma Schema Integration

```prisma
// schema.prisma

model Song {
  id          String   @id @default(uuid())
  title       String
  slug        String   @unique
  artistId    String
  categoryId  String
  isPublished Boolean  @default(false)
  createdAt   DateTime @default(now())

  artist   Artist   @relation(fields: [artistId], references: [id])
  category Category @relation(fields: [categoryId], references: [id])

  // Prisma indexes (B-tree)
  @@index([artistId])
  @@index([categoryId])
  @@index([slug])
  @@index([isPublished, createdAt(sort: Desc)])
}

model Favorite {
  id        String   @id @default(uuid())
  userId    String
  songId    String
  createdAt DateTime @default(now())

  user User @relation(fields: [userId], references: [id])
  song Song @relation(fields: [songId], references: [id])

  @@unique([userId, songId])
  @@index([userId, createdAt(sort: Desc)])
}

model Score {
  id         String   @id @default(uuid())
  userId     String
  songId     String
  totalScore Int
  createdAt  DateTime @default(now())

  user User @relation(fields: [userId], references: [id])
  song Song @relation(fields: [songId], references: [id])

  @@index([songId, totalScore(sort: Desc)])
  @@index([userId, songId, createdAt(sort: Desc)])
}
```

## Performance Targets

| Query Type | Target p95 | Max Dataset |
|------------|-----------|-------------|
| Fuzzy search | < 100ms | 10,000 songs |
| Category listing | < 50ms | 10,000 songs |
| User favorites | < 30ms | 1,000 favorites/user |
| Leaderboard | < 50ms | 100,000 scores |

## Migration Strategy

### Phase 1: MVP Launch
1. Enable pg_trgm extension
2. Create GiST indexes for title/artist fuzzy search
3. Create composite indexes for common queries

### Phase 2: Scale (>5,000 songs)
1. Monitor query performance
2. Add partial indexes for hot queries
3. Consider read replicas if needed

### Phase 3: Future (>10,000 songs)
1. Evaluate Meilisearch for dedicated search
2. Implement search indexing pipeline
3. Add Redis caching layer

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Index bloat over time | Medium | Regular VACUUM ANALYZE |
| Write performance degradation | Low | Async indexing for bulk imports |
| Khmer character handling | Medium | Test pg_trgm with Khmer corpus |
| Index not used by planner | Medium | EXPLAIN ANALYZE in development |

## Alternatives Considered

### 1. Full-Text Search (tsvector)
- **Pros**: Native PostgreSQL, language support
- **Cons**: Complex for Khmer, overkill for MVP
- **Decision**: Deferred - pg_trgm simpler for similarity

### 2. Elasticsearch/Meilisearch
- **Pros**: Powerful search, typo tolerance
- **Cons**: Additional infrastructure, complexity
- **Decision**: Deferred - pg_trgm sufficient for MVP scale

### 3. No Fuzzy Search (LIKE queries)
- **Pros**: Simple, no extensions
- **Cons**: Poor UX for Khmer input, slow on large datasets
- **Decision**: Rejected - fuzzy search essential for UX

## Related

- Decisions: D002_DBSchema, D012_SearchStrategy
- Implementation: I002_BackendAPI
- Features: F002_SongCMS
