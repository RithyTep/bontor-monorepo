Nice — below is a **complete, copy-pasteable Karaoke Mode React component** (TypeScript, Next.js 16 client component) that implements:

* LRC → JSON parser
* Lyrics sync + highlighting using `requestAnimationFrame`
* Microphone capture (WebRTC)
* Pitch detection using a simplified **YIN** algorithm
* Real-time pitch visualizer and basic scoring (frame-based accuracy)
* Controls for start/stop, transpose, and sensitivity
* Tailwind-friendly classes (easy to style)

Drop this into `apps/web/components/karaoke/KaraokeMode.tsx` and import it where you need. I also include the supporting types and small helper functions.

---

### ✅ What this component **does**

* Plays a supplied audio `audioUrl`.
* Shows synced lyrics (from `lyricsJson`).
* When user clicks **Start Singing**, it requests mic permission, starts capturing audio, runs pitch detection on the microphone input, and compares detected pitch to a provided reference pitch function (MVP: uses a *simple reference pitch line derived from lyrics timestamps — see notes*).
* Displays a real-time score as percent of tempo frames that were "in-tune".

### ⚠️ Limitations & Notes (important)

* This is **MVP-level** scoring: it assumes a coarse per-timestamp reference pitch or a flat pitch reference array. For production you should:

  * Precompute a reference pitch contour for each song (server-side or offline tool) and store it with the song.
  * Use an `AudioWorklet` for lower latency and better timing on complex devices.
  * Use proper noise handling, vocal onset detection, and more advanced scoring heuristics.
* Mobile browsers (esp. Safari iOS) have restrictions for `getUserMedia` and autoplay. Test on target devices.
* Keep testing with Khmer songs and adjust `toleranceCents` for better UX.

---

## 1) Types — `packages/types/song.ts` (or local)

```ts
export type LyricsLine = { t: number; l: string }; // t in seconds

export interface SongDTO {
  id: string;
  title: string;
  artistId?: string;
  artistName?: string;
  audioUrl: string;
  lrcUrl?: string;
  lyricsJson: LyricsLine[];
  durationSec: number;
  isPublished?: boolean;
}
```

---

## 2) Karaoke component — `KaraokeMode.tsx`

