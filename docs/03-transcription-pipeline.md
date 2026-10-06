# 03 · Transcription pipeline

> Part of [AI Meeting Note-Taker: System Design Notes](../README.md). Previous: [02 · Capture layer](02-capture-layer.md). Next: [04 · Notes and knowledge](04-notes-and-knowledge.md).

This document covers the path from audio segments to a finished transcript and a set of derived notes: streaming versus batch recognition, mixed-language and code-switched speech, diarization, timestamps, and the boundary between the immutable transcript and the generated summary. It is deliberately vendor-neutral. Where the pipeline calls out to remote services they are referred to as "the speech service" and "the language model service".

## 1. Where recognition runs

Three placements are possible for the speech recogniser; they trade latency, cost, hardware reach and privacy posture differently.

| Placement | Strengths | Costs |
| --- | --- | --- |
| **Fully on the user's machine** | No audio leaves the device; works without a network | Needs a capable CPU/GPU and a large local model; quality on code-switched Cantonese and South-East Asian languages lags; summarisation still needs a language model, usually brought by the user; excludes older hardware |
| **Cloud speech service, stateless** | Best available models for mixed-language speech; runs on any laptop including older Intel machines; model improvements ship without an app update | Audio is uploaded; the user must trust the vendor's retention statement; region choice affects latency and data-residency conversations |
| **Hybrid** (local for live preview, cloud for the final pass) | Instant feedback plus final quality | Two recognisers to maintain; two sets of behaviour to explain |

The important discipline is to **state the placement plainly**. A product that uploads audio should say "audio is sent to a speech service for transcription; it is not stored after the transcript is returned", not "your data stays on your device". The two are different promises, and users and IT teams can tell.

For the cloud placement, the service should be:

- **Stateless per request.** A segment goes in, text comes out, nothing is persisted on the service side. This is both a privacy property and an operational simplification (no per-user storage to secure or delete).
- **Region-pinned.** Choose a region and say which. For an Asia-Pacific user base, a service in Singapore keeps round-trips short and gives a concrete answer to "where does my audio go".
- **Reached over TLS with short-lived credentials** issued to the desktop app, never a long-lived API key embedded in the binary.

## 2. Streaming versus batch

**Streaming** recognition returns partial hypotheses as audio arrives, then finalises each utterance. **Batch** recognition takes a complete segment or file and returns a final result.

| | Streaming | Batch |
| --- | --- | --- |
| Live transcript during the meeting | Yes | Only with short segments (quasi-streaming) |
| Context available to the recogniser | Only the past | The whole segment, which helps language detection and punctuation |
| Diarization | Harder; speaker turns arrive incrementally | Easier; the whole segment can be clustered |
| Failure handling | Reconnect logic, duplicate-partial suppression | Retry a segment |
| Cost per audio second | Typically higher | Typically lower |

