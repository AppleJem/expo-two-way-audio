# Update Logs

## 2025-10-01: Added `onPlaybackFinished` Event

### Overview
Implemented a new event handler that fires when audio playback completes. This allows React Native developers to detect when all queued audio has finished playing on both iOS and Android platforms.

### Event Details

**Event Name:** `onPlaybackFinished`

**Event Type:** `PlaybackFinishedEvent`
```typescript
type PlaybackFinishedEvent = {
  data: Record<string, never>; // Empty object - no data needed
};
```

**When it fires:** After the last audio buffer finishes playing and no more audio is queued.

---

## Implementation Details

### iOS Implementation (AudioEngine.swift)

**Approach:** Buffer tracking with completion handlers

iOS uses `AVAudioPlayerNode` which supports per-buffer completion callbacks. We track how many buffers are currently scheduled and fire the event when all buffers complete.

**Key Components:**

1. **Added Properties:**
   ```swift
   public var onPlaybackFinishedCallback: (() -> Void)?
   private var scheduledBuffersCount = 0
   private let bufferCountQueue = DispatchQueue(label: "com.speechmatics.expotowayaudio.bufferCount")
   ```

2. **Modified `playPCMData()` function:**
   - Uses `scheduleBuffer(_:completionCallbackType:completionHandler:)`
   - `completionCallbackType: .dataPlayedBack` ensures callback fires when audio finishes playing (not just when scheduled)
   - Thread-safe counter increment/decrement using serial dispatch queue
   - Fires callback when `scheduledBuffersCount` reaches 0

**Why this approach:**
- AVAudioPlayerNode allows queuing multiple buffers
- Each buffer can have its own completion handler
- Completion handlers run on audio rendering thread, requiring thread-safe counter
- `.dataPlayedBack` callback type ensures we detect actual playback completion, not just scheduling

**Code excerpt:**
```swift
// Increment buffer count before scheduling
bufferCountQueue.sync {
    scheduledBuffersCount += 1
}

// Schedule buffer with completion handler
speechPlayer.scheduleBuffer(buffer, completionCallbackType: .dataPlayedBack) { [weak self] _ in
    guard let self = self else { return }

    self.bufferCountQueue.sync {
        self.scheduledBuffersCount -= 1

        if self.scheduledBuffersCount == 0 {
            self.onPlaybackFinishedCallback?()
        }
    }
}
```

---

### Android Implementation (AudioEngine.kt)

**Approach:** Queue-based completion detection

Android uses a queue (`LinkedList<ByteArray>`) to buffer audio samples. The `playAudioFromSampleQueue()` function processes this queue on a background thread. Completion is naturally detected when the queue empties.

**Key Components:**

1. **Added Property:**
   ```kotlin
   var onPlaybackFinishedCallback: (() -> Unit)? = null
   ```

2. **Modified `playAudioFromSampleQueue()` function:**
   - Added callback invocation in the `finally` block
   - Fires after queue empties and `isPlaying` becomes false

**Why this approach:**
- Android's AudioTrack doesn't provide per-buffer completion callbacks
- The existing queue architecture naturally knows when playback finishes
- `finally` block ensures callback fires whether playback completes normally or with an error
- Simpler than iOS - no manual tracking needed

**Code excerpt:**
```kotlin
private fun playAudioFromSampleQueue() {
    executorServicePlayback.execute{
        isPlaying = true
        try {
            while (audioSampleQueue.isNotEmpty()){
                val data = audioSampleQueue.poll()
                if (data != null){
                    playSample(data)
                    val audioVolume = calculateRMSLevel(data)
                    onOutputVolumeCallback?.invoke(audioVolume)
                }else{
                    break
                }
            }
        }catch (e: Exception){
            Log.e("AudioEngine", "Error playing audio", e)
            e.printStackTrace()
        }finally {
            isPlaying = false
            onOutputVolumeCallback?.invoke(0.0F)
            onPlaybackFinishedCallback?.invoke()  // ← Added
        }
    }
}
```

---

## Files Modified

