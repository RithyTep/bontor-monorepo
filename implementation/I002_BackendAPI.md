# I002: Backend API Implementation

## Overview

Implementation notes for the tRPC-based backend API, covering router structure, middleware, authentication, and database integration.

## Related Decisions

- D003: API Layer Architecture (tRPC)
- D002: Database Schema (PostgreSQL + Prisma)
- D007: Authentication (NextAuth v5)

## Package Structure

```
packages/trpc/
├── routers/
│   ├── song.ts
│   ├── user.ts
│   ├── auth.ts
│   ├── favorite.ts
│   ├── artist.ts
│   ├── category.ts
│   ├── score.ts
│   └── index.ts          # Root router
├── context.ts            # Request context
├── middleware.ts         # Auth & error middleware
├── trpc.ts              # tRPC instance
└── package.json
```

## tRPC Instance Setup

```typescript
// packages/trpc/trpc.ts
import { initTRPC, TRPCError } from '@trpc/server'
import { ZodError } from 'zod'
import type { Context } from './context'

const t = initTRPC.context<Context>().create({
  errorFormatter: ({ shape, error }) => ({
    ...shape,
    data: {
      ...shape.data,
      zodError: error.cause instanceof ZodError
        ? error.cause.flatten()
        : null,
    },
  }),
})

export const router = t.router
export const publicProcedure = t.procedure
export const middleware = t.middleware
```

## Context Creation

```typescript
// packages/trpc/context.ts
import { prisma } from '@karaoke/db'
import { getServerSession } from 'next-auth'
import { authOptions } from './auth-options'

export async function createContext({ req, res }: { req: Request; res: Response }) {
  const session = await getServerSession(authOptions)

  return {
    prisma,
    session,
    user: session?.user ?? null,
  }
}

export type Context = Awaited<ReturnType<typeof createContext>>
```

## Middleware

### Authentication Middleware

```typescript
// packages/trpc/middleware.ts
import { TRPCError } from '@trpc/server'
import { middleware } from './trpc'

export const isAuthenticated = middleware(async ({ ctx, next }) => {
  if (!ctx.session || !ctx.user) {
    throw new TRPCError({ code: 'UNAUTHORIZED' })
  }
  return next({
    ctx: {
      ...ctx,
      user: ctx.user, // Now typed as non-null
    },
  })
})

export const isAdmin = middleware(async ({ ctx, next }) => {
  if (!ctx.user || ctx.user.role !== 'admin') {
    throw new TRPCError({ code: 'FORBIDDEN' })
  }
  return next({ ctx })
})
```

### Rate Limiting Middleware

```typescript
// packages/trpc/middleware.ts
import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(100, '1 m'),
})

export const withRateLimit = (limit: number) =>
  middleware(async ({ ctx, next }) => {
    const ip = ctx.req.headers.get('x-forwarded-for') ?? 'anonymous'
    const { success } = await ratelimit.limit(`${ip}:${limit}`)

    if (!success) {
      throw new TRPCError({ code: 'TOO_MANY_REQUESTS' })
    }
    return next()
  })
```

## Procedure Types

```typescript
// packages/trpc/trpc.ts
import { publicProcedure, middleware } from './trpc'
import { isAuthenticated, isAdmin } from './middleware'

// Public - no auth required
export { publicProcedure }

// Protected - requires authentication
export const protectedProcedure = publicProcedure.use(isAuthenticated)

// Admin - requires admin role
export const adminProcedure = protectedProcedure.use(isAdmin)
```

## Router Implementations

### Song Router

```typescript
// packages/trpc/routers/song.ts
import { z } from 'zod'
import { router, publicProcedure } from '../trpc'

const SongListInput = z.object({
  q: z.string().optional(),
  categoryId: z.string().uuid().optional(),
  skip: z.number().int().min(0).default(0),
  take: z.number().int().min(1).max(50).default(20),
})

const SongGetInput = z.object({
  id: z.string().uuid(),
})

export const songRouter = router({
  list: publicProcedure
    .input(SongListInput)
    .query(async ({ input, ctx }) => {
      const { q, categoryId, skip, take } = input

      const where = {
        isPublished: true,
        ...(categoryId && { categoryId }),
        ...(q && {
          OR: [
            { title: { contains: q, mode: 'insensitive' as const } },
            { artist: { name: { contains: q, mode: 'insensitive' as const } } },
          ],
        }),
      }

      const [items, total] = await Promise.all([
        ctx.prisma.song.findMany({
          where,
          skip,
          take,
          include: { artist: true, category: true },
          orderBy: { createdAt: 'desc' },
        }),
        ctx.prisma.song.count({ where }),
      ])

      return { items, total }
    }),

  get: publicProcedure
    .input(SongGetInput)
    .query(async ({ input, ctx }) => {
      const song = await ctx.prisma.song.findUnique({
        where: { id: input.id, isPublished: true },
        include: { artist: true, category: true },
      })

      if (!song) {
        throw new TRPCError({ code: 'NOT_FOUND' })
      }

      // Increment play count
      await ctx.prisma.song.update({
        where: { id: input.id },
        data: { playCount: { increment: 1 } },
      })

      return song
    }),

  search: publicProcedure
    .input(z.object({ q: z.string().min(1), take: z.number().default(10) }))
    .query(async ({ input, ctx }) => {
      // Using pg_trgm for fuzzy search
      const songs = await ctx.prisma.$queryRaw`
        SELECT s.*, similarity(s.title, ${input.q}) as sim
        FROM "Song" s
        WHERE s."isPublished" = true
          AND (s.title % ${input.q} OR EXISTS (
            SELECT 1 FROM "Artist" a
            WHERE a.id = s."artistId" AND a.name % ${input.q}
          ))
        ORDER BY sim DESC
        LIMIT ${input.take}
      `
      return songs
    }),
})
```

