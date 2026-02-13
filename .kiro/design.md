# Design Document: Industrial Noise Monitor

## Overview

The Industrial Noise Monitor is a mobile application that provides real-time occupational noise hazard detection using continuous audio processing and signal analysis. The system architecture follows a pipeline pattern where audio data flows through capture, analysis, and alerting stages with minimal latency to ensure worker safety.

The application is designed to run on Android smartphones using Kotlin and native Android audio APIs without requiring specialized hardware. It uses the device's built-in microphone to capture environmental audio, processes it in discrete frames using A-weighted decibel calculations, and triggers immediate visual and optional haptic alerts when noise levels exceed the 85 dB occupational safety threshold.

Key design principles:
- **Real-time processing**: Sub-500ms latency from hazard detection to user notification
- **Privacy-first**: All audio processing occurs in-memory with no recording or transmission
- **Resource efficiency**: Minimal CPU and battery impact to enable continuous monitoring
- **Cross-platform**: Shared core logic with platform-specific audio and UI implementations

## Architecture

The system follows a layered architecture with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────┐
│                    Presentation Layer                    │
│  ┌──────────────────┐         ┌──────────────────┐     │
│  │  Alert UI        │         │  Settings UI     │     │
│  │  (Visual/Haptic) │         │  (Configuration) │     │
│  └──────────────────┘         └──────────────────┘     │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│                     Business Logic Layer                 │
│  ┌──────────────────┐         ┌──────────────────┐     │
│  │  Noise Analyzer  │         │  Alert Manager   │     │
│  │  (dB Calculation)│────────▶│  (Notification)  │     │
│  └──────────────────┘         └──────────────────┘     │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│                    Audio Capture Layer                   │
│  ┌──────────────────┐         ┌──────────────────┐     │
│  │  Audio Processor │         │  Frame Buffer    │     │
│  │  (Microphone)    │────────▶│  (Segmentation)  │     │
│  └──────────────────┘         └──────────────────┘     │
└─────────────────────────────────────────────────────────┘
```

### Data Flow

1. **Audio Capture**: Audio Processor continuously captures audio from the device microphone at 44.1 kHz sample rate
2. **Frame Segmentation**: Raw audio is buffered and segmented into 100-200ms frames
3. **Signal Analysis**: Noise Analyzer calculates A-weighted dB level for each frame
4. **Hazard Detection**: Analyzer compares dB level against 85 dB threshold
5. **Alert Dispatch**: Alert Manager triggers visual and optional haptic notifications when threshold is exceeded
6. **UI Update**: Presentation layer displays current dB level and alert status in real-time

## Components and Interfaces

### Audio Processor

**Responsibility**: Manages microphone access and continuous audio capture

**Interface**:
```typescript
interface AudioProcessor {
  // Request microphone permissions from the OS
  requestPermissions(): Promise<PermissionStatus>
  
  // Start continuous audio capture
  startCapture(config: AudioConfig): Result<void, AudioError>
  
  // Stop audio capture and release resources
  stopCapture(): void
  
  // Register callback for audio frame data
  onFrameReady(callback: (frame: AudioFrame) => void): void
  
  // Get current capture status
  getStatus(): CaptureStatus
}

interface AudioConfig {
  sampleRate: number        // 44100 Hz
  frameDurationMs: number   // 100-200 ms
  channelCount: number      // 1 (mono)
}

interface AudioFrame {
  samples: Float32Array     // Raw PCM audio samples
  sampleRate: number        // Sample rate in Hz
  timestamp: number         // Capture timestamp in ms
}

enum CaptureStatus {
  IDLE,
  CAPTURING,
  ERROR,
  PERMISSION_DENIED
}
```

**Platform-Specific Implementation**:
- **Android**: Uses AudioRecord API with VOICE_RECOGNITION audio source

**Error Handling**:
- Retry capture initialization up to 3 times with 2-second delays
- Notify Alert Manager if capture fails after retries
- Handle microphone conflicts with other applications gracefully

### Noise Analyzer

**Responsibility**: Performs frame-based signal analysis to calculate A-weighted decibel levels

**Interface**:
```typescript
interface NoiseAnalyzer {
  // Analyze audio frame and calculate dB level
  analyzeFrame(frame: AudioFrame): AnalysisResult
  
