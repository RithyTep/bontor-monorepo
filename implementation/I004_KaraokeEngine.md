# I004: Karaoke Engine Implementation

## Overview

Implementation notes for the karaoke engine package, covering pitch detection, lyrics synchronization, score calculation, and audio recording.

## Related Decisions

- D001: Karaoke Engine Architecture (Client-side)
- D008: Pitch Detection Algorithm (YIN)

## Package Structure

```
packages/karaoke-engine/
├── src/
│   ├── index.ts              # Main exports
│   ├── karaoke-engine.ts     # Main orchestrator class
│   ├── pitch-detector.ts     # YIN pitch detection
│   ├── lyrics-sync.ts        # Lyrics synchronization
│   ├── score-calculator.ts   # Scoring logic
│   ├── audio-recorder.ts     # MediaRecorder wrapper
│   └── types.ts              # TypeScript types
├── tests/
│   ├── pitch-detector.test.ts
│   ├── lyrics-sync.test.ts
│   └── score-calculator.test.ts
└── package.json
```

## Main Engine Class

```typescript
// packages/karaoke-engine/src/karaoke-engine.ts
import { PitchDetector } from './pitch-detector'
import { LyricsSync } from './lyrics-sync'
import { ScoreCalculator } from './score-calculator'
import { AudioRecorder } from './audio-recorder'
import type { LyricsLine, KaraokeConfig, KaraokeState } from './types'

export class KaraokeEngine {
  private audioElement: HTMLAudioElement
  private audioContext: AudioContext | null = null
  private analyser: AnalyserNode | null = null
  private micStream: MediaStream | null = null

  private pitchDetector: PitchDetector
  private lyricsSync: LyricsSync
  private scoreCalculator: ScoreCalculator
  private audioRecorder: AudioRecorder | null = null

  private rafId: number | null = null
  private isRunning: boolean = false

  private config: KaraokeConfig
  private onStateChange: (state: KaraokeState) => void

  constructor(config: KaraokeConfig) {
    this.config = config
    this.audioElement = config.audioElement
    this.onStateChange = config.onStateChange

    this.pitchDetector = new PitchDetector(config.sampleRate || 44100)
    this.lyricsSync = new LyricsSync(config.lyrics, (index) => {
      this.emitState({ currentLyricIndex: index })
    })
    this.scoreCalculator = new ScoreCalculator(config.toleranceCents || 80)

    if (config.enableRecording) {
      this.audioRecorder = new AudioRecorder()
    }
  }

  async start(): Promise<void> {
    if (this.isRunning) return

    try {
      // Request microphone access
      this.micStream = await navigator.mediaDevices.getUserMedia({
        audio: {
          echoCancellation: true,
          noiseSuppression: true,
          autoGainControl: true,
        },
      })

      // Set up audio context
      const AudioContextClass = window.AudioContext || (window as any).webkitAudioContext
      this.audioContext = new AudioContextClass()

      // Connect mic to analyser
      const source = this.audioContext.createMediaStreamSource(this.micStream)
      this.analyser = this.audioContext.createAnalyser()
      this.analyser.fftSize = 2048
      source.connect(this.analyser)

      // Update pitch detector sample rate
      this.pitchDetector = new PitchDetector(this.audioContext.sampleRate)

      // Start recording if enabled
      if (this.audioRecorder) {
        await this.audioRecorder.start(this.micStream)
      }

      // Start processing loop
      this.isRunning = true
      this.lyricsSync.start(this.audioElement)
      this.processLoop()

      // Play audio
      this.audioElement.play()

      this.emitState({ isRunning: true })
    } catch (error) {
      this.emitState({ error: 'Failed to access microphone' })
      throw error
    }
  }

  stop(): void {
    this.isRunning = false

    if (this.rafId) {
      cancelAnimationFrame(this.rafId)
      this.rafId = null
    }

    this.lyricsSync.stop()
    this.audioElement.pause()

    // Stop mic stream
    if (this.micStream) {
      this.micStream.getTracks().forEach((track) => track.stop())
      this.micStream = null
    }

    // Close audio context
    if (this.audioContext) {
      this.audioContext.close()
      this.audioContext = null
    }

    // Stop recording
    if (this.audioRecorder) {
      this.audioRecorder.stop()
    }

    this.emitState({
      isRunning: false,
      finalScore: this.scoreCalculator.getBreakdown(),
    })
  }

  setTranspose(semitones: number): void {
    this.config.transposeSemitones = semitones
  }

  setTolerance(cents: number): void {
    this.scoreCalculator = new ScoreCalculator(cents)
  }

  getRecording(): Blob | null {
    return this.audioRecorder?.getBlob() ?? null
  }

  private processLoop(): void {
    if (!this.isRunning || !this.analyser || !this.audioContext) return

    // Get audio data
    const buffer = new Float32Array(this.analyser.fftSize)
    this.analyser.getFloatTimeDomainData(buffer)

    // Detect pitch
    const detectedHz = this.pitchDetector.detect(buffer)

    // Get reference pitch for current time
    const currentTime = this.audioElement.currentTime
    const referenceHz = this.getReferencePitch(currentTime)

    // Apply transpose
    const transposedRef = referenceHz
      ? referenceHz * Math.pow(2, (this.config.transposeSemitones || 0) / 12)
      : null

    // Update score
    this.scoreCalculator.addFrame(detectedHz, transposedRef)

    // Emit state
    this.emitState({
      detectedFrequency: detectedHz,
      currentScore: this.scoreCalculator.getScore(),
    })

    // Continue loop
    this.rafId = requestAnimationFrame(() => this.processLoop())
  }

  private getReferencePitch(time: number): number | null {
    if (!this.config.referencePitch) return null

    // Binary search for closest reference pitch
    const pitchData = this.config.referencePitch
    let lo = 0
    let hi = pitchData.length - 1

    while (lo <= hi) {
      const mid = Math.floor((lo + hi) / 2)
      if (pitchData[mid].t === time) return pitchData[mid].hz
      if (pitchData[mid].t < time) lo = mid + 1
      else hi = mid - 1
    }

    // Return closest
    if (lo === 0) return pitchData[0].hz
    if (lo >= pitchData.length) return pitchData[pitchData.length - 1].hz

    // Interpolate
    const prev = pitchData[lo - 1]
    const next = pitchData[lo]
    const ratio = (time - prev.t) / (next.t - prev.t)
    return prev.hz + (next.hz - prev.hz) * ratio
  }

  private emitState(partial: Partial<KaraokeState>): void {
    this.onStateChange(partial as KaraokeState)
  }
}
```

