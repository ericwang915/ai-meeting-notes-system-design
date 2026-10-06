# 02 · Capture layer

> Part of [AI Meeting Note-Taker: System Design Notes](../README.md). Previous: [01 · Problem space](01-problem-space.md). Next: [03 · Transcription pipeline](03-transcription-pipeline.md).

The capture layer is the only part of the system that touches raw audio on the user's machine. It has to be boring: start reliably, never lose audio, never record when the user did not ask, and hand clean, timestamped segments to the transcription stage. This document covers audio sources, trigger models, platform detection, segmenting and spooling, and permissions.

## 1. Audio sources on a desktop

A meeting on a laptop has two audio paths:

| Path | Contains | Capture mechanism (macOS) |
| --- | --- | --- |
| **System output** | Remote participants, shared video audio, notification sounds | An audio tap on the output device or on specific processes. Modern macOS exposes system-audio capture through its screen/audio capture frameworks and, on recent versions, per-process audio taps. Older designs used a virtual audio device inserted into the output chain. |
| **Microphone** | The local user, plus room noise and (in person) everyone else in the room | Standard input-device capture. |

Capturing both as **separate streams**, rather than a pre-mixed stream, is the single most valuable decision at this layer:

- The two streams give you a free, perfectly accurate first cut of diarization: anything on the microphone stream is the local user; anything on the system stream is someone else. [Doc 03](03-transcription-pipeline.md) builds on this.
- Echo is a non-issue. The user's own voice is not on the system stream (the platform does not echo it back), and remote voices reach the microphone only as faint room bleed, which can be gated.
- Each stream can be level-normalised independently. Remote audio has already been compressed by the platform; the microphone has not.

For an in-person meeting there is only the microphone stream, and diarization has to fall back to acoustic methods.

### 1.1 Format decisions

- **Sample rate:** 16 kHz mono per stream is sufficient for speech recognition and a quarter of the bandwidth of 48 kHz stereo. Downsample at capture time.
- **Encoding for upload:** a speech-optimised codec at low bitrate keeps upload small enough to stream over a congested video call without competing with it. Keep the uncompressed buffer only as long as needed to encode.
- **Local retention of audio:** a policy decision, not a technical one. If the product promise is "notes are files on your machine", the audio should either be kept locally under the user's control with a visible setting, or discarded after transcription. It should not quietly accumulate.

## 2. Trigger models

How a recording starts determines who is in control. Three models exist in shipping products.

### 2.1 Calendar bot (auto-join)

The service reads the user's calendar, finds events with meeting links, and joins them. Coverage is maximal; control is minimal. Two consequences the user rarely sees up front: the service is reading invitations authored by other people, and the "off" switch has to work per meeting and per calendar. This model is intrinsic to bot architectures and is not available to a local-capture design, which has no way to join anything.

### 2.2 Press-to-record

The user presses a global shortcut or clicks a menu-bar item. Nothing happens otherwise. This is the baseline for a local-capture tool and the only trigger that works for every source (an unlisted VoIP app, a browser tab, an in-person conversation).

### 2.3 Detect-and-offer

The app notices that a meeting application has become active and *offers* to start, with a one-click confirm, or starts automatically if the user has opted into that for a specific app. The important design property is that the signal comes from *the user's own machine* (a process or window the user launched), not from other people's calendar data.

The recommended combination is 2.2 everywhere plus 2.3 for a short list of well-understood platforms. Auto-start without a prior opt-in is a mistake even when the detection is accurate: it is exactly the "it recorded a meeting I did not want recorded" complaint, relocated from the cloud to the desktop.

## 3. Platform detection as a design pattern

Platform detection is heuristics on top of public operating-system signals. It does not require, and should not involve, reverse-engineering a platform's network protocol or injecting into its process. Useful signals, roughly in order of reliability:

| Signal | How | Notes |
| --- | --- | --- |
| Process presence | Enumerate running applications by bundle identifier | Tells you the app is open, not that a call is active |
| Audio activity | The app is currently producing output / holding the microphone | The strongest "call in progress" signal; OS APIs expose which process owns an audio session |
| Window title | Accessibility or window-list APIs | Many clients put meeting name or state in the title; browser tabs (Google Meet) expose it via the tab title |
| Call-specific window | A known window class or size pattern appears when a call starts | Fragile across versions; use only as a hint |
| Roster / active-speaker UI | Accessibility tree of the client's participant panel | Platform-specific; see §3.2 |

Design rules:

1. **Degrade gracefully.** Every detector is a hint. If detection fails, press-to-record still works and the recording is identical in quality; only the metadata (platform name, speaker names) is poorer.
2. **Treat each platform as a plug-in.** A detector exposes `isCallActive()`, `meetingTitle()`, and optionally `activeSpeakerName()`. New platforms, or platforms whose UI changed, are a plug-in update, not a core change.
3. **Prefer OS-level signals over UI scraping.** Audio-session ownership and process presence survive UI redesigns; window titles and accessibility trees do not.
4. **Never require detection for correctness.** A platform that is not detected (a regional VoIP tool, a phone call bridged through the laptop) is still fully recordable.

### 3.1 Which platforms to detect

Zoom, Google Meet, Microsoft Teams and the regional equivalents (Feishu/Lark, Tencent Meeting, DingTalk, WeChat voice/video) cover the overwhelming majority of desk meetings. The regional ones matter more than Western designs usually assume: each tends to ship its own built-in minutes feature that works only inside that platform and often requires a membership tier, so a cross-platform local recorder is doing a job the platform will not do for its own users.

### 3.2 Speaker names from the platform

Some clients render the current speaker's name prominently enough that it can be read through accessibility APIs and attached to the system-audio stream in real time. Zoom is the canonical example, and a local-capture design can label remote speakers by name on Zoom without any acoustic speaker identification. Most other platforms do not expose a clean equivalent, and the honest design is: acoustic diarization assigns anonymous labels (Speaker 1, Speaker 2), the user renames each once, and the product remembers the voice-to-name mapping across future meetings. Products should not claim automatic speaker names on all platforms; they are a Zoom-shaped feature.

## 4. Segmenting, spooling and clocks

### 4.1 Segments

The transcription service receives audio in segments, not as one file at the end. Segment boundaries should fall on silence (voice-activity detection on each stream) with a maximum length (tens of seconds) so that a long monologue still produces timely transcript updates. Each segment carries:

- stream id (microphone or system),
- start and end offsets relative to the recording start (monotonic clock),
- the wall-clock time of recording start, recorded once.

Monotonic offsets plus a single wall-clock anchor avoid the classic bug where a laptop's clock adjusts mid-meeting and timestamps jump.

### 4.2 Local spool

Audio is written to a local spool before upload and deleted from the spool after the transcript for that segment has been received and persisted. This buys crash safety and tolerance of network outages: if the network drops for ten minutes, segments accumulate and are uploaded in order when it returns; if the app crashes, the next launch finds the spool and resumes. The spool is the only place raw audio exists on disk during a meeting and should be on an encrypted volume by default (which on macOS it will be, if the user has FileVault on).

### 4.3 Back-pressure

Upload bandwidth during a video call is contested. The uploader should run at low priority with a bounded queue and adaptive segment size, never blocking the capture thread. The capture thread's only job is to move samples from the OS into the spool.

## 5. Permissions and first-run UX

A local-capture app needs two OS permissions: microphone access and system-audio (screen-and-audio) capture. Both prompt the user with OS-controlled dialogs, and both are revocable in system settings. Design notes:

- Request each permission *at the moment it is needed* with an explanation, not at install.
- Show the recording state unambiguously (menu-bar indicator, dock badge). The OS will also show its own indicator for microphone use; do not try to hide it.
- **First-record consent reminder.** Before the first recording, show a short, dismissable reminder that participants must be told they are being recorded, with regional guidance linked. This is where the "no bot is not stealth" principle from [doc 01](01-problem-space.md) becomes a concrete screen.

## 6. Voice input as a second client of the same capture layer

Once a product has a reliable microphone capture path and a mixed-language transcription pipeline, system-wide dictation is a small addition: hold a key, speak into whatever text field has focus, release, and the transcript is inserted after filler-word removal and punctuation. It shares the capture code, the upload path and the metered transcription hours with meetings. The design consequence is that dictation and meetings should not be built as separate pipelines; they are two triggers on one.

> **Worked example: Audiee.ai**
> [Audiee.ai](https://audiee.ai) implements §2.2 plus §2.3: it auto-recognises Zoom, Google Meet, Microsoft Teams and Feishu, and the user presses Ctrl+M to record anything else, including Tencent Meeting, DingTalk, WeChat calls and in-person meetings through the Mac's microphone. It records only when the user starts it and does not read other people's calendar invitations. On Zoom, remote speakers are labelled by name automatically; on other platforms the user renames a speaker once and the name is remembered. The same capture path powers voice input: hold Ctrl in any text field to dictate with mixed-language recognition, filler words removed and punctuation added, and corrected terms are recognised correctly next time. Voice input shares the monthly transcription hours with meetings but does not count towards the monthly meeting quota. Before the first recording, Audiee reminds the user to tell participants. It runs on macOS 13 and later, on both Apple silicon and Intel Macs.

---

Previous: [01 · Problem space](01-problem-space.md) · Next: [03 · Transcription pipeline](03-transcription-pipeline.md)