  // Check if dB level exceeds safety threshold
  isHazardous(dbLevel: number): boolean
  
  // Get current safety threshold
  getThreshold(): number
}

interface AnalysisResult {
  dbLevel: number           // A-weighted dB SPL (1 decimal precision)
  isHazardous: boolean      // True if >= 85 dB
  timestamp: number         // Analysis timestamp in ms
  processingTimeMs: number  // Time taken for analysis
}
```

**Signal Processing Algorithm**:

1. **RMS Calculation**: Compute Root Mean Square of audio samples
   ```
   RMS = sqrt(sum(sample[i]^2) / N)
   ```

2. **A-Weighting**: Apply A-weighting frequency response curve
   - A-weighting emphasizes frequencies most damaging to human hearing (1-6 kHz)
   - Implementation uses pre-computed A-weighting filter coefficients
   - Filter applied in frequency domain using FFT

3. **dB SPL Conversion**: Convert RMS to decibels relative to reference pressure
   ```
   dB = 20 * log10(RMS / reference_pressure)
   reference_pressure = 20 micropascals (0 dB SPL threshold of hearing)
   ```

4. **Calibration**: Apply device-specific calibration offset
   - Each device microphone has different sensitivity
   - Calibration offset determined through testing with reference sound source
   - Default offset: 0 dB (can be adjusted in settings)

**Performance Requirements**:
- Frame analysis must complete within 100ms
- Use efficient FFT implementation (e.g., FFTW, vDSP on iOS)
- Minimize memory allocations during processing

### Alert Manager

**Responsibility**: Manages visual and haptic notifications based on noise analysis results

**Interface**:
```typescript
interface AlertManager {
  // Process analysis result and trigger alerts if needed
  processAnalysis(result: AnalysisResult): void
  
  // Update alert configuration
  updateConfig(config: AlertConfig): void
  
  // Get current alert state
  getAlertState(): AlertState
  
  // Clear all active alerts
  clearAlerts(): void
}

interface AlertConfig {
  vibrationEnabled: boolean
  visualAlertEnabled: boolean
  vibrationPattern: number[]  // [duration_ms, pause_ms, ...]
  repeatIntervalMs: number    // 5000 ms for continuous hazards
}

interface AlertState {
  isActive: boolean
  currentDbLevel: number
  alertStartTime: number | null
  lastVibrationTime: number | null
}
```

**Alert Logic**:

1. **Hazard Detection**:
   - When `result.isHazardous === true`, activate alert state
   - Record alert start time for duration tracking

2. **Visual Alert**:
   - Display high-contrast warning (red background, white text)
   - Show current dB level with 1 decimal precision
   - Update display in real-time as new results arrive
   - Clear alert when dB level drops below 85 dB for 1 second

3. **Haptic Alert** (if enabled):
   - Trigger vibration pattern: [200ms ON, 100ms OFF, 200ms ON, 100ms OFF, 200ms ON]
   - For continuous hazards (>5 seconds), repeat every 5 seconds
   - Track last vibration time to prevent excessive vibration

4. **Background Alerts**:
   - Use system notification API when app is backgrounded
   - Notification shows current dB level and hazard warning
   - Tapping notification brings app to foreground

### Settings Manager

**Responsibility**: Manages user preferences and application configuration

**Interface**:
```typescript
interface SettingsManager {
  // Get current settings
  getSettings(): AppSettings
  
  // Update specific setting
  updateSetting(key: string, value: any): void
  
  // Reset to default settings
  resetToDefaults(): void
  
  // Register callback for settings changes
  onSettingsChanged(callback: (settings: AppSettings) => void): void
}

interface AppSettings {
  vibrationEnabled: boolean
  backgroundMonitoringEnabled: boolean
  calibrationOffset: number      // dB offset for device calibration
  frameDurationMs: number         // 100-200 ms
  safetyThreshold: number         // 85 dB (configurable for different standards)
}
```

**Persistence**:
- Settings stored in platform-specific secure storage
- iOS: UserDefaults
- Android: SharedPreferences
- Settings loaded on app startup and cached in memory

## Data Models

### AudioFrame

Represents a discrete segment of captured audio data.

```typescript
class AudioFrame {
  samples: Float32Array      // PCM audio samples (normalized -1.0 to 1.0)
  sampleRate: number         // Sample rate in Hz (44100)
  timestamp: number          // Capture timestamp (milliseconds since epoch)
  durationMs: number         // Frame duration in milliseconds
  
