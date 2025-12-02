# P002: Real-Time Lyrics Synchronization

## Status: Approved

## Context

Users need to see lyrics highlighted in real-time as they sing along to karaoke tracks. The synchronization must be precise (within 100ms), handle Khmer text rendering correctly, and provide smooth visual transitions without jank or lag.

## Proposal

Implement a time-based lyrics synchronization system using `requestAnimationFrame` for smooth updates, with CSS-based highlighting for optimal performance.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   LYRICS SYNCHRONIZATION FLOW                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Audio Time ──────────────────────────────────────────────────────────▶ │
│       │                                                                  │
│       │  timeupdate event (250ms interval)                              │
│       │         │                                                        │
│       │         ▼                                                        │
│       │  ┌─────────────────┐                                            │
│       │  │ getCurrentTime  │                                            │
│       │  └────────┬────────┘                                            │
│       │           │                                                      │
│       │           ▼                                                      │
│       │  ┌─────────────────┐     ┌───────────────────┐                 │
│       │  │  Binary Search  │────▶│ Find Active Line  │                 │
│       │  │  O(log n)       │     │ lyrics[i].t <= t  │                 │
│       │  └─────────────────┘     └─────────┬─────────┘                 │
│       │                                     │                           │
│       │                                     ▼                           │
│       │                          ┌─────────────────────┐                │
│       ▼                          │  Update State       │                │
│  requestAnimationFrame ─────────▶│  currentIndex       │                │
│       │                          └─────────┬───────────┘                │
│       │                                    │                            │
│       │                                    ▼                            │
│       │                          ┌─────────────────────┐                │
│       │                          │  CSS Class Toggle   │                │
│       │                          │  .active-lyric      │                │
│       │                          └─────────┬───────────┘                │
│       │                                    │                            │
│       │                                    ▼                            │
│       │                          ┌─────────────────────┐                │
│       │                          │  Scroll Into View   │                │
│       │                          │  (smooth behavior)  │                │
│       │                          └─────────────────────┘                │
│       │                                                                 │
└───────┴─────────────────────────────────────────────────────────────────┘
```

## Technical Specifications

### Lyrics Data Structure

```typescript
interface LyricsLine {
  t: number   // Start time in seconds (float)
  l: string   // Lyric text (Khmer or English)
}

interface LyricsData {
  lyrics: LyricsLine[]
  metadata: {
    title?: string
    artist?: string
    language?: string
  }
}
```

### Synchronization Algorithm

```typescript
class LyricsSync {
  private lyrics: LyricsLine[]
  private currentIndex: number = -1
  private rafId: number | null = null

  constructor(lyrics: LyricsLine[]) {
    this.lyrics = lyrics
  }

  /**
   * Binary search for current lyric line
   * O(log n) complexity for large lyric files
   */
  findCurrentIndex(time: number): number {
    let left = 0
    let right = this.lyrics.length - 1
    let result = -1

    while (left <= right) {
      const mid = Math.floor((left + right) / 2)
      if (this.lyrics[mid].t <= time) {
        result = mid
        left = mid + 1
      } else {
        right = mid - 1
      }
    }

    return result
  }

  /**
   * Start synchronization loop
   */
  start(audioElement: HTMLAudioElement, onUpdate: (index: number) => void) {
    const sync = () => {
      const currentTime = audioElement.currentTime
      const newIndex = this.findCurrentIndex(currentTime)

      if (newIndex !== this.currentIndex) {
        this.currentIndex = newIndex
        onUpdate(newIndex)
      }

      if (!audioElement.paused) {
        this.rafId = requestAnimationFrame(sync)
      }
    }

    this.rafId = requestAnimationFrame(sync)
  }

  stop() {
    if (this.rafId) {
      cancelAnimationFrame(this.rafId)
      this.rafId = null
    }
  }
}
```

### React Component Implementation

```tsx
interface LyricsPanelProps {
  lyrics: LyricsLine[]
  currentIndex: number
  language?: 'km' | 'en'
}

