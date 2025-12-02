# I001: Frontend Structure Implementation

## Overview

Implementation notes for the Next.js 16 frontend application structure, covering component architecture, state management, and routing patterns.

## Related Decisions

- D009: Frontend Framework (Next.js 16)
- D010: State Management (Zustand)
- D001: Karaoke Engine (Client-side)

## Directory Structure

```
apps/web/
├── app/                          # Next.js App Router
│   ├── (public)/                 # Public routes (no auth required)
│   │   ├── page.tsx              # Home page
│   │   ├── songs/
│   │   │   ├── page.tsx          # Song listing
│   │   │   └── [id]/
│   │   │       └── page.tsx      # Song detail + karaoke
│   │   ├── search/
│   │   │   └── page.tsx          # Search results
│   │   └── layout.tsx            # Public layout (navbar, footer)
│   ├── (auth)/                   # Auth routes
│   │   ├── login/page.tsx
│   │   ├── register/page.tsx
│   │   └── layout.tsx
│   ├── (dashboard)/              # Protected routes
│   │   ├── profile/page.tsx
│   │   ├── favorites/page.tsx
│   │   └── layout.tsx            # Auth guard layout
│   ├── api/
│   │   ├── trpc/[trpc]/route.ts  # tRPC handler
│   │   └── auth/[...nextauth]/route.ts
│   ├── layout.tsx                # Root layout
│   └── globals.css
├── components/
│   ├── karaoke/                  # Karaoke-specific components
│   │   ├── KaraokeMode.tsx
│   │   ├── LyricsPanel.tsx
│   │   ├── PitchVisualizer.tsx
│   │   ├── ScoreDisplay.tsx
│   │   └── KaraokeControls.tsx
│   ├── audio/
│   │   ├── AudioPlayer.tsx
│   │   └── AudioControls.tsx
│   ├── song/
│   │   ├── SongCard.tsx
│   │   ├── SongList.tsx
│   │   └── SongDetail.tsx
│   ├── search/
│   │   └── SearchBar.tsx
│   ├── layout/
│   │   ├── Navbar.tsx
│   │   ├── Footer.tsx
│   │   └── Sidebar.tsx
│   └── ui/                       # shadcn/ui components
├── hooks/
│   ├── useKaraoke.ts
│   ├── usePitchDetection.ts
│   ├── useLyricsSync.ts
│   ├── useAudio.ts
│   └── useDebounce.ts
├── lib/
│   ├── trpc.ts                   # tRPC client setup
│   ├── auth.ts                   # NextAuth helpers
│   └── utils.ts                  # General utilities
├── stores/
│   ├── karaoke-store.ts          # Karaoke session state
│   └── user-store.ts             # User state
└── public/
    └── fonts/                    # Khmer fonts
```

## Component Types

### Server Components (RSC)

Used for static content and data fetching:

```tsx
// app/(public)/songs/page.tsx
import { trpc } from '@/lib/trpc'
import { SongList } from '@/components/song/SongList'

export default async function SongsPage() {
  const songs = await trpc.song.list.query({ take: 20 })

  return (
    <main>
      <h1>Browse Songs</h1>
      <SongList initialData={songs} />
    </main>
  )
}
```

### Client Components

Used for interactivity:

```tsx
// components/karaoke/KaraokeMode.tsx
'use client'

import { useKaraoke } from '@/hooks/useKaraoke'
import { LyricsPanel } from './LyricsPanel'
import { PitchVisualizer } from './PitchVisualizer'

export function KaraokeMode({ song }: { song: SongDTO }) {
  const { isPlaying, currentLyricIndex, score } = useKaraoke(song)

  return (
    <div>
      <LyricsPanel
        lyrics={song.lyricsJson}
        currentIndex={currentLyricIndex}
      />
      <PitchVisualizer />
      <ScoreDisplay score={score} />
    </div>
  )
}
```

## State Management