  constructor(samples: Float32Array, sampleRate: number, timestamp: number) {
    this.samples = samples
    this.sampleRate = sampleRate
    this.timestamp = timestamp
    this.durationMs = (samples.length / sampleRate) * 1000
  }
  
  // Get number of samples in frame
  getSampleCount(): number {
    return this.samples.length
  }
  
  // Validate frame data
  isValid(): boolean {
    return this.samples.length > 0 && 
           this.sampleRate > 0 && 
           this.timestamp > 0
  }
}
```

### AnalysisResult

Represents the output of noise analysis for a single audio frame.

```typescript
class AnalysisResult {
  dbLevel: number              // A-weighted dB SPL (1 decimal precision)
  isHazardous: boolean         // True if >= threshold
  timestamp: number            // Analysis timestamp
  processingTimeMs: number     // Analysis duration
  frameId: string              // Unique frame identifier
  
  constructor(
    dbLevel: number,
    threshold: number,
    timestamp: number,
    processingTimeMs: number
  ) {
    this.dbLevel = Math.round(dbLevel * 10) / 10  // Round to 1 decimal
    this.isHazardous = dbLevel >= threshold
    this.timestamp = timestamp
    this.processingTimeMs = processingTimeMs
    this.frameId = `${timestamp}-${Math.random()}`
  }
  
  // Format dB level for display
  formatDbLevel(): string {
    return `${this.dbLevel.toFixed(1)} dB`
  }
}
```

### AlertState

Represents the current state of the alert system.

```typescript
class AlertState {
  isActive: boolean
  currentDbLevel: number
  alertStartTime: number | null
  lastVibrationTime: number | null
  consecutiveHazardFrames: number
  
  constructor() {
    this.isActive = false
    this.currentDbLevel = 0
    this.alertStartTime = null
    this.lastVibrationTime = null
    this.consecutiveHazardFrames = 0
  }
  
  // Activate alert
  activate(dbLevel: number, timestamp: number): void {
    if (!this.isActive) {
      this.alertStartTime = timestamp
    }
    this.isActive = true
    this.currentDbLevel = dbLevel
    this.consecutiveHazardFrames++
  }
  
  // Deactivate alert
  deactivate(): void {
    this.isActive = false
    this.alertStartTime = null
    this.lastVibrationTime = null
    this.consecutiveHazardFrames = 0
  }
  
  // Check if vibration should be triggered
  shouldVibrate(currentTime: number, repeatIntervalMs: number): boolean {
    if (!this.isActive) return false
    if (this.lastVibrationTime === null) return true
    return (currentTime - this.lastVibrationTime) >= repeatIntervalMs
  }
  
  // Get alert duration in seconds
  getAlertDuration(currentTime: number): number {
    if (this.alertStartTime === null) return 0
    return (currentTime - this.alertStartTime) / 1000
  }
}
```

### AppSettings

Represents user preferences and configuration.

```typescript
class AppSettings {
  vibrationEnabled: boolean
  backgroundMonitoringEnabled: boolean
  calibrationOffset: number
  frameDurationMs: number
  safetyThreshold: number
  
  // Default settings
  static DEFAULT: AppSettings = {
    vibrationEnabled: true,
    backgroundMonitoringEnabled: true,
    calibrationOffset: 0,
    frameDurationMs: 150,
    safetyThreshold: 85
  }
  
  constructor(partial?: Partial<AppSettings>) {
    Object.assign(this, AppSettings.DEFAULT, partial)
  }
  