```tsx
'use client'
import React, { useEffect, useRef, useState } from 'react'
import type { SongDTO, LyricsLine } from '@/packages/types/song' // adjust import path

// ------------------- LRC parser (robust enough for MVP) -------------------
export function parseLrc(lrc: string): LyricsLine[] {
  const lines = lrc.split(/\r?\n/)
  const out: LyricsLine[] = []
  const timeRe = /\[(\d+):(\d+)(?:\.(\d+))?\]/g
  for (const rawLine of lines) {
    let match
    // find all timestamps in a line (some LRC have multiple)
    const times: number[] = []
    while ((match = timeRe.exec(rawLine)) !== null) {
      const min = Number(match[1])
      const sec = Number(match[2])
      const ms = match[3] ? Number(match[3].padEnd(3, '0')) : 0
      times.push(min * 60 + sec + ms / 1000)
    }
    // text after timestamps
    const text = rawLine.replace(timeRe, '').trim()
    for (const t of times) {
      out.push({ t, l: text })
    }
  }
  // sort by time
  out.sort((a, b) => a.t - b.t)
  return out
}

// ------------------- Simple YIN pitch detection (MVP) -------------------
// Reference: simplified adaptation for JS, works for common voice ranges
function autoCorrelateYIN(buffer: Float32Array, sampleRate: number) {
  // Short-time YIN-like / autocorrelation simplified
  const size = buffer.length
  const threshold = 0.15
  const yinBuffer = new Float32Array(size / 2)
  let tau, i
  // step 1: difference function
  for (tau = 0; tau < yinBuffer.length; tau++) {
    let sum = 0
    for (i = 0; i < yinBuffer.length; i++) {
      const diff = buffer[i] - buffer[i + tau]
      sum += diff * diff
    }
    yinBuffer[tau] = sum
  }
  // step 2: cumulative mean normalized difference
  let runningSum = 0
  yinBuffer[0] = 1
  for (tau = 1; tau < yinBuffer.length; tau++) {
    runningSum += yinBuffer[tau]
    yinBuffer[tau] = yinBuffer[tau] * tau / runningSum
  }
  // step 3: absolute threshold
  let tauEstimate = -1
  for (tau = 2; tau < yinBuffer.length; tau++) {
    if (yinBuffer[tau] < threshold) {
      // find local minimum
      while (tau + 1 < yinBuffer.length && yinBuffer[tau + 1] < yinBuffer[tau]) {
        tau++
      }
      tauEstimate = tau
      break
    }
  }
  if (tauEstimate === -1) return -1
  // step 4: parabolic interpolation for better accuracy
  const x0 = tauEstimate - 1 >= 0 ? yinBuffer[tauEstimate - 1] : yinBuffer[tauEstimate]
  const x1 = yinBuffer[tauEstimate]
  const x2 = tauEstimate + 1 < yinBuffer.length ? yinBuffer[tauEstimate + 1] : yinBuffer[tauEstimate]
  const a = (x0 + x2 - 2 * x1) / 2
  const b = (x2 - x0) / 2
  const tauRefined = tauEstimate - b / (2 * a || 1)
  const frequency = sampleRate / tauRefined
  if (!isFinite(frequency) || frequency > 5000 || frequency < 50) return -1
  return frequency
}

// convert hz to cents relative to ref hz
function centsDiff(hz: number, ref: number) {
  return 1200 * Math.log2(hz / ref)
}

// ------------------- KaraokeMode component -------------------
type KaraokeModeProps = {
  song: SongDTO
  // optional: precomputed reference pitch function:
  // a function that, given time (s) returns reference frequency in Hz
  referencePitchFn?: (timeSec: number) => number | null
  onScore?: (score: number) => void
}

export default function KaraokeMode({ song, referencePitchFn, onScore }: KaraokeModeProps) {
  const audioRef = useRef<HTMLAudioElement | null>(null)
  const rafRef = useRef<number | null>(null)
  const micStreamRef = useRef<MediaStream | null>(null)
  const analyserRef = useRef<AnalyserNode | null>(null)
  const audioCtxRef = useRef<AudioContext | null>(null)
  const dataBufRef = useRef<Float32Array | null>(null)

  const [lyrics, setLyrics] = useState<LyricsLine[]>(song.lyricsJson ?? [])
  const [currentIndex, setCurrentIndex] = useState(0)
  const [isSinging, setIsSinging] = useState(false)
  const [score, setScore] = useState(0)
  const [detectedFreq, setDetectedFreq] = useState<number | null>(null)
  const [framesTotal, setFramesTotal] = useState(0)
  const [framesCorrect, setFramesCorrect] = useState(0)
  const [transpose, setTranspose] = useState(0) // semitones
  const [toleranceCents, setToleranceCents] = useState(80) // threshold for correctness

  useEffect(() => {
    setLyrics(song.lyricsJson ?? [])
  }, [song.lyricsJson])

  // Cleanup on unmount
  useEffect(() => {
    return () => {
      stopSinging()
      stopRAF()
      if (audioCtxRef.current && audioCtxRef.current.state !== 'closed') {
        audioCtxRef.current.close().catch(() => {})
      }
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [])

  // ----------------- Lyrics sync RAF -----------------
  function rafLoop() {
    const audio = audioRef.current
    if (!audio) {
      rafRef.current = requestAnimationFrame(rafLoop)
      return
    }
    const t = audio.currentTime
    // binary search or linear advance
    let idx = currentIndex
    const n = lyrics.length
    if (n === 0) {
      rafRef.current = requestAnimationFrame(rafLoop)
      return
    }
    // if current time has passed next timestamp, advance
    while (idx + 1 < n && lyrics[idx + 1].t <= t + 0.05) idx++
    // if time is before current index, rewind
    while (idx - 1 >= 0 && lyrics[idx].t > t + 0.05) idx--
    if (idx !== currentIndex) setCurrentIndex(idx)
    rafRef.current = requestAnimationFrame(rafLoop)
  }

  function startRAF() {
    if (rafRef.current == null) {
      rafRef.current = requestAnimationFrame(rafLoop)
    }
  }
  function stopRAF() {
    if (rafRef.current != null) {
      cancelAnimationFrame(rafRef.current)
      rafRef.current = null
    }
  }

  // ----------------- Mic & pitch detection -----------------
  async function startSinging() {
    if (isSinging) return
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ audio: true })
      micStreamRef.current = stream
      const AudioContextCtor = (window.AudioContext || (window as any).webkitAudioContext)
      const audioCtx = new AudioContextCtor()
      audioCtxRef.current = audioCtx
      const source = audioCtx.createMediaStreamSource(stream)
      const analyser = audioCtx.createAnalyser()
      analyser.fftSize = 2048
      analyserRef.current = analyser
      source.connect(analyser)
      const buf = new Float32Array(analyser.fftSize)
      dataBufRef.current = buf

      setFramesTotal(0)
      setFramesCorrect(0)
      setScore(0)
      setIsSinging(true)
      startRAF()
      audioRef.current?.play().catch(() => {})

      // pitch detection loop (uses requestAnimationFrame)
      const sampleRate = audioCtx.sampleRate

      const pitchLoop = () => {
        if (!isSinging || !analyserRef.current || !audioCtxRef.current) return
        const arr = new Float32Array(analyserRef.current.fftSize)
        analyserRef.current.getFloatTimeDomainData(arr)
        const detected = autoCorrelateYIN(arr, sampleRate)
        if (detected > 0) {
          setDetectedFreq(detected)
        } else {
          setDetectedFreq(null)
        }
        // scoring logic: compare detected to reference
        const audioEl = audioRef.current
        if (audioEl) {
          const now = audioEl.currentTime
          // get reference pitch (Hz)
          let refHz: number | null = null
          if (referencePitchFn) {
            refHz = referencePitchFn(now)
          } else {
            // Fallback: use last lyric as approximate pitch by deriving a base freq from lyric line index
            // This is a naive fallback; production must replace with real contour
            // Map lyric index to some plausible pitch (A4 = 440)
            const idx = findLyricIndexByTime(lyrics, now)
            // naive mapping: middle C ~ 261.63, shift by index small amounts
            refHz = 261.63 * Math.pow(2, (idx % 12) / 12)
          }
          if (detected > 0 && refHz) {
            // apply transpose semitones
            const refTransposed = refHz * Math.pow(2, transpose / 12)
            const diffCents = Math.abs(centsDiff(detected, refTransposed))
            setFramesTotal((v) => v + 1)
            if (diffCents <= toleranceCents) {
              setFramesCorrect((v) => v + 1)
            }
          } else {
            // treat silence as frames too; here we still count as total but not correct
            setFramesTotal((v) => v + 1)
          }
        }
        // update score periodically (simple)
        setScore((prev) => {
          const total = Math.max(1, framesTotal || 1)
          const correct = framesCorrect || 0
          const newScore = Math.round((correct / total) * 100)
          if (onScore) onScore(newScore)
          return newScore
        })
        // repeat
        requestAnimationFrame(pitchLoop)
      }
      requestAnimationFrame(pitchLoop)
    } catch (err) {
      console.error('Mic error', err)
      alert('Microphone permission required to sing.')
    }
  }

  function stopSinging() {
    setIsSinging(false)
    if (micStreamRef.current) {
      micStreamRef.current.getTracks().forEach((t) => t.stop())
      micStreamRef.current = null
    }
    if (audioCtxRef.current) {
      audioCtxRef.current.close().catch(() => {})
      audioCtxRef.current = null
    }
    analyserRef.current = null
    dataBufRef.current = null
  }

  // ----------------- Utility: find lyric idx -----------------
  function findLyricIndexByTime(list: LyricsLine[], t: number) {
    if (!list || list.length === 0) return 0
    let lo = 0
    let hi = list.length - 1
    while (lo <= hi) {
      const mid = (lo + hi) >> 1
      if (list[mid].t === t) return mid
      if (list[mid].t < t) lo = mid + 1
      else hi = mid - 1
    }
    return Math.max(0, lo - 1)
  }

  // ----------------- UI Handlers -----------------
  return (
    <div className="w-full max-w-3xl mx-auto p-4">
      <div className="mb-4">
        <h2 className="text-xl font-semibold">{song.title}</h2>
        <p className="text-sm text-muted-foreground">{song.artistName}</p>
      </div>

      <audio
        ref={audioRef}
        src={song.audioUrl}
        controls
        className="w-full mb-4"
        onPlay={() => {
          startRAF()
        }}
        onPause={() => {
          stopRAF()
        }}
      />

      <div className="flex items-center gap-4 mb-3">
        <button
          className={`px-4 py-2 rounded ${isSinging ? 'bg-red-600 text-white' : 'bg-green-600 text-white'}`}
          onClick={() => {
            if (isSinging) stopSinging()
            else startSinging()
          }}
        >
          {isSinging ? 'Stop Singing' : 'Start Singing'}
        </button>

        <div className="flex items-center gap-2">
          <label className="text-sm">Transpose</label>
          <input
            type="range"
            min={-12}
            max={12}
            value={transpose}
            onChange={(e) => setTranspose(Number(e.target.value))}
          />
          <span className="text-sm">{transpose} st</span>
        </div>

        <div className="flex items-center gap-2">
          <label className="text-sm">Tolerance (cents)</label>
          <input
            type="range"
            min={10}
            max={200}
            value={toleranceCents}
            onChange={(e) => setToleranceCents(Number(e.target.value))}
          />
          <span className="text-sm">{toleranceCents}</span>
        </div>
      </div>

      {/* Real-time pitch & score */}
      <div className="mb-4">
        <div className="flex items-center gap-4">
          <div>
            <div className="text-xs text-muted-foreground">Detected Frequency</div>
            <div className="text-lg font-mono">{detectedFreq ? `${detectedFreq.toFixed(1)} Hz` : '—'}</div>
          </div>
          <div>
            <div className="text-xs text-muted-foreground">Score</div>
            <div className="text-lg font-bold">{score}%</div>
          </div>
          <div>
            <div className="text-xs text-muted-foreground">Frames</div>
            <div className="text-sm">{framesCorrect}/{Math.max(1, framesTotal)}</div>
          </div>
        </div>
      </div>

      {/* Lyrics panel with highlight */}
      <div className="bg-surface p-3 rounded shadow-inner max-h-64 overflow-auto">
        <ol className="space-y-2">
          {lyrics.map((ln, i) => {
            const active = i === currentIndex
            return (
              <li
                key={i}
                className={`transition-colors ${active ? 'bg-yellow-200 text-black p-1 rounded' : 'text-gray-700'}`}
              >
                <div className="text-xs text-muted-foreground">{formatTime(ln.t)}</div>
                <div className="whitespace-pre-wrap">{ln.l || '—'}</div>
              </li>
            )
          })}
        </ol>
      </div>
    </div>
  )
}

// small helper to format time
function formatTime(sec: number) {
  const s = Math.floor(sec % 60)
  const m = Math.floor(sec / 60)
  const mm = m.toString().padStart(2, '0')
  const ss = s.toString().padStart(2, '0')
  return `${mm}:${ss}`
}
```