## YIN Pitch Detector

```typescript
// packages/karaoke-engine/src/pitch-detector.ts

export class PitchDetector {
  private sampleRate: number
  private threshold: number = 0.15
  private minFreq: number = 50   // Hz
  private maxFreq: number = 2000 // Hz

  constructor(sampleRate: number) {
    this.sampleRate = sampleRate
  }

  detect(buffer: Float32Array): number | null {
    // Check for silence
    if (this.isSilent(buffer)) return null

    // Apply YIN algorithm
    const yinBuffer = this.calculateYIN(buffer)
    const tau = this.findTau(yinBuffer)

    if (tau === -1) return null

    // Parabolic interpolation for better accuracy
    const refinedTau = this.parabolicInterpolation(yinBuffer, tau)
    const frequency = this.sampleRate / refinedTau

    // Filter unrealistic frequencies
    if (frequency < this.minFreq || frequency > this.maxFreq) {
      return null
    }

    return frequency
  }

  private isSilent(buffer: Float32Array): boolean {
    let sum = 0
    for (let i = 0; i < buffer.length; i++) {
      sum += buffer[i] * buffer[i]
    }
    const rms = Math.sqrt(sum / buffer.length)
    return rms < 0.01 // Silence threshold
  }

  private calculateYIN(buffer: Float32Array): Float32Array {
    const halfSize = Math.floor(buffer.length / 2)
    const yinBuffer = new Float32Array(halfSize)

    // Step 1: Difference function
    for (let tau = 0; tau < halfSize; tau++) {
      let sum = 0
      for (let i = 0; i < halfSize; i++) {
        const diff = buffer[i] - buffer[i + tau]
        sum += diff * diff
      }
      yinBuffer[tau] = sum
    }

    // Step 2: Cumulative mean normalized difference
    yinBuffer[0] = 1
    let runningSum = 0
    for (let tau = 1; tau < halfSize; tau++) {
      runningSum += yinBuffer[tau]
      yinBuffer[tau] = yinBuffer[tau] * tau / runningSum
    }

    return yinBuffer
  }

  private findTau(yinBuffer: Float32Array): number {
    // Step 3: Absolute threshold
    for (let tau = 2; tau < yinBuffer.length; tau++) {
      if (yinBuffer[tau] < this.threshold) {
        // Find local minimum
        while (
          tau + 1 < yinBuffer.length &&
          yinBuffer[tau + 1] < yinBuffer[tau]
        ) {
          tau++
        }
        return tau
      }
    }
    return -1
  }

  private parabolicInterpolation(yinBuffer: Float32Array, tau: number): number {
    const x0 = tau > 0 ? yinBuffer[tau - 1] : yinBuffer[tau]
    const x1 = yinBuffer[tau]
    const x2 = tau + 1 < yinBuffer.length ? yinBuffer[tau + 1] : yinBuffer[tau]

    const a = (x0 + x2 - 2 * x1) / 2
    const b = (x2 - x0) / 2

    if (a === 0) return tau
    return tau - b / (2 * a)
  }
}
```