  // Validate settings
  validate(): boolean {
    return this.frameDurationMs >= 100 && 
           this.frameDurationMs <= 200 &&
           this.safetyThreshold >= 70 &&
           this.safetyThreshold <= 120 &&
           this.calibrationOffset >= -20 &&
           this.calibrationOffset <= 20
  }
}
```


## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

The following properties define the correctness criteria for the Industrial Noise Monitor. Each property is universally quantified and references the specific requirements it validates.

### Audio Capture Properties

**Property 1: Permission-gated capture**
*For any* application start, when microphone permissions are granted, audio capture should begin automatically.
**Validates: Requirements 1.2**

**Property 2: Audio configuration invariants**
*For any* captured audio frame, the sample rate should be at least 44.1 kHz and the frame duration should be between 100-200 milliseconds.
**Validates: Requirements 1.4, 2.1**

**Property 3: Capture error recovery**
*For any* audio capture failure, the system should attempt to restart capture and notify the alert system if recovery fails.
**Validates: Requirements 1.5**

**Property 4: Capture continuity**
*For any* sequence of captured audio frames during normal operation, there should be no gaps in timestamps exceeding the frame duration.
**Validates: Requirements 6.5**

### Signal Analysis Properties

**Property 5: Decibel calculation correctness**
*For any* audio frame, the calculated dB level should use A-weighting, reference pressure of 20 micropascals, and be formatted with exactly one decimal place precision.
**Validates: Requirements 2.2, 2.3, 2.4, 2.5**

**Property 6: Hazard classification correctness**
*For any* calculated dB level, it should be classified as hazardous if and only if it is greater than or equal to 85 dB.
**Validates: Requirements 3.2, 3.4**

**Property 7: Hazard notification**
*For any* audio frame classified as hazardous, the Alert System should be notified immediately.
**Validates: Requirements 3.3**

**Property 8: Stateless frame processing**
*For any* set of audio frames, processing them in different orders should produce identical dB results for each individual frame.
**Validates: Requirements 3.5**

**Property 9: Analysis performance**
*For any* audio frame, analysis should complete within 100 milliseconds.
**Validates: Requirements 6.1**

### Alert System Properties

**Property 10: Alert timing**
*For any* hazardous noise detection, a visual warning should be displayed within 500 milliseconds.
**Validates: Requirements 4.1, 6.2**

**Property 11: Alert visual requirements**
*For any* displayed visual warning, it should use high-contrast colors (red background with white text) and show the current decibel level.
**Validates: Requirements 4.2, 4.3**

**Property 12: Alert clearing timing**
*For any* transition from hazardous to safe noise levels, the visual warning should clear within 1 second.
**Validates: Requirements 4.4**

**Property 13: Real-time alert updates**
*For any* active visual warning, the displayed decibel level should update to reflect each new analysis result.
**Validates: Requirements 4.5**

**Property 14: Conditional vibration activation**
*For any* hazardous noise detection, device vibration should be triggered if and only if vibration notifications are enabled in settings.
**Validates: Requirements 5.1, 5.3**

**Property 15: Vibration pattern consistency**
*For any* triggered vibration when vibration is enabled, the pattern should consist of 3 short pulses (200ms ON, 100ms OFF, repeated 3 times).
**Validates: Requirements 5.2**

**Property 16: Repeated vibration timing**
*For any* continuous hazardous noise lasting more than 5 seconds with vibration enabled, vibration should repeat every 5 seconds.
**Validates: Requirements 5.5**

### Background Operation Properties

**Property 17: Background capture continuity**
*For any* app backgrounding event, audio capture and analysis should continue without interruption.
**Validates: Requirements 8.1**

**Property 18: Background notification mechanism**
*For any* hazardous noise detection while running in background, alerts should use the device's system notification API.
**Validates: Requirements 8.2**

**Property 19: Screen lock monitoring**
*For any* device screen lock event with background permissions granted, monitoring should continue.
**Validates: Requirements 8.3**

**Property 20: Background detection accuracy**
*For any* identical audio input, detection results should be the same whether the app is in foreground or background.
**Validates: Requirements 8.5**

**Property 21: Background performance maintenance**
*For any* audio frame processed in background mode, analysis should complete within the same 100ms limit as foreground mode.
**Validates: Requirements 6.3**

### Privacy and Security Properties

**Property 22: No audio persistence**
*For any* audio frame processed by the system, no audio data should be written to persistent storage or transmitted over the network.
**Validates: Requirements 9.1, 9.2, 9.3**

**Property 23: Immediate audio disposal**
*For any* audio frame after processing is complete, the raw audio data should be immediately discarded from memory.
**Validates: Requirements 9.4**

**Property 24: Minimal data retention**
*For any* data retained by the application, it should consist only of calculated decibel values, not raw audio samples.
**Validates: Requirements 9.5**

### Performance and Resource Properties

**Property 25: CPU usage limit**
*For any* continuous monitoring session, CPU consumption should not exceed 15% of device capacity.
**Validates: Requirements 7.5**

### Error Handling Properties

**Property 26: Capture restart timing**
*For any* audio capture interruption, a restart attempt should occur within 2 seconds.
**Validates: Requirements 10.1**

**Property 27: Graceful degradation**
*For any* low memory condition, the system should reduce processing overhead while maintaining core noise detection functionality.
**Validates: Requirements 10.4**

**Property 28: Error logging and recovery**
*For any* unexpected error, the system should log error details and attempt to continue operation.
**Validates: Requirements 10.5**

## Error Handling

The Industrial Noise Monitor implements comprehensive error handling across all system layers to ensure reliable operation in industrial environments.

### Audio Capture Errors

**Permission Denial**:
- Error: User denies microphone permissions
- Handling: Display clear error message explaining that audio capture is required for safety monitoring
- Recovery: Provide button to open system settings for permission grant
- User Impact: Application cannot function without microphone access

**Microphone Conflict**:
- Error: Another application is using the microphone
- Handling: Display notification that exclusive microphone access is required
- Recovery: Periodically retry capture (every 5 seconds) until microphone becomes available
- User Impact: Monitoring paused until conflict resolved

**Capture Interruption**:
- Error: Audio capture stops unexpectedly (e.g., phone call, system interruption)
- Handling: Attempt automatic restart within 2 seconds
- Recovery: Retry up to 3 times with 2-second delays between attempts
- User Impact: Brief monitoring gap (2-6 seconds), then automatic recovery or error notification

**Hardware Failure**:
- Error: Microphone hardware malfunction
- Handling: Log detailed error information, display critical error to user
- Recovery: No automatic recovery possible
- User Impact: Application cannot function, user must resolve hardware issue

### Signal Analysis Errors

**Invalid Audio Data**:
- Error: Audio frame contains invalid samples (NaN, Infinity, out of range)
- Handling: Skip frame, log warning, continue with next frame
- Recovery: Automatic (next valid frame processed normally)
- User Impact: Single frame dropped (100-200ms gap), minimal impact

**Analysis Timeout**:
- Error: Frame analysis exceeds 100ms performance budget
- Handling: Log performance warning with timing details
- Recovery: Complete analysis and continue (accept latency)
- User Impact: Delayed alert (still within 500ms total budget)

**FFT Computation Error**:
- Error: FFT library returns error during frequency analysis
- Handling: Fall back to time-domain RMS calculation without A-weighting
- Recovery: Automatic fallback, log degraded mode
- User Impact: Less accurate dB readings, but core safety monitoring maintained

### Alert System Errors

**Notification Permission Denial**:
- Error: User denies notification permissions (background alerts)
- Handling: Continue with in-app alerts only, display warning about limited background functionality
- Recovery: Prompt user to enable notifications in settings
- User Impact: No background alerts, must keep app in foreground

**Vibration Unavailable**:
- Error: Device does not support vibration or vibration disabled at system level
- Handling: Disable vibration alerts, rely on visual alerts only
- Recovery: None (hardware limitation)
- User Impact: No haptic feedback, visual alerts still functional

**UI Rendering Failure**:
- Error: Alert UI fails to render
- Handling: Fall back to system notification, log UI error
- Recovery: Attempt to recreate UI on next alert
- User Impact: Alerts shown via system notifications instead of in-app UI

### Resource Exhaustion Errors

**Low Memory**:
- Error: System reports low memory condition
- Handling: Reduce frame buffer size, increase frame processing interval
- Recovery: Automatic when memory available
- User Impact: Slightly reduced temporal resolution (e.g., 200ms frames instead of 100ms)

**CPU Throttling**:
- Error: Device thermal throttling or battery saver mode
- Handling: Reduce FFT resolution, simplify A-weighting filter
- Recovery: Automatic when thermal/power conditions improve
- User Impact: Slightly less accurate dB readings, core functionality maintained

**Battery Critical**:
- Error: Device battery below 5%
- Handling: Display warning that monitoring may stop, suggest connecting to power
- Recovery: None (user must charge device)
- User Impact: System may be killed by OS, monitoring interrupted

### Background Operation Errors

**Background Permission Revoked**:
- Error: User revokes background execution permission
- Handling: Display notification that continuous monitoring requires foreground operation
- Recovery: Prompt user to re-enable background permission
- User Impact: Monitoring stops when app backgrounded

**Background Task Suspended**:
- Error: OS suspends background task (iOS/Android background limits)
- Handling: Request extended background execution time, display warning if denied
- Recovery: Automatic when app returns to foreground
- User Impact: Monitoring paused while backgrounded on some devices

### Error Logging

All errors are logged with the following information:
- Timestamp (ISO 8601 format)
- Error type and severity (CRITICAL, ERROR, WARNING)
- Component that generated the error
- Detailed error message and stack trace
- Device information (OS version, device model)
- Application state (foreground/background, current settings)

Logs are stored locally in a rolling file (max 10 MB) and can be exported for debugging.

## Testing Strategy

The Industrial Noise Monitor employs a dual testing approach combining unit tests for specific examples and edge cases with property-based tests for universal correctness guarantees.

### Testing Framework Selection

**iOS**:
- Unit Testing: XCTest (built-in)
- Property-Based Testing: SwiftCheck
- UI Testing: XCTest UI Testing

**Android**:
- Unit Testing: JUnit 4
- Property-Based Testing: junit-quickcheck
- UI Testing: Espresso

### Property-Based Testing Configuration

All property-based tests must:
- Run a minimum of 100 iterations per test to ensure comprehensive input coverage
- Include a comment tag referencing the design document property
- Tag format: `// Feature: industrial-noise-monitor, Property N: [property text]`
- Use appropriate generators for test data (audio samples, dB levels, timestamps)