For a note-taker whose main artefact is the *post-meeting* notes, segment-level batch is usually the right default: the capture layer already produces silence-bounded segments of tens of seconds ([doc 02 §4](02-capture-layer.md#4-segmenting-spooling-and-clocks)), so the transcript still appears during the meeting with a lag of a few seconds, and each segment gets full-context recognition. True streaming matters when the product sells live captions, which is a different feature with different accuracy expectations.

## 3. Mixed-language and code-switched speech

This is where most pipelines fail users in Hong Kong, Singapore, Taiwan and South-East Asia, and where most of the engineering effort in this layer should go.

### 3.1 The failure modes

1. **Single-language lock.** The user picks one language per meeting; everything else is forced into it. English technical terms inside a Cantonese sentence become phonetic nonsense.
2. **Utterance-level auto-detect.** The recogniser detects a language per utterance and switches models. Works for alternating monolingual sentences; fails inside a single sentence, which is where real code-switching happens.
3. **Dropped words.** When the acoustic model is uncertain about a stretch of speech in the "other" language, the cheapest output is to emit nothing. Cantonese speech mixed with English is especially vulnerable; a concrete write-up of this failure is [Why Cantonese transcription drops words](https://audiee.ai/news/why-cantonese-transcription-drops-words).
4. **Register normalisation.** Post-processing that rewrites Cantonese colloquial forms into standard written Chinese (佢哋 → 他們, 唔係 → 不是) and deletes particles (啦, 喎, 囉). This looks tidy and is a transcription error: it is not what was said.
5. **Script inconsistency.** Mandarin from a Taiwan speaker rendered in simplified characters, or the reverse, without the user having chosen.

### 3.2 Design response

- **Mixed mode is the default**, not a setting. The recogniser is asked to transcribe code-switched speech as spoken, keeping each word in its own language and script. A single-language mode remains available for the rare monolingual meeting where it is marginally better.
- **Verbatim transcript, formal summary.** The transcript keeps colloquial forms, particles and false starts. The summary, generated later from the transcript, is written in a formal register appropriate to the dominant language. These are two artefacts with two jobs, and the user can always see both.
- **Term memory.** Names, product terms and jargon the user corrects once are fed back as hints to subsequent recognition. For voice input this is an immediate quality multiplier; for meetings it mostly fixes proper nouns.
- **No accuracy claims without a published benchmark.** Public word-error-rate figures for mixed English/Mandarin/Cantonese speech are scarce and methodology-sensitive. Until a product publishes a reproducible benchmark on code-switched audio, it should not quote a percentage, and neither does this repository.

### 3.3 Languages beyond Chinese and English

The same mixed-mode discipline applies to Thai, Vietnamese, Indonesian, Malay and Filipino, each of which is spoken with heavy English mixing in professional settings. These languages are missing from most meeting tools' lists entirely, so support here is a coverage question before it is a quality question.

## 4. Diarization

Diarization answers "who spoke when". A local-capture design has three sources of evidence, best used in this order:

1. **Channel.** Microphone stream = the local user. System stream = everyone else. This is exact and free, and it already solves the most common two-person case.
2. **Platform active-speaker signal.** Where the meeting client exposes the current speaker's name through accessibility APIs (Zoom does), attach it to the system-stream audio at that time. This yields named remote speakers without any acoustic modelling. It is platform-specific and should be treated as an optional enrichment.
3. **Acoustic clustering.** On the system stream (or the whole microphone stream for in-person meetings), extract speaker embeddings per utterance and cluster. Output anonymous labels. Let the user rename a label once, and persist the embedding-to-name mapping so the same voice is recognised next meeting.

Design notes:

- Diarization errors are more annoying than recognition errors because they misattribute statements. Prefer under-segmentation with a clear "unknown speaker" label to confident wrong names.
- Do not claim automatic speaker names on every platform. Say exactly where they are automatic and that elsewhere the user renames once.
- Store speaker labels as a layer *over* the transcript, not baked into the text, so a rename rewrites metadata, not the transcript.

## 5. Timestamps

Every transcript unit carries timing, and timing is what makes everything downstream verifiable.

- **Granularity.** Segment-level timestamps are the minimum; word-level timestamps, when the speech service returns them, allow precise cite-back and better alignment with speaker turns.
- **Reference frame.** Store offsets from recording start (monotonic) plus one wall-clock anchor per recording ([doc 02 §4.1](02-capture-layer.md#41-segments)). Display in meeting-relative time (`00:12:34`) by default; wall-clock on demand.
- **Stable identifiers.** Each transcript sentence or segment gets an identifier that never changes after the transcript is finalised. Notes, search results and agent tool outputs refer to these identifiers, so a reference made today still resolves next year.

## 6. The immutable transcript and the fallible summary

The two artefacts this pipeline produces have different epistemic status, and the architecture should enforce the difference.

**The transcript** is a record of what the speech service heard. It can contain recognition errors, but it is never *rewritten* by a language model: no "cleaning up", no re-phrasing, no deletion of hedges or particles. Once finalised it is append-only (speaker renames and user corrections are layered metadata with their own history).

**The notes** (summary, decisions, the user's action items, items the user is waiting on from others) are generated by a language model from the transcript. Generation can hallucinate: it can invent a decision that was discussed but not made, merge two speakers' positions, or produce bullets so generic they could describe any meeting. Two mitigations are architectural rather than prompt-level:

1. **Mandatory cite-back.** Every generated bullet must carry at least one transcript identifier, and the UI renders it as a link that jumps to that sentence with audio position. A bullet without a citation is dropped or flagged. This does not prevent hallucination; it makes it *checkable in one click*, which is the realistic goal.
2. **Transcript as ground truth in the UI.** The summary is presented as a view over the transcript, visually subordinate to it, not as the primary record. When a user disputes a bullet, the resolution path is "read the cited sentence", not "trust the model".

The language model service, like the speech service, should be stateless and retention-free: the transcript is sent, the structured notes come back, nothing is stored. The notes are then written to the user's local files ([doc 04](04-notes-and-knowledge.md)).

### 6.1 Latency budget

Users judge a note-taker by how soon after the meeting the notes exist. Because segments were transcribed during the meeting, the only post-meeting work is the structuring pass over an already-complete transcript, which should finish within about a minute for a typical meeting. Anything that forces the user to wait while a full re-transcription runs has made the wrong call in §2.

## 7. Metering

The expensive operations in this pipeline are the two service calls, and the dominant one is speech recognition, whose cost scales with seconds of audio. The natural metered unit is therefore **transcription hours per month**, pooled across every feature that produces audio (meetings and voice input). Operations that do not call the speech service, such as reading, searching, asking questions over existing notes, sharing, and agent tool calls, should be free of metering. Per-meeting duration caps are a worse fit: they truncate exactly the long meetings where notes matter most.

> **Worked example: Audiee.ai**
> [Audiee.ai](https://audiee.ai) sends audio segments to a speech service in Singapore for transcription and the finished transcript to a language model service to draft the notes; neither service retains the audio or the transcript, and neither is used for model training. Mixed mode is the default for English, Mandarin and Cantonese, with Cantonese transcripts keeping colloquial forms and particles and the summary written in a formal register; Thai, Vietnamese, Indonesian, Malay and Filipino mixed with English are also supported. Diarization follows §4: on Zoom remote speakers are named automatically; on other platforms the user renames a speaker once and it is remembered. Within about a minute of the meeting ending the notes contain the summary, the user's own action items and the items they are waiting on from others; every bullet carries a timestamp that jumps to the corresponding transcript sentence, and the transcript itself is never rewritten. Usage is metered in transcription hours shared between meetings and voice input (3 hours and 20 meetings per month on the free plan; paid plans at [audiee.ai/pricing](https://audiee.ai/pricing)); reading, search, questions, sharing and MCP access are not metered. No accuracy percentage is quoted because a mixed-language benchmark has not been published.

---

Previous: [02 · Capture layer](02-capture-layer.md) · Next: [04 · Notes and knowledge](04-notes-and-knowledge.md)