### TypeScript Layer
- **src/events.ts**: Added `PlaybackFinishedEvent` type and `PlaybackFinishedCallback` (already existed)

### iOS Native Layer
- **ios/ExpoTwoWayAudioModule.swift**:
  - Added `ON_PLAYBACK_FINISHED_EVENT_NAME` constant
  - Registered event in Events array
  - Added `setupPlaybackFinishedCallback()` function

- **ios/AudioEngine.swift**:
  - Added callback property and buffer tracking
  - Modified `playPCMData()` to use completion handlers

### Android Native Layer
- **android/.../ExpoTwoWayAudioModule.kt**:
  - Added `ON_PLAYBACK_FINISHED_EVENT` constant
  - Registered event in Events array
  - Added callback setup in `setupCallbacks()`

- **android/.../AudioEngine.kt**:
  - Added callback property
  - Invoked callback in `playAudioFromSampleQueue()` finally block

---

## Usage Example

### Basic Usage
```typescript
import { useExpoTwoWayAudioEventListener } from "@speechmatics/expo-two-way-audio";
import { useCallback } from "react";

function MyComponent() {
  useExpoTwoWayAudioEventListener(
    "onPlaybackFinished",
    useCallback(() => {
      console.log("Audio playback finished!");
      // Your logic here - play next audio, show UI update, etc.
    }, [])
  );

  // ... rest of your component
}
```

### Advanced Example: Sequential Playback
```typescript
import {
  useExpoTwoWayAudioEventListener,
  playPCMData
} from "@speechmatics/expo-two-way-audio";
import { useCallback, useState } from "react";

function SequentialAudioPlayer() {
  const [audioQueue, setAudioQueue] = useState<Uint8Array[]>([]);

  useExpoTwoWayAudioEventListener(
    "onPlaybackFinished",
    useCallback(() => {
      // Play next audio in queue when current finishes
      setAudioQueue(prev => {
        const [, ...rest] = prev;
        if (rest.length > 0) {
          playPCMData(rest[0]);
        }
        return rest;
      });
    }, [])
  );

  const queueAudio = (audioData: Uint8Array) => {
    setAudioQueue(prev => {
      const newQueue = [...prev, audioData];
      // If this is the first item, start playing
      if (prev.length === 0) {
        playPCMData(audioData);
      }
      return newQueue;
    });
  };

  return (
    <View>
      <Text>Queue length: {audioQueue.length}</Text>
      {/* Your UI here */}
    </View>
  );
}
```

### With Alert/Notification
```typescript
import { Alert } from "react-native";

useExpoTwoWayAudioEventListener(
  "onPlaybackFinished",
  useCallback(() => {
    Alert.alert("Playback Complete", "Audio has finished playing");
  }, [])
);
```

---

## Testing

### Development Testing
The module includes example apps for fast iteration:

```bash
# Navigate to example
cd examples/basic-usage

# Build and run on device (required for native changes)
npx expo run:ios --device --configuration Release
# or
npx expo run:android --device --variant release
```

### Test Scenario
1. Initialize the audio module
2. Register the `onPlaybackFinished` event listener
3. Play PCM audio data using `playPCMData()`
4. Verify the callback fires when audio finishes
5. Test with multiple rapid `playPCMData()` calls to ensure event fires only once at the end

---

## Platform Differences

| Aspect | iOS | Android |
|--------|-----|---------|
| **Detection Method** | Per-buffer completion handlers | Queue emptying |
| **Threading** | Serial dispatch queue for thread safety | ExecutorService with finally block |
| **Complexity** | Manual counter tracking | Automatic via queue architecture |
| **Callback Type** | `.dataPlayedBack` (AVAudioPlayerNode) | N/A (no built-in callback) |
| **Edge Cases** | Must handle buffer count on audio thread | Naturally handles via queue state |

---

## Notes

- The event fires **once** when all audio finishes, not after each buffer
- Works with rapid consecutive `playPCMData()` calls - event fires after the last buffer
- Both platforms reset output volume to 0 when playback finishes
- Thread-safe implementation on both platforms
- No breaking changes - purely additive feature