### Test Coverage by Component

#### Audio Processor Tests

**Unit Tests**:
- Permission request flow on app startup (Example: Requirement 1.1)
- Error message display when permissions denied (Example: Requirement 1.3)
- Microphone conflict notification (Example: Requirement 10.3)
- Capture restart after 3 failed attempts (Example: Requirement 10.2)

**Property Tests**:
- Property 1: Permission-gated capture (100+ permission grant scenarios)
- Property 2: Audio configuration invariants (100+ captured frames)
- Property 3: Capture error recovery (100+ failure scenarios)
- Property 4: Capture continuity (100+ frame sequences)

#### Noise Analyzer Tests

**Unit Tests**:
- Known audio signal produces expected dB level (e.g., 1 kHz sine wave at known amplitude)
- Silent audio (all zeros) produces very low dB level
- Maximum amplitude audio produces high dB level
- Invalid audio data (NaN, Infinity) handled gracefully

**Property Tests**:
- Property 5: Decibel calculation correctness (100+ random audio frames)
- Property 6: Hazard classification correctness (100+ dB levels from 60-110 dB)
- Property 7: Hazard notification (100+ hazardous frames)
- Property 8: Stateless frame processing (100+ frame permutations)
- Property 9: Analysis performance (100+ frames with timing measurement)

