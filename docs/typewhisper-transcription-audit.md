# TypeWhisper transcription pipeline audit

For implementation, start with the [technical roadmap](transcription-implementation-plan.md); for gathered findings and exploration, see the [consolidated research brief](transcription-research-brief.md). This audit is the supporting source investigation.

Read-only source review on 2026-11-07 of `/Users/sihochoi/Developer/Projects/typewhisper-mac`, at commit `9852feb1`. The review follows the shared dictation pipeline, both capture transports, Apple and Grok speech adapters, text delivery, recovery, and the separate recorder/file paths. Relevant regression tests were inspected as evidence of intended behavior; they were not executed. Other third-party recognition engines were not individually audited. No reference engine code has been imported.

This document distinguishes observed implementation behavior from recommendations for TypeThis. It contains no measured claim that TypeThis is faster. A passing test name or source comment is not a hardware measurement.

## Overall result

Our original design covered streaming, bounded buffers, basic silence handling, provider composition, and finalization. It needed stronger requirements for capture lifetime, device compatibility, incomplete audio, deadlines, delivery uncertainty, vocabulary capabilities, and recovery semantics.

TypeWhisper already has substantial performance engineering. Its input-only HAL path uses preallocated C storage, lock-free producer/consumer indices, explicit frame-loss accounting, and callback lifetime gates. Replacing that with a simpler-looking Swift implementation is not evidence of improvement. TypeThis's clearest opportunities are its narrower scope, fewer representations/copies, one recognition session, and less nonessential work on the path to insertion.

```mermaid
flowchart LR
    H[Hotkey and permissions] --> P[Snapshot target and session options]
    P --> C[Prepare capture and recognizer]
    C --> A[Capture, metrics, conversion]
    A --> R[Recognition and preview]
    R --> F[Drain and finalize]
    F --> T[Optional text transforms]
    T --> D[Deliver and verify]
    D --> O[Retain result and deferred persistence]
```

Cancellation, deadlines, audio loss, and resource cleanup cross every stage. They cannot be confined to an error handler at the end.

## Corrections and qualifications to the earlier design

| Earlier simplification | Finding | Revised decision |
| --- | --- | --- |
| An event-driven bounded buffer should outperform the reference capture engine. | Its HAL transport already uses a preallocated lock-free ring with a two-second capacity target, drained every 10 ms. The separate recognition feed polls at 350 ms. | Treat these as different layers. Remove avoidable recognition-feed waiting, but compare notification/coalescing with short active-capture polling before choosing. Keep idle capture off by default. |
| Voice processing should always be off. | The reference has a targeted three-channel built-in microphone workaround for concurrent call use, enabling voice processing and system AGC. | Leave optional DSP off normally, with a narrow capture compatibility policy when tests demonstrate a requirement. |
| A provider's advertised vocabulary support means the model uses hints. | Apple hints are passed by the reference, but published Apple guidance limits their effect to DictationTranscriber. Conversely, the Grok adapter declares hints unsupported while current xAI docs expose keyterms. | Validate capabilities per selected model and target OS/API version. |
| Finishing recognition means the operation succeeded. | Native text writes and synthetic pastes can fail, arrive late, or be unverifiable. | Track recognition outcome and delivery outcome separately. |
| A recovery file provides crash recovery. | TypeWhisper deletes its active WAV on next launch because its header is finalized only when preserving the recording. | Distinguish handled-failure recovery, crash recovery, and history/archive retention. |
| Freezing the provider is enough. | Locale, asset lease, account, model, format, original target, and output transformations also affect interpretation. | Freeze a complete session configuration; changes apply to the next dictation. |

## Findings across the pipeline

### 1. Capture and recognition have separate readiness

The reference starts microphone work asynchronously and defers expensive app metadata/URL resolution. Its ready cue waits for an actual first audio buffer and a recording-state/context payload. Bluetooth has additional readiness checks. A hotkey press, successful engine start, and usable microphone audio are distinct events.

TypeThis should snapshot lightweight target identity and configuration immediately, then start capture and backend preparation concurrently. Bound audio accumulated before the backend is ready. Expensive AX text reads, app icons, history, and optional context extraction must not delay the first captured word. A release/cancel while preparing must not allow the microphone to start afterward. Start cues should reflect capture readiness; backend connection/model loading can be represented separately.

