# Wtr-Lab Core Engine & Controller Manual
## Native Android Subsystems, Services, and Bridges

This document details the mechanics of Wtr-Lab's core JVM subsystems, detailing initialization scopes, foreground states, audio stream channels, and bridge interfaces.

---

## 📱 MainActivity.kt (The Bootloader)

- **Namespace**: `com.example`
- **Inheritance**: `androidx.activity.ComponentActivity`

### 1. Responsibilities
- Bootstraps the application, hooks Jetpack Compose content view via `setContent`.
- Integrates Edge-to-Edge display elements `enableEdgeToEdge()`.
- Controls runtime permission grants (e.g., `POST_NOTIFICATIONS` for Android 13+ foreground services).
- Initializes the concurrent **Active WebView Pooling Pool** (`MainActivity.activeWebViewsPool`) inside global memory to avoid activity-context memory leaks across dynamic swaps.

### 2. Interface Definitions and Memory Pool
```kotlin
companion object {
    // Stores in-memory WebViews bound to applicationContext to prevent context leaks
    val activeWebViewsPool = mutableMapOf<Int, android.webkit.WebView>()
}
```

---

## 🎧 WtrBrowserService.kt (The Core Playback Engine)

- **Namespace**: `com.example`
- **Inheritance**: `android.app.Service` (Foreground Services)

### 1. High-Level Flow Chart
```
 [OnStartCommand] ----------------> [Configure Notification Channel]
        |                                       |
        v                                       v
 [Initialize TTS Engine] ------------> [Register Media Buttons Receiver]
        |                                       |
        +<========== [Observe State Changes in Playback List] <==========+
        |                                                                |
        v                                                                |
  [Speech Playback Loop: Speak chunk sequentially]                       |
        |                                                                |
        +------------------> [Trigger TTS OnRangeStart callback] --------+
```

### 2. Key Attributes and Implementations
- **TextToSpeech `ttsEngine`**: Initialized using standard `android.speech.tts.TextToSpeech`. Handles paragraph parsing.
- **MediaSessionCompat `mediaSession`**: Provides deep integration into system lockscreens, lockscreen metadata (novel name, chap title) update routines, BT accessories, and earphones widgets.
- **Throttling Gate**: Restricts notifications/media session updates of title metadata details to `1.5s` gates, eliminating IPC binder serialization bottlenecks when tracking word highlight updates.

### 3. Speech Sequential Pipeline & Auto-Advance Loop
```kotlin
private fun speakParagraph(text: String, index: Int) {
    if (ttsEngine == null) return
    val params = Bundle().apply {
        putString(TextToSpeech.Engine.KEY_PARAM_UTTERANCE_ID, "p_$index")
    }
    // Speak using state queue setup
    ttsEngine?.speak(text, TextToSpeech.QUEUE_FLUSH, params, "p_$index")
}
```

---

## 🌉 WtrAudioControlBridge.kt (The Unified Communication Bus)

- **Namespace**: `com.example`
- **Type**: `object` (Singleton)

Acts as the shared programmatic pipeline binding Compose views, WebViews, and the foreground speaker loop.

### 1. Reactive State Pipelines (StateFlows)
- `isPlaying: MutableStateFlow<Boolean>` (Default: `false`)
- `novelName: MutableStateFlow<String>` (Default: `""`)
- `chapterTitle: MutableStateFlow<String>` (Default: `""`)
- `activeWebsite: MutableStateFlow<String>` (Default: `""`)
- `playTrackInputList: MutableStateFlow<List<String>>` (Default: `emptyList()`)
- `currentTrackIndex: MutableStateFlow<Int>` (Default: `0`)
- `ttsSpeed: MutableStateFlow<Float>` (Default: `1.0f`)
- `ttsPitch: MutableStateFlow<Float>` (Default: `1.0f`)
- `activeTtsTabId: MutableStateFlow<Int?>` (Default: `null`)

---

## 🌐 WtrWebAppInterface.kt (The Android JavaScript Bridge)

- **Namespace**: `com.example`
- **Inheritance**: Any object mapped via `@JavascriptInterface`

Binds client-side DOM events into native JVM listener callbacks.

### 1. Concrete Interface API Callbacks
- `@JavascriptInterface fun postPlaybackState(isPlaying: Boolean, name: String, chTitle: String, activeUrl: String, speed: Float, pitch: Float)`: Updates metadata records with webpage-specified novel details.
- `@JavascriptInterface fun syncPollState(isPlaying: Boolean, currentParagraphIndex: Int, currentWordIndex: Int)`: Receives real-time DOM polling updates to keep physical paragraph indices in lockstep with TTS highlighter positions.
- `@JavascriptInterface fun onUrlSynced(triggeringTabId: Int, syncedUrl: String)`: Handles core address bar text state migrations on pages when active page content changes.

---

## 📈 BrowserViewModel.kt (The Orchestrator)

- **Namespace**: `com.example`
- **Inheritance**: `androidx.lifecycle.ViewModel`

The center of business logic execution. Runs query processing and tab orchestration.

### 1. Tab Management States & Persistence Loops
- Tracks and persists active tabs inside local Room memory structures via repository commands.
- Processes search engine input queries, detecting if an input is a valid HTTP URL or requires parsing as a Google Search fallback query.
- Coordinates the synchronization of tabs during dynamic lifecycle changes.

---

## 📝 WtrLogManager.kt (The Diagnostic System)

- **Namespace**: `com.example`
- **Type**: `object` (Thread-Safe Ring Buffer)

Houses logging buffers to track actions.

### 1. Diagnostic Architecture Specification
- Capped at `100` historical diagnostic events inside an in-memory thread-safe `linkedList` to avoid UI hangs or extreme memory ballooning.
- Log formats write the current timestamp alongside system-critical events:
  ```kotlin
  fun log(context: Context, event: String) {
      val timestamp = SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.getDefault()).format(Date())
      val record = "[$timestamp] $event"
      // Serialized and cached safely inside Shared Preferences when logging is turned on
  }
  ```
- Checked against `enable_logs` inside Android Shared Preferences to enforce privacy-focused opt-in behaviors.
- Critical system operations (e.g. paragraph extraction, page loads, volume adjustments) must write debug events through `WtrLogManager.log` for in-app debugging views.
