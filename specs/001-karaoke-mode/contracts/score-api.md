# API Contract: Score Submission

**Feature**: `001-karaoke-mode`
**Created**: 2024-12-02

---

## Overview

This contract defines the tRPC procedures for submitting and retrieving karaoke scores.

---

## Procedures

### score.submit

Submit a score after completing a karaoke session.

**Type**: `mutation`
**Auth**: Required (protected procedure)

#### Input Schema

```typescript
const ScoreSubmitInput = z.object({
  songId: z.string().uuid(),
  totalScore: z.number().int().min(0).max(100),
  pitchAccuracy: z.number().min(0).max(1),
  durationCoverage: z.number().min(0).max(1),
  transposeSemitones: z.number().int().min(-12).max(12).default(0),
  toleranceCents: z.number().int().min(20).max(200).default(80),
})
```

#### Output Schema

```typescript
const ScoreSubmitOutput = z.object({
  id: z.string().uuid(),
  totalScore: z.number(),
  isPersonalBest: z.boolean(),
  rank: z.number().nullable(), // Position on leaderboard, null if not top 100
})
```

#### Example Request

```json
{
  "songId": "550e8400-e29b-41d4-a716-446655440000",
  "totalScore": 85,
  "pitchAccuracy": 0.82,
  "durationCoverage": 0.91,
  "transposeSemitones": 0,
  "toleranceCents": 80
}
```

#### Example Response

```json
{
  "id": "660e8400-e29b-41d4-a716-446655440001",
  "totalScore": 85,
  "isPersonalBest": true,
  "rank": 42
}
```

#### Error Codes

| Code | Condition |
|------|-----------|
| `UNAUTHORIZED` | User not authenticated |
| `NOT_FOUND` | Song does not exist |
| `BAD_REQUEST` | Invalid input values |

---

### score.history

Get user's score history for a specific song.

**Type**: `query`
**Auth**: Required (protected procedure)

#### Input Schema

```typescript
const ScoreHistoryInput = z.object({
  songId: z.string().uuid(),
  limit: z.number().int().min(1).max(50).default(10),
})
```

#### Output Schema

```typescript
const ScoreHistoryOutput = z.array(z.object({
  id: z.string().uuid(),
  totalScore: z.number(),
  pitchAccuracy: z.number(),
  durationCoverage: z.number(),
  createdAt: z.date(),
}))
```

#### Example Request

```json
{
  "songId": "550e8400-e29b-41d4-a716-446655440000",
  "limit": 5
}
```

#### Example Response

```json
[
  {
    "id": "660e8400-e29b-41d4-a716-446655440001",
    "totalScore": 85,
    "pitchAccuracy": 0.82,
    "durationCoverage": 0.91,
    "createdAt": "2024-12-02T10:30:00.000Z"
  },
  {
    "id": "660e8400-e29b-41d4-a716-446655440002",
    "totalScore": 78,
    "pitchAccuracy": 0.75,
    "durationCoverage": 0.85,
    "createdAt": "2024-12-01T14:20:00.000Z"
  }
]
```

---

### score.best

Get user's personal best score for a song.

**Type**: `query`
**Auth**: Required (protected procedure)

#### Input Schema

```typescript
const ScoreBestInput = z.object({
  songId: z.string().uuid(),
})
```

#### Output Schema

```typescript
const ScoreBestOutput = z.object({
  id: z.string().uuid(),
  totalScore: z.number(),
  pitchAccuracy: z.number(),
  durationCoverage: z.number(),
  createdAt: z.date(),
}).nullable()
```

#### Example Response (has best score)

```json
{
  "id": "660e8400-e29b-41d4-a716-446655440001",
  "totalScore": 92,
  "pitchAccuracy": 0.90,
  "durationCoverage": 0.95,
  "createdAt": "2024-11-28T16:45:00.000Z"
}
```

#### Example Response (no scores yet)

```json
null
```

---

### score.leaderboard

Get top scores for a song (public).

**Type**: `query`
**Auth**: None (public procedure)

#### Input Schema

```typescript
const ScoreLeaderboardInput = z.object({
  songId: z.string().uuid(),
  limit: z.number().int().min(1).max(100).default(10),
})
```

#### Output Schema

```typescript
const ScoreLeaderboardOutput = z.array(z.object({
  rank: z.number(),
  userId: z.string().uuid(),
  userName: z.string(),
  userImage: z.string().url().nullable(),
  totalScore: z.number(),
  createdAt: z.date(),
}))
```

#### Example Response

```json
[
  {
    "rank": 1,
    "userId": "user-001",
    "userName": "Sokha",
    "userImage": "https://example.com/avatar1.jpg",
    "totalScore": 98,
    "createdAt": "2024-12-01T09:00:00.000Z"
  },
  {
    "rank": 2,
    "userId": "user-002",
    "userName": "Dara",
    "userImage": null,
    "totalScore": 95,
    "createdAt": "2024-11-30T15:30:00.000Z"
  }
]
```

---

## tRPC Router Implementation

```typescript
// packages/trpc/routers/score.ts
import { z } from 'zod'
import { router, protectedProcedure, publicProcedure } from '../trpc'

export const scoreRouter = router({
  submit: protectedProcedure
    .input(ScoreSubmitInput)
    .output(ScoreSubmitOutput)
    .mutation(async ({ input, ctx }) => {
      // Implementation
    }),

  history: protectedProcedure
    .input(ScoreHistoryInput)
    .output(ScoreHistoryOutput)
    .query(async ({ input, ctx }) => {
      // Implementation
    }),

  best: protectedProcedure
    .input(ScoreBestInput)
    .output(ScoreBestOutput)
    .query(async ({ input, ctx }) => {
      // Implementation
    }),

  leaderboard: publicProcedure
    .input(ScoreLeaderboardInput)
    .output(ScoreLeaderboardOutput)
    .query(async ({ input, ctx }) => {
      // Implementation
    }),
})
```

---

## Contract Tests

```typescript
// packages/trpc/tests/score.test.ts
describe('score router', () => {
  describe('submit', () => {
    it('should save score and return personal best status', async () => {
      // Test implementation
    })

    it('should reject unauthenticated requests', async () => {
      // Test implementation
    })

    it('should reject invalid song ID', async () => {
      // Test implementation
    })
  })

  describe('leaderboard', () => {
    it('should return top scores ordered by totalScore desc', async () => {
      // Test implementation
    })

    it('should respect limit parameter', async () => {
      // Test implementation
    })
  })
})
```
