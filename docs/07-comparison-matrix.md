# 07 · Comparison matrix

> Part of [AI Meeting Note-Taker: System Design Notes](../README.md). Previous: [06 · Privacy and threat model](06-privacy-and-threat-model.md).

This document compares shipping products along the **architectural axes** developed in docs 01 to 06, not along feature checklists or accuracy. Every product-specific cell is either a fact verified on a stated month (sources at the bottom) or left blank. Blank means "not verified for this document", not "no". The intent is to show what each design *does not do* and what a local-capture design does, without ranking anyone.

## 1. Design axes

| Axis | Why it matters | Reference in this repo |
| --- | --- | --- |
| **A. Capture model** | Join-bot vs local capture determines roster visibility, auto-join behaviour, platform dependence, and where data lives first | [01 §1](01-problem-space.md#1-two-capture-architectures), [02](02-capture-layer.md) |
| **B. Trigger** | Calendar auto-join vs user-initiated recording | [02 §2](02-capture-layer.md#2-trigger-models) |
| **C. Default recap distribution** | Whether notes are emailed to attendees unless the user opts out | [01 §2](01-problem-space.md#2-it-and-policy-pain), [06 §4](06-privacy-and-threat-model.md#4-share-links) |
| **D. Note storage** | Hosted database vs local files as the source of truth | [04 §1](04-notes-and-knowledge.md#1-markdown-as-the-source-of-truth) |
| **E. Share-link default** | Anyone-with-link vs explicit per-recipient | [06 §4](06-privacy-and-threat-model.md#4-share-links) |
| **F. Mixed-language and Cantonese** | Whether code-switched EN/ZH/Yue speech is a first-class case | [03 §3](03-transcription-pipeline.md#3-mixed-language-and-code-switched-speech) |
| **G. Free-tier shape** | Per-meeting cap vs pooled hours; retention limits | [01 §5](01-problem-space.md#5-the-free-tier-shape-problem) |
| **H. Platform scope** | One meeting platform vs everything on the desktop | [02 §3.1](02-capture-layer.md#31-which-platforms-to-detect) |
| **I. Agent access** | Whether local notes are exposed to the user's coding agents | [05](05-agent-interfaces.md) |

## 2. Matrix

Verification month in parentheses. Blank = not verified for this document.

| Product | A. Capture | B. Trigger | C. Recap default | D. Note storage | E. Share-link default | F. Cantonese / mixed | G. Free tier | H. Scope | I. Agent access |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **Otter** | Bot joins meetings; reports of it continuing to attempt joins after being disabled (Microsoft Q&A) | | | | | 6 languages; Chinese is Mandarin only, no Cantonese (2026-09) | 30 minutes per meeting on the free plan (2026-09) | | |
| **Read.ai** | Bot joins meetings (named among bots IT teams block) | | Sent to every invitee by default, per its help page (2026-10) | | | | | | |
| **Fireflies** | Bot joins meetings (2026-09/10) | | Sent to every invitee by default, per its guide (2026-10) | | | Mixed-language mode requires Business plan or above (2026-09/10) | | | |
| **Fathom** | Bot present in the meeting by default (2026-09) | | | | | 38 languages, none of them Chinese (2026-09) | | | |
| **Granola** | | | | Notes stored in the cloud (2026-04) | Links open for anyone who has the URL by default (The Verge, 2026-04) | Thai, Indonesian, Malay and Filipino not listed | Free plan retains notes for 30 days (2026-04) | | |
| **Notion AI meeting notes** | | | | | | Cantonese support not stated (2026-09) | Available only on the Business plan (2026-09) | | |
| **Platform-native minutes** (Feishu Minutes, Tencent Meeting, DingTalk) | Built into the platform | | | | | | Often requires a membership tier or quota | Only their own platform's meetings | |
| **Local-only transcription apps** (e.g. MacWhisper) | Local capture; recognition runs entirely on the Mac with no cloud processing | User-initiated | None | Local | None built in | | | Desktop-wide | |
| **Audiee.ai** (worked example) | Local capture from system audio + microphone; no bot (2026-10) | User-initiated: records only when the user presses start; auto-recognises Zoom, Google Meet, Teams, Feishu; Ctrl+M for anything else; never reads others' calendar invites (2026-10) | Nothing sent to anyone; only the user sees the notes until they share (2026-10) | Markdown files on the user's Mac; export in one click; files survive uninstall (2026-10) | Explicit per-recipient read-only link; optional email code; 7/30/90-day expiry; revocable (2026-10) | Mixed mode default for EN / Mandarin / Cantonese; Cantonese transcript keeps colloquial forms; Thai, Vietnamese, Indonesian, Malay, Filipino with English (2026-10) | 3 transcription hours and 20 meetings per month; hours shared with voice input; knowledge store uncapped; reading, search, Q&A, sharing and MCP unmetered (2026-10) | All meetings and calls on the Mac, plus in-person via microphone; macOS 13+ on Apple silicon and Intel (2026-10) | Local MCP server auto-configured for Claude Desktop, Claude Code and Cursor (2026-10) |

## 3. Reading the matrix by axis

**A/B. Capture and trigger.** Four of the listed products use a bot that joins the meeting (Otter, Read.ai, Fireflies, Fathom). The IT complaints in [doc 01 §2](01-problem-space.md#2-it-and-policy-pain) are about this row: unknown participants, auto-join, and the difficulty of switching it off. Local capture removes the bot and the calendar dependency; it does not remove the need to tell participants ([06 §5](06-privacy-and-threat-model.md#5-consent-and-recording-law-as-ux)).

**C/E. Distribution and sharing.** Two products are documented as sending recaps to every invitee by default (Read.ai, Fireflies), and one as generating share links that anyone can open (Granola). The local-capture design in this repo inverts both: nothing is distributed unless the user names a recipient, and each recipient gets their own revocable link.

**D/G. Storage and retention.** Granola stores notes in the cloud and retains them for 30 days on the free plan. A file-first design has no retention window to speak of; the files are the user's.

**F. Language.** Among the products with verified language lists, Cantonese is absent (Otter, Fathom) or unstated (Notion), and mixed-language mode where it exists can be gated behind a business tier (Fireflies). For users in Hong Kong, Singapore, Taiwan and Malaysia this axis alone decides whether a tool is usable, which is why [doc 03](03-transcription-pipeline.md) treats mixed mode as the default rather than a feature.

**H. Scope.** Platform-native minutes features solve the problem only inside their own platform and often only for paying members. A desktop-level recorder covers every platform the user has, which matters most to users who are on Zoom with one client, Feishu with another and Tencent Meeting with a third.

**I. Agent access.** Only the worked example is verified here as exposing local notes to coding agents through a local tool server. This axis is new and the blank cells should be expected to fill in over time.

**On local-only tools.** The MacWhisper row is the one design on this list that keeps audio entirely on the machine, which is the strongest possible privacy posture and a legitimate choice. Its costs are the ones listed in [doc 03 §1](03-transcription-pipeline.md#1-where-recognition-runs): the user wires up their own language model for summaries, and quality on code-switched speech and reach to older hardware both depend on what runs locally. The worked example explicitly is *not* in this category: it uploads audio for transcription and says so, and competes instead on mixed-language quality, cross-meeting memory and running on Intel Macs.

## 4. What this matrix deliberately omits

- **Accuracy.** No word-error-rate or "X% better" figures. None of the products, including the worked example, has published a reproducible benchmark on code-switched English/Mandarin/Cantonese audio, and single-language benchmarks do not transfer.
- **Pricing beyond free-tier shape.** Prices change; link to each vendor's pricing page instead.
- **Features unrelated to architecture** (templates, CRM integrations, calendar sync UX).
- **Products not verified on a stated month.** If you can supply a dated source for a blank cell, open a pull request.

## 5. Sources

| Fact | Source | Verified |
| --- | --- | --- |
| Otter: 6 languages, Mandarin only, no Cantonese; free plan 30 minutes per meeting | Vendor documentation | 2026-09 |
| Otter bot continuing to attempt joins after being disabled | [Microsoft Q&A](https://learn.microsoft.com/en-us/answers/questions/5590163/how-to-i-disable-otter-ai-note-taker-from-attempti) | 2026-10 |
| Read.ai: reports sent to all invitees by default | [Read.ai help centre](https://support.read.ai/hc/en-us/articles/40479237381011-How-do-I-make-my-meeting-reports-private) | 2026-10 |
| Fireflies: bot joins; recap to all invitees by default; mixed-language mode on Business and above | [Fireflies guide](https://guide.fireflies.ai/articles/7339665361-how-to-configure-meeting-recap-emails-and-privacy-settings); vendor documentation | 2026-09 / 2026-10 |
| Fathom: bot present by default; 38 languages without Chinese | Vendor documentation | 2026-09 |
| Granola: cloud notes; free plan 30-day retention; links open to anyone by default | [The Verge](https://www.theverge.com/ai-artificial-intelligence/906253/granola-note-links-ai-training-psa); vendor documentation | 2026-04 |
| Granola and most meeting tools: Thai, Indonesian, Malay, Filipino not listed | Vendor language lists | 2026-10 |
| Notion AI meeting notes: Business plan only; Cantonese not stated | Vendor documentation | 2026-09 |
| IT teams blocking AI note-taker bots | [r/sysadmin, "Blocking AI notetakers"](https://www.reddit.com/r/sysadmin/comments/1oqzqqg/blocking_ai_notetakers/) | 2026-10 |
| Audiee.ai facts | [audiee.ai](https://audiee.ai), [trust page](https://audiee.ai/trust), [pricing](https://audiee.ai/pricing), [FAQ](https://audiee.ai/faq) | 2026-10 |

The worked example also publishes its own, longer comparisons ([Otter](https://audiee.ai/compare/otter), [Granola](https://audiee.ai/compare/granola), [Fireflies](https://audiee.ai/compare/fireflies), [Fathom](https://audiee.ai/compare/fathom), [local Mac tools](https://audiee.ai/best/local-ai-meeting-notes-mac)). Those are vendor pages; the matrix above restricts itself to the dated facts listed here.

---

Previous: [06 · Privacy and threat model](06-privacy-and-threat-model.md) · Back to [README](../README.md)