## Lyrics Synchronization

```typescript
// packages/karaoke-engine/src/lyrics-sync.ts
import type { LyricsLine } from './types'

export class LyricsSync {
  private lyrics: LyricsLine[]
  private currentIndex: number = 0
  private rafId: number | null = null
  private onIndexChange: (index: number) => void

  constructor(lyrics: LyricsLine[], onIndexChange: (index: number) => void) {
    this.lyrics = lyrics
    this.onIndexChange = onIndexChange
  }

  start(audioElement: HTMLAudioElement): void {
    const loop = () => {
      this.update(audioElement.currentTime)
      this.rafId = requestAnimationFrame(loop)
    }
    this.rafId = requestAnimationFrame(loop)
  }

  stop(): void {
    if (this.rafId !== null) {
      cancelAnimationFrame(this.rafId)
      this.rafId = null
    }
  }

  seek(time: number): void {
    this.currentIndex = this.binarySearch(time)
    this.onIndexChange(this.currentIndex)
  }

  private update(currentTime: number): void {
    const n = this.lyrics.length
    if (n === 0) return

    const lookAhead = 0.05 // 50ms look-ahead for smoother transitions

    // Advance if next lyric should be shown
    while (
      this.currentIndex + 1 < n &&
      this.lyrics[this.currentIndex + 1].t <= currentTime + lookAhead
    ) {
      this.currentIndex++
      this.onIndexChange(this.currentIndex)
    }

    // Rewind if seeked backwards
    while (
      this.currentIndex > 0 &&
      this.lyrics[this.currentIndex].t > currentTime + lookAhead
    ) {
      this.currentIndex--
      this.onIndexChange(this.currentIndex)
    }
  }

  private binarySearch(time: number): number {
    let lo = 0
    let hi = this.lyrics.length - 1

    while (lo <= hi) {
      const mid = Math.floor((lo + hi) / 2)
      if (this.lyrics[mid].t === time) return mid
      if (this.lyrics[mid].t < time) lo = mid + 1
      else hi = mid - 1
    }

    return Math.max(0, lo - 1)
  }
}
```

## Score Calculator

```typescript
// packages/karaoke-engine/src/score-calculator.ts
import type { ScoreBreakdown } from './types'

export class ScoreCalculator {
  private correctFrames: number = 0
  private totalFrames: number = 0
  private framesWithPitch: number = 0
  private toleranceCents: number

  constructor(toleranceCents: number = 80) {
    this.toleranceCents = toleranceCents
  }

  addFrame(detectedHz: number | null, referenceHz: number | null): void {
    this.totalFrames++

    if (detectedHz !== null) {
      this.framesWithPitch++
    }

    if (detectedHz === null || referenceHz === null) {
      return
    }

    const centsDiff = Math.abs(this.hzToCents(detectedHz, referenceHz))
    if (centsDiff <= this.toleranceCents) {
      this.correctFrames++
    }
  }

  getScore(): number {
    if (this.totalFrames === 0) return 0
    return Math.round((this.correctFrames / this.totalFrames) * 100)
  }

  getBreakdown(): ScoreBreakdown {
    const total = this.totalFrames || 1

    return {
      totalScore: this.getScore(),
      pitchAccuracy: this.correctFrames / total,
      durationCoverage: this.framesWithPitch / total,
      correctFrames: this.correctFrames,
      totalFrames: this.totalFrames,
    }
  }

  reset(): void {
    this.correctFrames = 0
    this.totalFrames = 0
    this.framesWithPitch = 0
  }

  private hzToCents(hz: number, refHz: number): number {
    return 1200 * Math.log2(hz / refHz)
  }
}
```

## React Hook Integration