### Favorite Router

```typescript
// packages/trpc/routers/favorite.ts
import { z } from 'zod'
import { router, protectedProcedure } from '../trpc'

export const favoriteRouter = router({
  list: protectedProcedure.query(async ({ ctx }) => {
    return ctx.prisma.favorite.findMany({
      where: { userId: ctx.user.id },
      include: { song: { include: { artist: true } } },
      orderBy: { createdAt: 'desc' },
    })
  }),

  add: protectedProcedure
    .input(z.object({ songId: z.string().uuid() }))
    .mutation(async ({ input, ctx }) => {
      return ctx.prisma.favorite.create({
        data: {
          userId: ctx.user.id,
          songId: input.songId,
        },
      })
    }),

  remove: protectedProcedure
    .input(z.object({ songId: z.string().uuid() }))
    .mutation(async ({ input, ctx }) => {
      return ctx.prisma.favorite.delete({
        where: {
          userId_songId: {
            userId: ctx.user.id,
            songId: input.songId,
          },
        },
      })
    }),

  isFavorite: protectedProcedure
    .input(z.object({ songId: z.string().uuid() }))
    .query(async ({ input, ctx }) => {
      const favorite = await ctx.prisma.favorite.findUnique({
        where: {
          userId_songId: {
            userId: ctx.user.id,
            songId: input.songId,
          },
        },
      })
      return !!favorite
    }),
})
```

### Score Router

```typescript
// packages/trpc/routers/score.ts
import { z } from 'zod'
import { router, protectedProcedure, publicProcedure } from '../trpc'

const ScoreSubmitInput = z.object({
  songId: z.string().uuid(),
  totalScore: z.number().int().min(0).max(100),
  pitchAccuracy: z.number().min(0).max(1),
  durationCoverage: z.number().min(0).max(1),
  transposeSemitones: z.number().int().default(0),
  toleranceCents: z.number().int().default(80),
})

export const scoreRouter = router({
  submit: protectedProcedure
    .input(ScoreSubmitInput)
    .mutation(async ({ input, ctx }) => {
      return ctx.prisma.score.create({
        data: {
          userId: ctx.user.id,
          ...input,
        },
      })
    }),

  history: protectedProcedure
    .input(z.object({ songId: z.string().uuid() }))
    .query(async ({ input, ctx }) => {
      return ctx.prisma.score.findMany({
        where: {
          userId: ctx.user.id,
          songId: input.songId,
        },
        orderBy: { createdAt: 'desc' },
        take: 10,
      })
    }),

  best: protectedProcedure
    .input(z.object({ songId: z.string().uuid() }))
    .query(async ({ input, ctx }) => {
      return ctx.prisma.score.findFirst({
        where: {
          userId: ctx.user.id,
          songId: input.songId,
        },
        orderBy: { totalScore: 'desc' },
      })
    }),
})
```

## Root Router

```typescript
// packages/trpc/routers/index.ts
import { router } from '../trpc'
import { songRouter } from './song'
import { userRouter } from './user'
import { authRouter } from './auth'
import { favoriteRouter } from './favorite'
import { artistRouter } from './artist'
import { categoryRouter } from './category'
import { scoreRouter } from './score'

export const appRouter = router({
  song: songRouter,
  user: userRouter,
  auth: authRouter,
  favorite: favoriteRouter,
  artist: artistRouter,
  category: categoryRouter,
  score: scoreRouter,
})

export type AppRouter = typeof appRouter
```

## Next.js API Route Handler

```typescript
// apps/web/app/api/trpc/[trpc]/route.ts
import { fetchRequestHandler } from '@trpc/server/adapters/fetch'
import { appRouter } from '@karaoke/trpc'
import { createContext } from '@karaoke/trpc/context'

const handler = (req: Request) =>
  fetchRequestHandler({
    endpoint: '/api/trpc',
    req,
    router: appRouter,
    createContext: () => createContext({ req }),
  })

export { handler as GET, handler as POST }
```

## Error Handling

```typescript
// packages/trpc/error-handler.ts
import { TRPCError } from '@trpc/server'
import * as Sentry from '@sentry/node'

export function handleError(error: unknown): TRPCError {
  if (error instanceof TRPCError) {
    return error
  }

  // Log to Sentry
  Sentry.captureException(error)

  return new TRPCError({
    code: 'INTERNAL_SERVER_ERROR',
    message: 'An unexpected error occurred',
  })
}
```

## Checklist

- [ ] Set up tRPC package with routers
- [ ] Configure context with Prisma and session
- [ ] Implement authentication middleware
- [ ] Create all CRUD routers
- [ ] Add input validation with Zod
- [ ] Configure error handling
- [ ] Set up rate limiting
- [ ] Test all endpoints
- [ ] Add logging