#### Alert Manager Tests

**Unit Tests**:
- Settings UI provides vibration toggle (Example: Requirement 5.4)
- Background permission denial shows appropriate message (Example: Requirement 8.4)
- Alert displays red background and white text
- Alert shows dB level with one decimal place

**Property Tests**:
- Property 10: Alert timing (100+ hazard detections with latency measurement)
- Property 11: Alert visual requirements (100+ alert displays)
- Property 12: Alert clearing timing (100+ hazard-to-safe transitions)
- Property 13: Real-time alert updates (100+ sequences of changing dB levels)
- Property 14: Conditional vibration activation (100+ hazard detections with varied settings)
- Property 15: Vibration pattern consistency (100+ vibration triggers)
- Property 16: Repeated vibration timing (100+ continuous hazard scenarios)

#### Background Operation Tests

**Unit Tests**:
- App backgrounding triggers system notification API
- Screen lock with permissions allows continued monitoring
- Background task suspension displays warning

**Property Tests**:
- Property 17: Background capture continuity (100+ backgrounding events)
- Property 18: Background notification mechanism (100+ background hazard detections)
- Property 19: Screen lock monitoring (100+ screen lock events)
- Property 20: Background detection accuracy (100+ identical audio inputs in both modes)
- Property 21: Background performance maintenance (100+ background frame analyses)