```typescript
// apps/web/hooks/useKaraoke.ts
'use client'

import { useRef, useState, useCallback, useEffect } from 'react'
import { KaraokeEngine } from '@karaoke/karaoke-engine'
import type { SongDTO } from '@karaoke/types'

interface UseKaraokeOptions {
  song: SongDTO
  onScoreUpdate?: (score: number) => void
  onComplete?: () => void
}

export function useKaraoke({ song, onScoreUpdate, onComplete }: UseKaraokeOptions) {
  const audioRef = useRef<HTMLAudioElement | null>(null)
  const engineRef = useRef<KaraokeEngine | null>(null)

  const [isRunning, setIsRunning] = useState(false)
  const [currentLyricIndex, setCurrentLyricIndex] = useState(0)
  const [detectedFrequency, setDetectedFrequency] = useState<number | null>(null)
  const [currentScore, setCurrentScore] = useState(0)
  const [finalScore, setFinalScore] = useState<ScoreBreakdown | null>(null)
  const [transpose, setTranspose] = useState(0)
  const [tolerance, setTolerance] = useState(80)
  const [error, setError] = useState<string | null>(null)

  const start = useCallback(async () => {
    if (!audioRef.current) return

    try {
      engineRef.current = new KaraokeEngine({
        audioElement: audioRef.current,
        lyrics: song.lyricsJson,
        referencePitch: song.pitchData,
        toleranceCents: tolerance,
        transposeSemitones: transpose,
        onStateChange: (state) => {
          if (state.isRunning !== undefined) setIsRunning(state.isRunning)
          if (state.currentLyricIndex !== undefined) setCurrentLyricIndex(state.currentLyricIndex)
          if (state.detectedFrequency !== undefined) setDetectedFrequency(state.detectedFrequency)
          if (state.currentScore !== undefined) {
            setCurrentScore(state.currentScore)
            onScoreUpdate?.(state.currentScore)
          }
          if (state.finalScore) {
            setFinalScore(state.finalScore)
            onComplete?.()
          }
          if (state.error) setError(state.error)
        },
      })

      await engineRef.current.start()
    } catch (err) {
      setError('Failed to start karaoke session')
    }
  }, [song, tolerance, transpose, onScoreUpdate, onComplete])

  const stop = useCallback(() => {
    engineRef.current?.stop()
    engineRef.current = null
  }, [])

  const updateTranspose = useCallback((semitones: number) => {
    setTranspose(semitones)
    engineRef.current?.setTranspose(semitones)
  }, [])

  const updateTolerance = useCallback((cents: number) => {
    setTolerance(cents)
    engineRef.current?.setTolerance(cents)
  }, [])

  // Cleanup on unmount
  useEffect(() => {
    return () => {
      engineRef.current?.stop()
    }
  }, [])

  return {
    audioRef,
    isRunning,
    currentLyricIndex,
    detectedFrequency,
    currentScore,
    finalScore,
    transpose,
    tolerance,
    error,
    start,
    stop,
    updateTranspose,
    updateTolerance,
  }
}
```

## Types

```typescript
// packages/karaoke-engine/src/types.ts

export interface LyricsLine {
  t: number  // Time in seconds
  l: string  // Lyric text
}

export interface PitchPoint {
  t: number  // Time in seconds
  hz: number // Frequency in Hz
}

export interface KaraokeConfig {
  audioElement: HTMLAudioElement
  lyrics: LyricsLine[]
  referencePitch?: PitchPoint[]
  toleranceCents?: number
  transposeSemitones?: number
  sampleRate?: number
  enableRecording?: boolean
  onStateChange: (state: Partial<KaraokeState>) => void
}

export interface KaraokeState {
  isRunning: boolean
  currentLyricIndex: number
  detectedFrequency: number | null
  currentScore: number
  finalScore?: ScoreBreakdown
  error?: string
}

export interface ScoreBreakdown {
  totalScore: number
  pitchAccuracy: number
  durationCoverage: number
  correctFrames: number
  totalFrames: number
}
```

## Checklist

- [ ] Implement KaraokeEngine main class
- [ ] Implement YIN pitch detector
- [ ] Implement lyrics synchronization
- [ ] Implement score calculator
- [ ] Implement audio recorder
- [ ] Create useKaraoke React hook
- [ ] Write unit tests for pitch detector
- [ ] Write unit tests for score calculator
- [ ] Test on various devices and browsers
- [ ] Profile and optimize performance