### Zustand Store Setup

```typescript
// stores/karaoke-store.ts
import { create } from 'zustand'

interface KaraokeState {
  isPlaying: boolean
  isSinging: boolean
  currentTime: number
  currentLyricIndex: number
  detectedFrequency: number | null
  transpose: number
  toleranceCents: number
  framesCorrect: number
  framesTotal: number

  // Actions
  setPlaying: (playing: boolean) => void
  setSinging: (singing: boolean) => void
  updateTime: (time: number) => void
  updateLyricIndex: (index: number) => void
  updateScore: (correct: number, total: number) => void
  setTranspose: (semitones: number) => void
  setTolerance: (cents: number) => void
  reset: () => void
}

export const useKaraokeStore = create<KaraokeState>((set) => ({
  isPlaying: false,
  isSinging: false,
  currentTime: 0,
  currentLyricIndex: 0,
  detectedFrequency: null,
  transpose: 0,
  toleranceCents: 80,
  framesCorrect: 0,
  framesTotal: 0,

  setPlaying: (playing) => set({ isPlaying: playing }),
  setSinging: (singing) => set({ isSinging: singing }),
  updateTime: (time) => set({ currentTime: time }),
  updateLyricIndex: (index) => set({ currentLyricIndex: index }),
  updateScore: (correct, total) => set({
    framesCorrect: correct,
    framesTotal: total
  }),
  setTranspose: (semitones) => set({ transpose: semitones }),
  setTolerance: (cents) => set({ toleranceCents: cents }),
  reset: () => set({
    isPlaying: false,
    isSinging: false,
    currentTime: 0,
    currentLyricIndex: 0,
    framesCorrect: 0,
    framesTotal: 0,
  }),
}))
```

## tRPC Client Setup

```typescript
// lib/trpc.ts
import { createTRPCReact } from '@trpc/react-query'
import { httpBatchLink } from '@trpc/client'
import type { AppRouter } from '@karaoke/trpc'

export const trpc = createTRPCReact<AppRouter>()

export function createTRPCClient() {
  return trpc.createClient({
    links: [
      httpBatchLink({
        url: `${process.env.NEXT_PUBLIC_APP_URL}/api/trpc`,
      }),
    ],
  })
}
```

## Streaming & Suspense Patterns

```tsx
// app/(public)/songs/page.tsx
import { Suspense } from 'react'
import { SongListSkeleton } from '@/components/song/SongListSkeleton'
import { SongList } from '@/components/song/SongList'

export default function SongsPage() {
  return (
    <main>
      <h1>Browse Songs</h1>
      <Suspense fallback={<SongListSkeleton />}>
        <SongList />
      </Suspense>
    </main>
  )
}
```

## Khmer Font Support

```css
/* globals.css */
@import url('https://fonts.googleapis.com/css2?family=Noto+Sans+Khmer:wght@400;700&display=swap');

:root {
  --font-khmer: 'Noto Sans Khmer', sans-serif;
}

.khmer-text {
  font-family: var(--font-khmer);
}
```

## Performance Optimizations

1. **Memoization**: Use `React.memo` for LyricsPanel items
2. **RAF for animations**: Use `requestAnimationFrame` for lyrics sync
3. **DOM manipulation**: Update lyric highlight via CSS class, not re-render
4. **Code splitting**: Dynamic import for karaoke engine
5. **Image optimization**: Next.js Image component for covers

## Testing Setup

```typescript
// vitest.config.ts
export default {
  test: {
    environment: 'jsdom',
    setupFiles: ['./test/setup.ts'],
  },
}
```

## Checklist

- [ ] Set up Next.js 16 with App Router
- [ ] Configure Tailwind CSS
- [ ] Install and configure shadcn/ui
- [ ] Set up tRPC client
- [ ] Create Zustand stores
- [ ] Implement layout components
- [ ] Add Khmer font support
- [ ] Configure path aliases
