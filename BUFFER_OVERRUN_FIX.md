# Buffer Overrun Fix for Windows Crash (Exception 0xc0000409)

## Problem Summary

The Handy application was crashing on Windows with exception code `0xc0000409` (STATUS_STACK_BUFFER_OVERRUN) when users pressed the hotkey to start recording. This exception indicates that Windows detected a stack buffer overrun and terminated the process as a security measure.

## Root Cause Analysis

After analyzing the codebase, I identified the issue in the audio recording callback function in `src-tauri/src/audio_toolkit/audio/recorder.rs`:

### Primary Issue: Unsafe Buffer Operations in Audio Callback

**Location**: `recorder.rs` lines 157-193 (original code)

The `build_stream` function creates an audio input stream with a callback that processes incoming audio data. The callback had several unsafe buffer operations:

1. **No pre-allocation**: `Vec::new()` started with zero capacity, causing multiple reallocations
2. **No bounds checking**: No validation that incoming audio data length was valid
3. **Unsafe chunk handling**: Used `chunks_exact()` which panics if data length isn't evenly divisible by channel count
4. **No validation of frame data**: Assumed all audio frames were complete and valid

### Secondary Issues: Buffer Processing Pipeline

**Location**: `src-tauri/src/audio_toolkit/audio/resampler.rs`

The resampler that processes audio buffers also lacked safety checks:

1. No validation of input data before processing
2. No bounds checking before buffer operations
3. Could overflow if invalid data was passed from the audio callback

## The Fix

### 1. Audio Callback Buffer Safety (recorder.rs)

**Changes made**:

```rust
// BEFORE:
let mut output_buffer = Vec::new();

// AFTER:
let mut output_buffer = Vec::with_capacity(4096);
```

**Additional safety improvements**:

- Added empty buffer validation
- Pre-reserve exact capacity before extending buffer
- Added warning for non-evenly divisible channel data
- Changed from `chunks_exact()` to `chunks().take()` with frame validation
- Added frame completeness check before processing
- Only send non-empty buffers

### 2. Resampler Safety (resampler.rs)

**Changes made to `push()` method**:

- Added empty input validation
- Added buffer space overflow protection
- Added bounds checking before slice operations
- Added error logging for debugging

**Changes made to `emit_frames()` method**:

- Added empty data validation
- Added pending buffer overflow protection
- Added bounds checking before slice operations
- Added error logging for debugging

## Technical Details

### Why This Caused Stack Buffer Overrun on Windows

On Windows, the audio driver callback can sometimes deliver:
- Unexpected buffer sizes
- Partial frames when drivers are under stress
- Data that doesn't align with expected channel counts

Without proper validation and bounds checking, the Vec operations could:
1. Write beyond allocated capacity
2. Access invalid memory when slicing
3. Cause the Windows /GS stack protection to detect the corruption and trigger 0xc0000409

The stack protection feature (Fail Fast) detected these unsafe memory operations and immediately terminated the process to prevent potential security exploits.

### Why Version 0.6.10 Still Had the Issue

The fault offset changed between versions (0x803b15 in v0.6.9.0 vs 0x7e03f5 in v0.6.10.0), indicating the code layout changed but the underlying bug persisted. This is typical when:
- The bug is a logic error rather than being fixed
- Code compilation slightly shifted memory layout
- Optimization levels changed between builds

## Testing Recommendations

After applying this fix, test the following scenarios on Windows:

1. **Basic recording**: Press hotkey, speak, release - verify transcription works
2. **Rapid toggling**: Quickly press/release hotkey multiple times
3. **Different audio devices**: Test with various microphones and audio interfaces
4. **Multi-channel audio**: Test with stereo and multi-channel input devices
5. **Driver stress**: Test while system is under high audio load
6. **Long recordings**: Test recordings longer than 30 seconds

## Additional Hardening Recommendations

### 1. Consider Using Bounded Channels

Replace unbounded `mpsc::channel` with bounded channels to prevent memory exhaustion:

```rust
let (sample_tx, sample_rx) = mpsc::sync_channel::<Vec<f32>>(100);
```

### 2. Add Telemetry for Audio Buffer Stats

Log buffer sizes periodically to catch anomalies:

```rust
if data.len() > 8192 {
    log::warn!("Unusually large audio buffer: {} samples", data.len());
}
```

### 3. Add Panic Handler

Add a panic hook to log detailed information before crash:

```rust
std::panic::set_hook(Box::new(|panic_info| {
    log::error!("PANIC: {:?}", panic_info);
}));
```

## Files Modified

1. `src-tauri/src/audio_toolkit/audio/recorder.rs` - Lines 157-222
2. `src-tauri/src/audio_toolkit/audio/resampler.rs` - Lines 37-133

## Build Instructions

To build and test the fix:

```bash
cd Handy
npm install
cd src-tauri
cargo build --release
```

To run in development mode:

```bash
npm run tauri dev
```

## References

- Windows Error Code 0xc0000409: https://learn.microsoft.com/en-us/windows/win32/debug/system-error-codes
- Rust Vec documentation: https://doc.rust-lang.org/std/vec/struct.Vec.html
- CPAL audio library: https://github.com/RustAudio/cpal

