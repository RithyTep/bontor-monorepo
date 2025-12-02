# D004: Caching Strategy

## Status
- [ ] Proposed
- [x] Approved
- [ ] Superseded by D###

## Context

Karaoke Cambodia needs efficient caching to:
- Reduce database load for frequently accessed songs
- Minimize audio load times
- Improve perceived performance
- Control infrastructure costs

## Alternatives Considered

### Option A: Redis (Dedicated Cache)

**Pros:**
- Powerful caching capabilities
- TTL support, pub/sub
- Session storage option
- Industry standard

**Cons:**
- Additional infrastructure cost
- Another service to manage
- Over-engineering for MVP scale
- Adds latency (network hop)

### Option B: Vercel KV (Redis-compatible)

**Pros:**
- Managed by Vercel
- Edge-optimized
- Simple integration
- No infrastructure management

**Cons:**
- Vendor lock-in
- Cost at scale
- Limited to Vercel ecosystem

### Option C: CDN + Browser Caching

**Pros:**
- Zero infrastructure cost (included with Vercel/Cloudflare)
- Edge-distributed globally
- Browser caching for repeat visits
- Simple implementation

**Cons:**
- Less granular cache control
- Cache invalidation limited
- No server-side session caching

## Decision

**CDN (Vercel Edge + Cloudflare) + Browser Caching**

For MVP, rely on CDN edge caching for static assets and audio files, plus browser caching with appropriate headers. Defer Redis/KV to post-MVP if needed.

## Rationale

1. **Simplicity:** No additional infrastructure to manage. CDN caching is automatic with proper headers.

2. **Cost:** CDN caching is included with Vercel and Cloudflare R2. No additional services needed.

3. **Performance:** Edge caching delivers content from nearest location. For Cambodia, Cloudflare's Singapore/Bangkok POPs provide good coverage.

4. **MVP Scale:** Expected traffic doesn't justify dedicated caching layer. Database queries are efficient with proper indexing.

5. **Audio Optimization:** Audio files are immutable (versioned URLs), perfect for aggressive CDN caching.

## Impact

### Caching Headers

| Resource | Cache-Control | TTL |
|----------|---------------|-----|
| Audio files (R2) | `public, max-age=31536000, immutable` | 1 year |
| Static assets | `public, max-age=31536000, immutable` | 1 year |
| API responses | `private, no-cache` | None |
| HTML pages | `s-maxage=60, stale-while-revalidate` | 60s |
| Song metadata | Client-side SWR | 5 min |

### Implementation

```typescript
// Next.js page with revalidation
export const revalidate = 60 // Revalidate every 60 seconds

// tRPC with React Query (SWR pattern)
const { data: songs } = trpc.song.list.useQuery(
  { take: 20 },
  { staleTime: 5 * 60 * 1000 } // 5 minutes
)
```

### Audio URL Versioning

```
https://r2.karaoke.example.com/audio/{songId}/{version}/{filename}.mp3
```

Version in URL enables:
- Aggressive caching (immutable)
- Cache busting on audio updates
- CDN cache invalidation avoided

## Related

- Decisions: D005_Deployment, D006_AudioStorage
- Implementation: I001_FrontendStructure