Evidence: [start orchestration](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/ViewModels/DictationViewModel.swift:2116), [ready cue](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/ViewModels/DictationViewModel.swift:1648), [deferred metadata](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/ViewModels/DictationViewModel.swift:2588).

### 2. Device selection needs an explicit capture policy

TypeWhisper distinguishes default built-in, Bluetooth, and explicit input-only HAL routes. It persists device UIDs and resolves runtime device IDs. Bluetooth startup can deliver continuous zero samples, and route changes can occur between reading a format and installing a tap. Selected devices can disappear after undocking or sleep.

Start with a small capture transport boundary behind `AudioCapture`; keep it independent of Apple/Grok recognition. AVAudioEngine is a reasonable initial default, but do not promise universal USB, virtual, aggregate, or Bluetooth behavior until tested. Resolve saved UIDs each session. An explicitly selected microphone disappearing should surface an input error rather than silently choosing a different microphone. Reconfiguration must preserve continuity metadata or mark the result incomplete.

Evidence: [device identity](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/AudioDeviceService.swift:264), [input-only route](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/AudioRecordingService.swift:3399), [format precondition](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/AudioRecordingService.swift:3883).

### 3. Capture compatibility can require targeted processing

The reference's built-in three-channel policy is a compatibility workaround documented in its source, not proof that every three-channel input needs AGC. Its voice-processing path can duck other apps and is deliberately excluded from idle preparation. Teardown disables voice processing even when retaining the stopped engine temporarily.

TypeThis should test dictation while a call app is active. Introduce a targeted workaround only when reproduced, and restore owned changes on every stop/failure path. Keep amplitude normalization, channel conversion, voice processing, and text normalization separate in code and measurements.

Evidence: [capture policy](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/AudioRecordingService.swift:29), [voice processing](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/AudioRecordingService.swift:3295), [teardown](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/AudioRecordingService.swift:3189).

### 4. Bounded memory does not establish lossless capture

The HAL ring rejects whole slices when full, counts dropped frames and render failures, and drains its tail on stop. The Swift delivery layer logs those counts. That is stronger accounting than a generic bounded AsyncStream, but logging loss is different from making it part of the final result.

TypeThis needs frame sequence/count accounting through capture, conversion, and backend submission. Pool exhaustion, a full queue, a conversion failure, and a failed send must become explicit session health events. A result based on missing audio must not be reported as complete. Define limits on every queue, including disk and network work, rather than just the first ring.

Evidence: [ring producer](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/CoreAudioHALCallbackContext.c:327), [loss accounting](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/AudioDeviceService.swift:2908), [concurrent accounting test](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisperTests/AudioEngineRecoverySupportTests.swift:5343).

### 5. Stop is a drain barrier; cancellation has different semantics

TypeWhisper gives exceptionally short recordings a bounded opportunity to receive real tail audio: the default stop policy targets 50 ms buffered after the request, with up to 60 ms extra capture. This differs from later appending synthetic silence. The HAL stop drains admitted slices before returning, while capture generation tokens reject late callbacks from retired streams.

TypeThis must stop the producer, drain accepted buffers, flush conversion state, then end recognizer input. Cancellation can abandon recognition, but must still release capture resources. Test immediate press/release and rapid restart. Do not copy the reference's timing constants without a short-word/device corpus. Synthetic zero padding cannot recover an uncaptured consonant.

Evidence: [short-stop policy](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/AudioRecordingService.swift:335), [stop drain regression](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisperTests/AudioEngineRecoverySupportTests.swift:4820), [retired stream handling](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/AudioRecordingService.swift:3547).

### 6. Native callback lifetime outlives a Swift task

The C callback gate counts admitted callbacks, closes before teardown, waits for quiescence, and seals against a last-moment admission before freeing context. Some AVAudioEngine instances are retained after stop because delayed native callbacks can outlive shutdown. Swift task cancellation alone does not provide either guarantee.

