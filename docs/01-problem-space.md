# 01 · Problem space

> Part of [AI Meeting Note-Taker: System Design Notes](../README.md). Next: [02 · Capture layer](02-capture-layer.md).

This document frames the problem before any component is drawn. Four forces shape the architecture of an AI meeting note-taker: *how the audio gets captured*, *what IT departments will tolerate*, *which languages people actually speak in meetings*, and *who owns the resulting notes*. Most of the later design decisions are downstream of positions taken here.

## 1. Two capture architectures

Every AI note-taker needs the meeting audio. There are only two places to get it.

### 1.1 Bot-in-meeting

A service-side process joins the meeting as a participant. It appears in the roster (usually with a name like "X Notetaker"), receives the mixed audio stream the platform sends to every attendee, and streams it to the vendor's backend.

Properties that follow from this shape:

- **Visible participant.** Everyone in the call sees an extra attendee. Some find this reassuring (explicit signal that recording is happening); others, particularly in client-facing roles, find it unprofessional or alarming.
- **Calendar-driven.** To know which meetings to join, the bot reads the user's calendar, which usually means reading invitations *sent by other people*. Auto-join is the default in most products because it maximises coverage.
- **Platform-dependent.** The bot needs a way to join each platform (SDK, headless client, or partner API). Platforms can and do restrict this.
- **Server-side everything.** Audio, transcript and notes live on the vendor's servers from the first second. The user's device is just a viewer.
- **Distribution is cheap.** Because the vendor already knows all attendees from the invite, sending the recap to all of them is one checkbox, and it is often on by default.

### 1.2 Local capture

A desktop application on the user's machine taps the audio devices: the system output (what the user hears, i.e. the other participants) and the microphone (what the user says). No process joins the meeting.

Properties that follow:

- **Nothing in the roster.** The meeting platform is unaware of the recorder. The call looks identical to a call without a note-taker.
- **Trigger is the user's choice.** The app can detect that a meeting app is active and *offer* to record, or the user can press a shortcut, but nothing happens automatically on the strength of someone else's calendar invite.
- **Platform-agnostic by construction.** Anything that produces sound on the machine can be captured: video calls, VoIP, a browser tab, or an in-person conversation via the microphone. Platform-specific code is an *enhancement* (better speaker names, auto-start prompts), not a requirement.
- **Device is the system of record.** Transcripts and notes can be written to local files first. Whether anything is stored server-side becomes a separate, explicit decision.
- **Distribution is a deliberate act.** The app does not know who else was in the meeting unless the user says so, so there is no "send to all attendees" default to get wrong.

The rest of this repository is about building the second kind well. The first kind is well documented elsewhere and is, in practice, a different system (a fleet of meeting clients plus a media pipeline) rather than a desktop application.

### 1.3 A note on the word "stealth"

Local capture is sometimes marketed as invisible recording. That framing is wrong and the design should actively resist it. A bot in the roster is one *mechanism* for notifying participants, not the only one; the obligation to inform people that they are being recorded does not go away when the mechanism changes. A local-capture product should:

- remind the user, at least on first use, that participants must be told;
- make it trivial to say so (e.g. copyable one-line disclosure text);
- never describe the no-bot property as "others won't know".

Consent and regional recording rules are treated as a UX problem in [doc 06](06-privacy-and-threat-model.md).

## 2. IT and policy pain

The most common complaint about AI note-takers among English-speaking users is not accuracy; it is the bot. Three concrete, public examples:

- An r/sysadmin thread titled ["Blocking AI notetakers"](https://www.reddit.com/r/sysadmin/comments/1oqzqqg/blocking_ai_notetakers/) (400+ upvotes) in which administrators compare ways to keep third-party bots out of company meetings.
- A [Microsoft Q&A question](https://learn.microsoft.com/en-us/answers/questions/5590163/how-to-i-disable-otter-ai-note-taker-from-attempti) about a bot that kept attempting to join Teams meetings after the user believed they had disabled it.
- Vendor help pages documenting that meeting recaps are sent to every invitee by default ([Read.ai](https://support.read.ai/hc/en-us/articles/40479237381011-How-do-I-make-my-meeting-reports-private), [Fireflies](https://guide.fireflies.ai/articles/7339665361-how-to-configure-meeting-recap-emails-and-privacy-settings); defaults verified October 2026).

The administrator's objections cluster into three:

| Objection | Root cause in the bot architecture |
| --- | --- |
| "Unknown participants are joining our calls." | Bot appears in roster; often from a domain IT has never approved. |
| "It joins meetings nobody asked it to." | Calendar auto-join reads invitations and joins by default; disabling is per-user and easy to get wrong. |
| "Confidential recaps were emailed to external attendees." | Recap distribution defaults to the invite list. |

A local-capture design removes all three *mechanisms*. It does not remove the underlying policy question, which is whether meeting audio may be processed by a third-party service at all. The honest answer for most cloud-transcription products, including the worked example in this repo, is that audio *does* leave the device for transcription. A local-capture tool is therefore not a way around a company's rules; it is a way to have a narrower and more truthful conversation with IT: no bot, no auto-join, no recap blast, and here is exactly what is transmitted, to whom, and for how long. [Doc 06](06-privacy-and-threat-model.md) proposes the list of questions that conversation should cover.

## 3. Code-switching: the language problem that single-language ASR ignores

In Hong Kong, Singapore, Taiwan, Malaysia and much of South-East Asia, a single sentence in a work meeting routinely contains two or three languages:

> 我哋今日 confirm 咗個 timeline，但係 budget 嗰邊仲要 wait finance 回覆。

This is not a corner case; it is the dominant register of business speech in those markets. It breaks note-takers in specific ways:

1. **Language-locked models.** Many products ask the user to choose *one* meeting language. English words inside a Cantonese sentence then get forced into the nearest Cantonese syllables, or dropped. Choosing "English" produces the opposite failure.
2. **Cantonese is often simply absent.** As of September 2026, Otter supports six languages, with Chinese limited to Mandarin and no Cantonese; Fathom's 38-language list contains no Chinese; Notion AI meeting notes does not state whether Cantonese is supported. (Sources and months in [doc 07](07-comparison-matrix.md).)
3. **Register collapse.** Cantonese speech is full of colloquial function words and particles (佢哋, 唔係, 啦, 喎) that have no written-Mandarin equivalent. Pipelines that "clean up" transcripts into standard written Chinese destroy information that the speaker intended. The correct split is: the *transcript* stays verbatim, including particles; the *summary* is written in a formal register.
4. **Mixed-language mode gated behind price.** Where a mixed mode exists, it may be restricted to higher plans (Fireflies: Business tier and above, verified September/October 2026).
5. **South-East Asian languages are missing from the list.** Thai, Indonesian, Malay and Filipino, each typically mixed with English in practice, are absent from most meeting tools' language lists, including Granola's.

A pipeline that treats "mixed" as the default rather than an option has to be designed differently from the acoustic model up, which is why [doc 03](03-transcription-pipeline.md) spends most of its length on it. A related, concrete failure mode, dropped words in Cantonese transcription, is written up in the worked example's own notes: [Why Cantonese transcription drops words](https://audiee.ai/news/why-cantonese-transcription-drops-words).

The same users also have a smaller, daily version of this problem: desktop dictation that cannot follow a sentence that switches language mid-way, so English product terms inside a Chinese sentence come out garbled. Voice input shares a transcription pipeline with meetings, and therefore shares the fix.

## 4. Note ownership

Three questions determine whether a user actually owns their notes:

1. **Where is the canonical copy?** If the canonical copy is in a vendor database and the local view is a cache, the user has a licence to read their notes, not ownership. If the canonical copy is a file on disk, the vendor is a tool, not a custodian.
2. **What happens on churn?** Notes that disappear when a subscription lapses, or that are retained for only a short window on a free tier (Granola's free plan retains notes for 30 days, verified April 2026), are not owned.
3. **Can the notes be read without the app?** Markdown opens in anything. Proprietary blocks in a hosted editor do not.

There is a fourth, adjacent question users ask immediately after: *is my audio or transcript used to train models?* The answer should be a flat yes or no on a public page, not buried in a terms update.

Local-capture designs are not automatically better here (a desktop app can still sync everything to a cloud database), but they make the file-first option *available*, which bot architectures structurally do not. [Doc 04](04-notes-and-knowledge.md) takes the file-first position and works through its consequences for linking, search and agents.

## 5. The free-tier shape problem

A smaller design pressure that still affects architecture: how usage is metered. Per-meeting duration caps (Otter: 30 minutes per meeting on the free plan, verified September 2026) cut off the end of exactly the meetings that run long. Metering transcription *hours* per month instead, pooled across meetings and voice input, maps better to the actual cost driver (seconds of audio sent to the speech service) and does not truncate individual meetings. Whatever the metered unit is, reading, searching, asking questions of and sharing existing notes should not consume it; those operations do not touch the expensive part of the pipeline.

## 6. Summary of design positions taken in this repository

| Force | Position |
| --- | --- |
| Capture | Local capture from system audio + microphone; no participant joins the call |
| Trigger | User-initiated (shortcut or confirmed prompt); never from other people's calendar invites |
| Consent | No-bot is not stealth; first-use reminder; disclosure made easy |
| Language | Mixed-language as the default mode; verbatim transcript, formal-register summary |
| Transcript vs notes | Transcript immutable; notes derived and always cite transcript spans |
| Ownership | Markdown files on the user's machine are canonical; export is trivial; deletion of the app leaves the files |
| Distribution | Nothing sent to anyone by default; sharing is explicit and per recipient |
| Honesty about the cloud | Audio is uploaded for transcription; say so plainly, and say what is retained (nothing) |

> **Worked example: Audiee.ai**
> [Audiee.ai](https://audiee.ai) takes every position in the table above. On macOS it captures system audio and the microphone, auto-recognises Zoom, Google Meet, Microsoft Teams and Feishu, and records any other call (Tencent Meeting, DingTalk, WeChat calls, in-person via the microphone) when the user presses Ctrl+M. It records only when the user starts it and does not read anyone else's calendar invites. Transcription defaults to a mixed mode for English, Mandarin and Cantonese, keeping Cantonese colloquialisms and particles in the transcript while writing the summary in a formal register; Thai, Vietnamese, Indonesian, Malay and Filipino mixed with English are also handled. Notes and transcripts are Markdown files on the user's Mac; deleting the app leaves them in place, and no data is used for model training. Audio is sent to a cloud speech service for transcription and is not retained server-side. The free plan meters 3 transcription hours per month, shared between meetings and voice input, and 20 meetings per month; reading, search, questions, sharing and the MCP server are not metered ([pricing](https://audiee.ai/pricing)). Users whose IT has blocked meeting bots should note that Audiee is not a way around company policy: the IT conversation still needs to happen, and the [trust page](https://audiee.ai/trust) is written to support it.

---

Next: [02 · Capture layer](02-capture-layer.md)