#### Privacy and Security Tests

**Unit Tests**:
- No audio files created in app directory after processing
- No network requests contain audio data
- Only dB values stored in app state

**Property Tests**:
- Property 22: No audio persistence (100+ processing sessions with filesystem/network monitoring)
- Property 23: Immediate audio disposal (100+ frames with memory inspection)
- Property 24: Minimal data retention (100+ processing sessions with data inspection)

#### Performance Tests

**Property Tests**:
- Property 25: CPU usage limit (100+ monitoring sessions with CPU profiling)

#### Error Handling Tests

**Unit Tests**:
- Capture interruption triggers restart within 2 seconds
- Low memory condition reduces processing overhead
- Unexpected errors are logged with full details

**Property Tests**:
- Property 26: Capture restart timing (100+ interruption scenarios)
- Property 27: Graceful degradation (100+ low memory scenarios)
- Property 28: Error logging and recovery (100+ error scenarios)

### Integration Tests

Integration tests verify end-to-end flows across multiple components:

1. **Complete monitoring flow**: Audio capture → Analysis → Hazard detection → Alert display
2. **Background transition flow**: Foreground monitoring → App backgrounding → Background monitoring → Return to foreground
3. **Permission flow**: App start → Permission request → Permission grant → Capture start
4. **Error recovery flow**: Capture failure → Retry attempts → Recovery or error notification
5. **Settings change flow**: Change vibration setting → Detect hazard → Verify vibration behavior matches setting

### Test Data Generators

Property-based tests require generators for random test data:

**Audio Frame Generator**:
```typescript
generateAudioFrame(): AudioFrame {
  const sampleRate = 44100
  const durationMs = randomInt(100, 200)
  const sampleCount = Math.floor(sampleRate * durationMs / 1000)
  const samples = new Float32Array(sampleCount)
  
  // Generate random audio samples in valid range [-1.0, 1.0]
  for (let i = 0; i < sampleCount; i++) {
    samples[i] = randomFloat(-1.0, 1.0)
  }
  
  return new AudioFrame(samples, sampleRate, Date.now())
}
```

**Decibel Level Generator**:
```typescript
generateDbLevel(): number {
  // Generate dB levels in realistic range (40-120 dB)
  return randomFloat(40, 120)
}
```

**Hazardous Audio Generator**:
```typescript
generateHazardousAudio(): AudioFrame {
  // Generate audio that will produce >= 85 dB
  const frame = generateAudioFrame()
  // Scale samples to ensure high amplitude
  for (let i = 0; i < frame.samples.length; i++) {
    frame.samples[i] *= randomFloat(0.7, 1.0)
  }
  return frame
}
```

**Safe Audio Generator**:
```typescript
generateSafeAudio(): AudioFrame {
  // Generate audio that will produce < 85 dB
  const frame = generateAudioFrame()
  // Scale samples to ensure low amplitude
  for (let i = 0; i < frame.samples.length; i++) {
    frame.samples[i] *= randomFloat(0.0, 0.3)
  }
  return frame
}
```

### Continuous Integration

All tests run automatically on:
- Every commit to feature branches
- Pull requests to main branch
- Nightly builds for extended property test runs (1000+ iterations)

CI pipeline fails if:
- Any unit test fails
- Any property test fails
- Code coverage drops below 80%
- Performance tests exceed latency budgets
- Integration tests fail

### Manual Testing

Manual testing focuses on aspects not covered by automated tests:
- Subjective UI/UX quality (visual design, animations)
- Real-world industrial environment testing
- Device-specific compatibility issues
- Battery life impact over extended periods
- Accessibility features (VoiceOver, TalkBack)

Manual test scenarios:
1. Use app in actual industrial environment with known noise sources
2. Verify alerts are noticeable in high-noise environments
3. Test on variety of device models (old and new)
4. Verify battery consumption over 8-hour work shift
5. Test with accessibility features enabled