export function LyricsPanel({ lyrics, currentIndex, language = 'km' }: LyricsPanelProps) {
  const containerRef = useRef<HTMLDivElement>(null)
  const activeRef = useRef<HTMLDivElement>(null)

  // Auto-scroll to active lyric
  useEffect(() => {
    if (activeRef.current && containerRef.current) {
      activeRef.current.scrollIntoView({
        behavior: 'smooth',
        block: 'center'
      })
    }
  }, [currentIndex])

  return (
    <div
      ref={containerRef}
      className="lyrics-container h-[400px] overflow-y-auto"
    >
      {lyrics.map((line, index) => (
        <div
          key={index}
          ref={index === currentIndex ? activeRef : null}
          className={cn(
            'lyric-line py-3 px-4 transition-all duration-200',
            language === 'km' && 'font-khmer text-xl',
            index === currentIndex && 'active-lyric bg-primary/20 scale-105',
            index < currentIndex && 'text-muted-foreground',
            index > currentIndex && 'text-foreground/70'
          )}
        >
          {line.l || '♪'}
        </div>
      ))}
    </div>
  )
}
```

### Performance Optimizations

#### 1. Virtual Scrolling for Long Lyrics

```tsx
import { useVirtualizer } from '@tanstack/react-virtual'

function VirtualLyricsPanel({ lyrics, currentIndex }: LyricsPanelProps) {
  const parentRef = useRef<HTMLDivElement>(null)

  const virtualizer = useVirtualizer({
    count: lyrics.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48, // Approximate line height
    overscan: 5,
  })

  // Auto-scroll to active
  useEffect(() => {
    virtualizer.scrollToIndex(currentIndex, { align: 'center', behavior: 'smooth' })
  }, [currentIndex])

  return (
    <div ref={parentRef} className="h-[400px] overflow-auto">
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map((virtualItem) => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: 0,
              transform: `translateY(${virtualItem.start}px)`,
            }}
            className={virtualItem.index === currentIndex ? 'active-lyric' : ''}
          >
            {lyrics[virtualItem.index].l}
          </div>
        ))}
      </div>
    </div>
  )
}
```

#### 2. CSS-Only Highlighting (No Re-renders)

```typescript
// Update via DOM manipulation, not React state
function updateHighlight(index: number) {
  // Remove previous highlight
  document.querySelector('.active-lyric')?.classList.remove('active-lyric')

  // Add new highlight
  const element = document.querySelector(`[data-lyric-index="${index}"]`)
  element?.classList.add('active-lyric')
  element?.scrollIntoView({ behavior: 'smooth', block: 'center' })
}
```

#### 3. Throttled Updates

```typescript
// Only update UI every 50ms max, even if RAF fires more often
const throttledUpdate = throttle((index: number) => {
  setCurrentIndex(index)
}, 50)
```

### Khmer Text Rendering

```css
/* globals.css */
@import url('https://fonts.googleapis.com/css2?family=Noto+Sans+Khmer:wght@400;600;700&display=swap');

.font-khmer {
  font-family: 'Noto Sans Khmer', sans-serif;
  line-height: 1.8;
  letter-spacing: 0.02em;
}

.lyric-line.font-khmer {
  font-size: 1.5rem;
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
}

/* Active lyric glow effect */
.active-lyric {
  background: linear-gradient(90deg, transparent, rgba(var(--primary), 0.1), transparent);
  text-shadow: 0 0 20px rgba(var(--primary), 0.3);
  transform: scale(1.02);
}
```

### Edge Cases

| Scenario | Handling |
|----------|----------|
| No lyrics at timestamp | Show placeholder "♪" or instrumental indicator |
| Seeking forward/backward | Immediate jump to correct lyric (no animation) |
| Very fast lyrics | Ensure minimum display time of 200ms |
| Empty lyric lines | Display as instrumental break with visual indicator |
| Mixed language songs | Auto-detect and switch font family |

## Alternatives Considered

### 1. Polling with setInterval
- **Pros**: Simple implementation
- **Cons**: Inconsistent timing, battery drain, not synced to display refresh
- **Decision**: Rejected - RAF provides better precision and efficiency

### 2. Media Element timeupdate Event Only
- **Pros**: Native browser event, low overhead
- **Cons**: Only fires ~4 times/second (250ms), too slow for precise sync
- **Decision**: Rejected - need higher precision for smooth transitions

### 3. Pre-computed Animation Keyframes
- **Pros**: No runtime computation
- **Cons**: Can't handle seeking, inflexible
- **Decision**: Rejected - user interaction requires dynamic sync

## Metrics

| Metric | Target |
|--------|--------|
| Sync precision | ±50ms |
| Frame rate | 60 FPS |
| Scroll smoothness | No jank |
| First lyric display | < 100ms |
| Memory usage | < 5MB for 500 lines |

## Related

- Decisions: D011_LyricsFormat
- Implementation: I004_KaraokeEngine
- Features: F001_KaraokeMode