---

## 3) How to use the component (example)

```tsx
// pages/song/[id]/page.tsx or a client wrapper
'use client'
import KaraokeMode from '@/components/karaoke/KaraokeMode'
import type { SongDTO } from '@/packages/types/song'

// fetch song on server-side and pass as prop or fetch in client
const song: SongDTO = {
  id: '1',
  title: 'Example Song',
  audioUrl: 'https://example.com/song.mp3',
  lyricsJson: [
    { t: 0.0, l: 'First line' },
    { t: 5.2, l: 'Second line' },
    { t: 9.8, l: 'Third line' }
  ],
  durationSec: 120
}

export default function SongPage() {
  return (
    <div>
      <KaraokeMode song={song} />
    </div>
  )
}
```

---

## 4) Recommended next improvements (after dropping in)

1. **Reference pitch contour**: Replace the fallback `referencePitchFn` with a real contour: precompute (server-side) an array of `[timeSec, frequencyHz]` and interpolate.
2. **AudioWorklet**: Replace `AnalyserNode` + `requestAnimationFrame` with `AudioWorklet` for deterministic low-latency pitch detection.
3. **Noise reduction & voice activity detection**: Use VAD to ignore silence and reduce false negatives.
4. **Visual polish**: Add waveform, pitch overlay graph, and smoother scoring animations.
5. **Record & upload**: Allow saving user recordings and server-side scoring for verification.

---
