# P001: Audio Processing Pipeline

## Status: Approved

## Context

The karaoke application requires real-time audio processing for pitch detection, voice analysis, and scoring. The pipeline must handle microphone input while simultaneously playing backing tracks, with minimal latency to provide responsive visual feedback.

## Proposal

Implement a client-side audio processing pipeline using the Web Audio API with the following architecture:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     AUDIO PROCESSING PIPELINE                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────────────┐    │
│  │  Microphone  │────▶│ MediaStream  │────▶│   AudioContext       │    │
│  │    Input     │     │   Source     │     │                      │    │
│  └──────────────┘     └──────────────┘     │  ┌────────────────┐  │    │
│                                            │  │  AnalyserNode  │  │    │
│  ┌──────────────┐     ┌──────────────┐     │  │  (FFT Size:    │  │    │
│  │   MP3 Audio  │────▶│  Audio       │────▶│  │   2048)        │  │    │
│  │   Element    │     │  Source      │     │  └───────┬────────┘  │    │
│  └──────────────┘     └──────────────┘     │          │           │    │
│                                            │          ▼           │    │
│                                            │  ┌────────────────┐  │    │
│                                            │  │ ScriptProcessor│  │    │
│                                            │  │ (deprecated)   │  │    │
│                                            │  │      OR        │  │    │
│                                            │  │ AudioWorklet   │  │    │
│                                            │  │ (preferred)    │  │    │
│                                            │  └───────┬────────┘  │    │
│                                            └──────────┼───────────┘    │
│                                                       │                │
│                                                       ▼                │
│                                            ┌──────────────────┐        │
│                                            │  Pitch Detector  │        │
│                                            │  (YIN Algorithm) │        │
│                                            └───────┬──────────┘        │
│                                                    │                   │
│                                                    ▼                   │
│                                            ┌──────────────────┐        │
│                                            │ Score Calculator │        │
│                                            └──────────────────┘        │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

## Technical Specifications

### Audio Context Configuration

```typescript
const audioContext = new AudioContext({
  sampleRate: 44100,
  latencyHint: 'interactive'
})
```

### Pipeline Stages

#### Stage 1: Input Capture
- **Microphone**: MediaDevices.getUserMedia() with constraints
- **Backing Track**: HTMLAudioElement or fetch() + decodeAudioData()

```typescript
const micConstraints: MediaStreamConstraints = {
  audio: {
    echoCancellation: true,
    noiseSuppression: true,
    autoGainControl: true,
    sampleRate: 44100,
    channelCount: 1
  }
}
```

#### Stage 2: Analysis
- **FFT Analysis**: AnalyserNode with fftSize: 2048
- **Time Domain**: Float32Array for pitch detection
- **Frequency Domain**: Uint8Array for visualization

#### Stage 3: Pitch Detection
- **Algorithm**: YIN autocorrelation
- **Sample Rate**: 44100 Hz
- **Buffer Size**: 2048 samples (~46ms window)
- **Threshold**: 0.15 (configurable)

#### Stage 4: Scoring
- **Frame Rate**: ~60 FPS (requestAnimationFrame)
- **Tolerance**: ±80 cents (configurable)
- **Smoothing**: 5-frame rolling average

### AudioWorklet Implementation

```typescript
// pitch-processor.worklet.ts
class PitchProcessor extends AudioWorkletProcessor {
  process(inputs: Float32Array[][], outputs: Float32Array[][], parameters: Record<string, Float32Array>) {
    const input = inputs[0]?.[0]
    if (!input) return true

    // Run YIN algorithm
    const frequency = this.detectPitch(input)

    // Send to main thread
    this.port.postMessage({ frequency, timestamp: currentTime })

    return true
  }
}

registerProcessor('pitch-processor', PitchProcessor)
```

### Latency Budget

| Stage | Target Latency |
|-------|---------------|
| Mic capture | < 10ms |
| Audio analysis | < 15ms |
| Pitch detection | < 10ms |
| UI update | < 16ms |
| **Total** | **< 51ms** |

## Alternatives Considered

### 1. Server-Side Processing
- **Pros**: More powerful algorithms, consistent across devices
- **Cons**: Network latency (50-200ms), server costs, scaling issues
- **Decision**: Rejected - latency too high for real-time feedback

### 2. WebAssembly (WASM)
- **Pros**: Near-native performance, consistent cross-browser
- **Cons**: Added complexity, bundle size increase
- **Decision**: Deferred - JavaScript sufficient for MVP, can optimize later

### 3. Web Workers
- **Pros**: Off-main-thread processing
- **Cons**: Limited Audio API access, message passing overhead
- **Decision**: Rejected - AudioWorklet is the correct abstraction

## Migration Path

### Phase 1: MVP (ScriptProcessor)
- Use deprecated ScriptProcessorNode for broad compatibility
- Works in all browsers including older Safari

### Phase 2: Modern Browsers (AudioWorklet)
- Feature detect AudioWorklet support
- Progressive enhancement for supported browsers

### Phase 3: WASM Optimization
- Compile YIN algorithm to WebAssembly
- Further reduce latency and CPU usage

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Browser audio permissions denied | High | Clear permission prompts, fallback UI |
| High CPU usage on low-end devices | Medium | Adaptive quality settings, frame throttling |
| AudioWorklet not supported | Medium | ScriptProcessor fallback |
| Echo feedback loop | High | Echo cancellation, headphone detection |

## Success Metrics

- Pitch detection accuracy: > 95% within ±50 cents
- Processing latency: < 50ms end-to-end
- CPU usage: < 30% on mid-range devices
- Browser compatibility: Chrome, Firefox, Safari, Edge

## Related

- Decisions: D001_KaraokeEngine
- Implementation: I004_KaraokeEngine
- Features: F001_KaraokeMode, F004_Scoring