Apple also documents that hardware format changes stop and uninitialize AVAudioEngine, and warns against deallocating it synchronously in the configuration-change notification handler. Schedule control work outside that callback. Keep any C/Objective-C bridge narrow and justified by the transport; do not erase a lifetime safeguard to reduce line count. [Apple configuration-change documentation](https://developer.apple.com/documentation/foundation/nsnotification/name-swift.struct/avaudioengineconfigurationchange)

Evidence: [callback gate](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/CoreAudioHALCallbackContext.c:17), [native finalization](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/AudioDeviceService.swift:2960), [engine retention rationale](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/AudioRecordingService.swift:639).

### 7. Recovery must have a circuit breaker

The reference debounces audio-configuration notifications, separates benign startup renegotiation from real changes, applies backoff, and gives up after a burst of failures. Idle pre-roll rearming has its own failure limit. Sleep and screen lock suspend armed input; wake invalidates prepared routes. Hotkey handling also has event-tap and lost-release recovery.

TypeThis should have one bounded recovery owner for capture. Track stop/cleanup separately from user-facing idle state so an old cleanup cannot stop a new recording. Ignore stale recovery notifications by generation. Release on suspension, refresh routing on resume, and avoid an indefinite restart loop. Deliberately limit first-release shortcut modes instead of implementing every hybrid/mouse/workflow interaction.

Evidence: [recovery policy](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/AudioEngineRecoverySupport.swift:19), [recovery coordinator integration](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/AudioRecordingService.swift:2464), [sleep/wake integration](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/App/TypeWhisperApp.swift:1067), [queued/cancelled starts](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/ViewModels/DictationViewModel.swift:2352).

### 8. Input liveness, silence, and turn completion use different clocks

The reference's silence endpoint detector counts audio frames. No incoming buffers therefore do not count as a speaker's silence. It requires an initial speech run before arming auto-stop, and can use backend turn-end information. Preview hallucination suppression shares the same activity rule, while separate startup logic handles zero-only Bluetooth input.

TypeThis should use audio sample time for speech/quiet duration and monotonic wall time for input stalls and network deadlines. Keep no frames, digital silence, quiet sound, non-speech, and recognizer no-result outcomes distinct. Push-to-talk release should finish promptly; automatic silence stop is a separate optional behavior. Do not remove quiet packets from an active stream merely because a meter is low.

Evidence: [audio-clock endpoint detector](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/ViewModels/DictationSilenceEndpoint.swift:23), [preview activity gate](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/ViewModels/StreamingHandler.swift:598).

### 9. Signal metrics can be shared without adding a detector model

The reference computes levels, optional gain, streaming activity, and endpoint activity in separate places. A small shared stream of fixed-duration activity summaries can avoid rescanning/copying samples for each consumer. Actual speech classification remains distinct from energy measurement.

TypeThis should calculate input statistics once in its audio worker and use them for metering, input health, and any optional endpoint detector. Capture blocks, activity frames, UI updates, and provider packets are different granularities. If gain is added, its smoothing constants should derive from audio duration rather than depend on a variable callback/packet count. Validate finite samples and saturating PCM conversion without amplifying noise by default.

Evidence: [gain and RMS](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/AudioRecordingService.swift:171), [activity frames](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/ViewModels/DictationSilenceEndpoint.swift:6).

### 10. Provider availability is a capability/model/session property

TypeWhisper prepares locales and assets, waits for existing releases, and protects active model use from auto-unload. Its plugin-wide selected model is temporarily overridden and restored, which needs careful lifetime handling. Its language selection distinguishes auto, exact language, and multiple hints.

TypeThis should pass model, locale, account, and options by value into a new session rather than mutating a global selected model. Availability should distinguish unsupported, needs asset download, needs sign-in, preparing, and ready. Apple speech availability remains independent of Apple Intelligence. Hold an asset/model lease until finish/cancel completes. Do not label a fixed-locale recognizer as automatic multilingual recognition.

Evidence: [session preparation and lease](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/ModelManagerService.swift:1179), [Apple asset reservation](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisperPluginSDK/Plugins/SpeechAnalyzerPlugin/SpeechAnalyzerPlugin.swift:283), [language selection](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/ViewModels/SettingsViewModel.swift:17).

### 11. Vocabulary hints need behavioral validation

The local Apple adapter passes contextual strings into a SpeechAnalyzer hosting SpeechTranscriber. Apple's accepted engineering response dated January 2026 says contextual strings affect DictationTranscriber and are ignored by SpeechTranscriber; the contextualStrings documentation also specifically describes DictationTranscriber. Passing the API call successfully is insufficient evidence of model support. Verify any later target-OS change before advertising that feature. [Apple engineering response](https://developer.apple.com/forums/thread/811083), [contextualStrings](https://developer.apple.com/documentation/speech/analysiscontext/contextualstrings)

The local Grok plugin declares dictionary terms unsupported, but the current public STT reference documents keyterm biasing. TypeThis can expose a small optional vocabulary capability for supported models. Do not inherit the reference adapter's limitation or compensate with an always-on LLM pass. Test names and technical terms both with speech and with silence, because bias must not create invented words. [xAI keyterms](https://docs.x.ai/developers/rest-api-reference/inference/speech-to-text)

Evidence: [Apple context setup](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisperPluginSDK/Plugins/SpeechAnalyzerPlugin/SpeechAnalyzerPlugin.swift:229), [Grok capability declaration](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisperPluginSDK/Plugins/XAIPlugin/XAIPlugin.swift:616).

### 12. Connected, producing partials, and complete are different recognition states

The reference detects failed appends and recognizes a live session that remains connected but stops adding text despite additional speech. It can reject a final result that contains only a short tail and retain a more substantive accumulated preview. Apple and Grok distinguish draft text from finalized text; Grok also has utterance and session completion snapshots.

TypeThis needs ordered transcript events with explicit revision and completion semantics. Do not concatenate every final-looking message. Keep provider-specific reconciliation inside the adapter, retain confirmed output after failures, and attach capture continuity to the result. Transport heartbeat, active speech with no new text, and a quiet speaker need different checks. Show a bounded preview tail rather than repeatedly rebuilding and fuzzy-matching an entire long transcript.

Evidence: [live completion validation](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/ViewModels/StreamingHandler.swift:234), [progress accounting](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/ViewModels/StreamingHandler.swift:582), [Grok collector](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisperPluginSDK/Plugins/XAIPlugin/XAIPlugin.swift:158).

### 13. Total deadlines must cover every asynchronous phase

TypeWhisper's final-batch path has an outer deadline that can return even when a provider ignores cancellation. Its shared HTTP retry budget separately bounds retry scheduling, not total elapsed operation time. The Grok live path waits for the server-created event and later waits for its receive task; do not infer that a URLRequest timeout provides the same total deadline guarantee for every live phase.

TypeThis should bound connect/readiness, queued audio age, finalization after stop, optional post-processing, and delivery verification separately. Cancellation/timeout must close the relevant transport and fence late results. A timeout race that returns while a resource continues running still needs bounded resource ownership. Do not start unlimited replacements for noncooperative work. Respect permanent authentication errors and retryability; a server that accepted audio may already have performed chargeable work.

Evidence: [outer final deadline](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/ViewModels/DictationViewModel.swift:3799), [retry-budget semantics](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisperPluginSDK/Sources/TypeWhisperPluginSDK/HostServices.swift:165), [Grok readiness](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisperPluginSDK/Plugins/XAIPlugin/XAIPlugin.swift:1020), [live finish](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisperPluginSDK/Plugins/XAIPlugin/XAIPlugin.swift:292).

### 14. Cloud reconnect cannot silently promise resume

The Grok reference falls back from some streaming requests to REST, which can recognize the same audio again. Live and batch transcription are different transports. Sending bytes successfully is not proof the recognizer incorporated them, and an arbitrary reconnect/replay strategy can repeat text or omit audio.

TypeThis should make stream failure explicit and only retry retained audio through a defined whole-session policy. Automatic resume requires an actual provider resume/acknowledgment protocol; otherwise do not claim it. Keep API-key and OAuth credential sources inside the Grok provider, coalesce refresh, and check speech access independently from language-model access. The earlier OAuth speech entitlement question remains unresolved by this source review.

Evidence: [stream-to-REST fallback](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisperPluginSDK/Plugins/XAIPlugin/XAIPlugin.swift:637).

### 15. Text delivery is a transaction with uncertainty

TypeWhisper tries direct Accessibility writes where appropriate, verifies that the field changed, and has exceptions for terminal applications, hidden input proxies, and apps known to ignore writes. Some writes report errors but apply; others report success but do nothing. An accepted write can become unreadable before verification. It therefore avoids blindly pasting after every uncertain write.

TypeThis's delivery result should distinguish confirmed, unverified, target changed/unavailable, and clipboard-only. Snapshot process/field/selection identity at activation, then validate again before mutation. Keep the recognized result available when delivery fails. Reject unsupported secure/composing targets explicitly rather than pretending all editors behave the same. Use UTF-16 range conversion correctly for AX operations, including emoji and combining characters. Defer live in-field replacement until its own compatibility tests pass.

Evidence: [verified AX insertion](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/TextInsertionService.swift:779), [insertion routing](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/TextInsertionService.swift:1541), [target validation](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/TextInsertionService.swift:2474).

### 16. Clipboard restoration is owned cleanup, even after cancellation

The reference snapshots all pasteboard representations, associates its generated payload with the pasteboard change count, verifies/delays restoration, and yields ownership when the user copies something new. Overlapping insertions transfer the original snapshot rather than restoring another dictation's temporary payload. A posted paste may still land after its caller cancels, so verification/restore cleanup continues independently.

TypeThis needs this ownership principle even if its implementation is smaller. Before posting a paste, cancellation can prevent it. After posting, do not pretend cancellation undid the external edit. Do not retry an uncertain edit automatically. Never restore stale clipboard content over a user's newer copy, or replace a queued paste's payload before it is consumed. Clipboard preservation is not an immediate save/write/restore sequence.

Evidence: [pending-paste barrier](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/TextInsertionService.swift:903), [restore ownership](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/TextInsertionService.swift:947), [cancellation regression](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisperTests/TypeWhisperIntegrationTests.swift:9014).

### 17. Post-processing can change meaning and invalidate earlier work

The reference applies ordered number/date/time normalization, speech punctuation, dictionary corrections, snippets, formatting, plugins, and optional LLM steps. Some dictionary rules run before and after the LLM. Language selection matters: its normalizer explicitly protects ordinary words in other languages from reinterpretation. Incremental workflow processing only reuses work when its configuration and final transcript prefix remain compatible.

TypeThis should preserve raw recognition separately from formatted output. Here raw means the provider's output before app-owned transformations; a provider may already have removed fillers or normalized punctuation/numbers. It is not automatically a verbatim record of the audio. Make formatting predictable and language-aware. Keep the baseline free of mandatory LLM cleanup. If optional cleanup fails, preserve usable raw text. Later incremental cleanup needs stable segments, bounded concurrency, and invalidation on transcript revision; parallel requests alone do not guarantee less end-to-end latency. Auto-submit, if ever added, requires separate confirmation of delivery/target state and must not follow a failed transformation as if it succeeded.

Evidence: [ordered transforms](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/PostProcessingPipeline.swift:197), [multilingual normalization](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/TranscriptionNormalizationService.swift:65), [incremental eligibility](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/ViewModels/DictationViewModel.swift:5211), [incremental session](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/WorkflowSegmentedPostProcessing.swift:180).

### 18. Recovery, history, and crash durability are separate features

TypeWhisper writes an active PCM16 WAV in batches on a utility queue, can preserve it after handled failures, compresses preserved audio later, retains a small recent-success set, and applies retention policy. It deletes the unfinished active file on launch. Thus an active recording is not a crash-recoverable journal. Audio write failures also invalidate that active recovery file.

A bounded transport queue cannot replay an entire earlier utterance. TypeThis must explicitly choose between no whole-session replay and a bounded temporary disk spool. If a spool is included, define byte/duration quotas, disk-full behavior, deletion after delivery, and whether crash recovery is supported. Crash recovery needs a recoverable data/journal format or header reconstruction, not just a file named recovery. Independently retain successful raw/final text through delivery errors; an optional historical audio archive need not be on the critical path.

Evidence: [recovery format and launch cleanup](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/DictationRecoveryAudioStore.swift:93), [batched writes](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/DictationRecoveryAudioStore.swift:216).

### 19. Persistence should follow insertion, and undo must be scoped

The reference defers history audio encoding, usage-counter persistence, and history saves until after insertion. Its safe undo records an exact target, value, caret, and freshness snapshot; it does not search a document or blindly synthesize a sequence of deletions. Contextual insertion formatting also handles word boundaries and CJK spacing separately from recognition.

TypeThis should keep slow storage and optional history behind delivery. Retain the immediate result first, then persist according to policy. Do not make every dictation wait for browser metadata. If undo is added, make it conditional on the same unchanged target/text. For initial plain-text delivery, avoid automatic casing/punctuation edits until tested; spacing and replacement boundaries still need Unicode-aware handling.

Evidence: [deferred persistence](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/ViewModels/DictationViewModel.swift:1419), [safe undo](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/DictationUndoService.swift:5), [insertion formatting](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/ViewModels/DictationViewModel.swift:6035).

### 20. Long files need a separate resource policy

The file loader reads via AVAssetReader but accumulates all decoded Float32 samples. The shared AudioData carries both samples and an encoded WAV. One hour at 16 kHz mono is 230.4 MB as Float32, and a simultaneous PCM16 representation adds approximately 115.2 MB before extra copies. These are format calculations, not measured application footprints.

The recorder path improves queued-job memory by referring to a finalized file, verifies that its duration covers captured audio, and avoids reloading audio when a live result is sufficient. The chunking helper cuts near quiet stretches, offsets timestamps, and keeps speaker identifiers from different requests distinct. Grok's file path also avoids replaying a long prerecorded file at real-time speed.

TypeThis should not make file transcription an alias for live microphone upload. Later file support should decode/process incrementally or stream the original supported file, respect backend-specific size/time limits, and define timestamp mapping across cuts. Speaker diarization and microphone/system-audio mixing remain separate optional workloads.

Evidence: [file loader](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/AudioFileService.swift:54), [dual representation](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisperPluginSDK/Sources/TypeWhisperPluginSDK/TypeWhisperPlugin.swift:590), [file-backed recorder jobs](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/ViewModels/AudioRecorderViewModel.swift:1033), [chunking and timestamp offsets](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisperPluginSDK/Sources/TypeWhisperPluginSDK/PluginAudioChunking.swift:14), [Grok file routing](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisperPluginSDK/Plugins/XAIPlugin/XAIPlugin.swift:1061).

## Concrete design changes for TypeThis

Keep one application target and narrow interfaces. The additional correctness requirements do not require a general plugin system or dozens of service protocols.

| Owner | Responsibility |
| --- | --- |
| Dictation coordinator | Frozen session configuration, lifecycle, deadlines, cancellation, terminal outcome |
| Audio capture/worker | Device routing, owned buffers, conversion, statistics, continuity/loss, drain and teardown |
| Speech backend/session | Model preparation, asset lease, ordered audio input, provider events, explicit finish/cancel |
| Transcript reducer | Segments/revisions, raw final text, completion and incompleteness |
| Text delivery | Target identity, AX/paste routing, verification status, clipboard ownership and cleanup |
| Result/recovery storage | Immediate retained result, optional disk spool, deferred history and retention |
| Provider directory/credentials | Apple/Grok capability resolution and provider-specific authentication |

These can be concrete types with two or three interface boundaries. UI receives small observable state. Audio buffers do not travel through SwiftUI observation. Keep per-session objects short-lived; keep only genuinely reusable connection/conversion resources alive.

The later autocomplete feature should share provider credentials and the platform text-delivery boundary. Give text mutations an origin/revision identity, invalidate old suggestions after a dictation commit, and serialize host-editor writes. Avoid treating our own insertion as fresh human keystrokes or letting autocomplete replace text while a dictation commit is pending. A general inference scheduler is unnecessary initially; pausing suggestions during active dictation is a simple starting policy.

Distribution is an early product decision: the reference has separate App Store behavior, including manual-paste and disabled live-field paths. Do not silently promise the same cross-app capabilities for every entitlement/distribution configuration. Validate the intended signed build instead of implementing both editions preemptively. [Reference distribution branch](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/TextInsertionService.swift:1541).

## First-release scope and explicit tradeoffs

Include continuous Apple/Grok speech, one preview/final session, input health, bounded queues, complete-session option snapshots, cancellation/teardown guarantees, per-phase deadlines, revision-aware transcripts, and an honest delivery outcome. Retain the result when insertion fails. Include model/language/asset/account readiness without a broad model-management framework.

Keep optional or defer: always-on pre-roll, automatic silence stop, multiple hotkey styles, live in-field replacement, automatic retry/second recognition pass, background LLM cleanup, auto-submit, speaker detection, meeting automation, screen/browser context, archives, and dynamic plugins. Pre-roll's 0.5-second ring can improve perceived startup only by keeping eligible inputs active; its power/privacy/route effects are a real tradeoff.

Choose and document the maximum supported dictation duration and recovery policy before claiming bounded total memory. Transport audio can be bounded while transcript, deferred results, history jobs, and retry audio still grow.

## Verification matrix before implementation is considered complete

| Area | Minimum scenarios |
| --- | --- |
| Startup | Cold/warm Apple model, Grok handshake delay, absent model asset, denied permission, release before first buffer |
| Hardware | Built-in, Bluetooth, USB, virtual input; input-only selection; concurrent call; route/sample-rate change; undock; sleep/wake |
| Signal | No frames, exact zeros, quiet one-word utterance, clipping/invalid samples, keyboard noise, background speech, long pause |
| Accounting | Pool/queue overflow, failed conversion, failed send, final short packet, no duplicated/dropped accepted frames, rapid stop/start |
| Recognition | Repeated partials, overlapping finals, authoritative completion snapshot, speech without new text, server close before completion |
| Lifecycle | Cancel during startup/handshake/send/finalization, late results after cancellation, noncooperative task, idempotent finish/cancel |
| Delivery | Native editor, browser contenteditable, Electron editor, terminal, changed field/selection, secure/composing input, app quit/relaunch |
| Clipboard | Delayed paste, uncertain AX write, user copies during restore, consecutive dictations, cancellation after paste, quit during restore |
| Language | Exact locale, unsupported locale, mixed-language speech, CJK boundaries, emoji/combining marks, vocabulary hints with/without speech |
| Recovery | Network failure after audio acceptance, unavailable replay audio, disk full, crash during recording, crash after recognition, cleanup/retention |
| Shared typing | Old autocomplete result after dictation, two writes to one target, generated text reentering input observation |

Use recorded fixtures to compare backend output without microphone variance, plus physical-device runs for startup, capture continuity, and external-app delivery. Separate pure reducers/accounting tests, adapter protocol tests, and platform integration tests. Inspecting reference tests does not establish that our implementation passes them.

## Measuring the intended improvement

Measure release builds on the same Mac and provider/model/locale, with the same audio and equivalent enabled features. Report cold and warm distributions rather than a single anecdote. Separate hotkey-to-first-buffer, first partial, release-to-provider-final, transformation time, insertion verification, and clipboard cleanup. The reference already records these distinctions in content-free traces. [Latency trace](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisper/Services/DictationLatencyTrace.swift:5)

Also measure idle wakeups, audio-worker CPU, allocations/copies, queue high-water marks, dropped frames, network bytes, disk writes, process physical footprint, and model/system-service memory. Compare recognition accuracy, quiet-word loss, false text on silence, and failed/duplicate insertions alongside speed. Wait for known teardown retention to settle before classifying retained engines as leaks.

Treat eliminating the 350 ms recognition polling wait, reusing format/converter state, segment-based preview updates, and filling PCM packets in one allocation as candidates. The Grok float encoder currently appends two bytes per sample, whereas the SDK WAV encoder fills an allocated payload. Their absolute cost may be small; profile before spending effort. [Grok PCM conversion](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisperPluginSDK/Plugins/XAIPlugin/XAIPlugin.swift:1005), [SDK WAV payload](/Users/sihochoi/Developer/Projects/typewhisper-mac/TypeWhisperPluginSDK/Sources/TypeWhisperPluginSDK/HostServices.swift:738).

The reference's package dependency list and source size are not its shipped footprint. Inspect the actual app bundle, linked libraries, resources, and loaded providers. Its launch CI also has a defined profile that excludes additional downloaded plugins/models; it does not substitute for dictation benchmarks. [Reference benchmark methodology](/Users/sihochoi/Developer/Projects/typewhisper-mac/docs/launch-performance.md:7)

Lean is most relevant to pure sample accounting, queue bounds, allowed state transitions, and transcript reduction. Proofs need an explicit connection to the executable implementation, including floating-point or atomic memory behavior where applicable. They cannot establish recognizer accuracy, microphone compatibility, actual energy use, or that an external editor accepted a paste.

Outstanding validation: target macOS/hardware baseline, Grok CLI OAuth speech access, intended signed distribution, maximum dictation duration, and temporary recovery retention. These do not prevent defining and implementing the core pipeline, but they affect the final support and resource claims.
