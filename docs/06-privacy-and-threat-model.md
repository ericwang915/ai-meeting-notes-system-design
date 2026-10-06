# 06 · Privacy and threat model

> Part of [AI Meeting Note-Taker: System Design Notes](../README.md). Previous: [05 · Agent interfaces](05-agent-interfaces.md). Next: [07 · Comparison matrix](07-comparison-matrix.md).

A meeting note-taker handles some of the most sensitive data a knowledge worker produces: candid conversations with clients, colleagues and candidates, in their own voices. This document lays out a threat model for the local-capture architecture, states precisely what leaves the device and what is retained, treats consent and recording law as a UX requirement, and argues for an honest security posture over an impressive-sounding one. Nothing here is legal advice.

## 1. Assets and actors

**Assets**, roughly in order of sensitivity:

1. Raw audio of meetings
2. Verbatim transcripts (with speaker names)
3. Derived notes (summaries, action items)
4. The cross-meeting knowledge base and its index
5. Share links and whoever holds them
6. Account identity and billing

**Actors who might want them:**

| Actor | Capability | Typical goal |
| --- | --- | --- |
| Thief or finder of the laptop | Physical access to disk | Opportunistic data |
| Share-link recipient (or whoever they forward it to) | Holds a URL | Read more or for longer than intended |
| Network observer | Sees traffic in transit | Intercept audio |
| Cloud processing provider (speech / language model) | Sees data during processing | Retain, mine, or train on it |
| The note-taker vendor itself | Operates the pipeline | Retain, mine, or train on it |
| A compromised agent / prompt injection | Via tool calls ([doc 05 §3.5](05-agent-interfaces.md#35-the-transcript-is-untrusted-input)) | Exfiltrate note content |
| Meeting participants | Were in the room | Object to being recorded |

## 2. Data flow: what leaves the device

The single most important table in this repository. Fill it in truthfully for any product, including your own.

| Step | Data | Leaves the device? | To whom | Retained there? |
| --- | --- | --- | --- | --- |
| Capture | Raw audio | No | — | Local spool until transcribed, then per policy |
| Transcription | Audio segments | **Yes** | Speech service (stateless, region-pinned) | **No** (returned text only; audio not stored) |
| Structuring | Transcript | **Yes** | Language model service (stateless) | **No** |
| Notes at rest | Markdown files | No | — | On the user's disk |
| Search / Q&A | Query + retrieved chunks | Depends on where the language model runs; state it | — | Should be no |
| Agent tools | Returned tool results | **Yes**, to the agent client's model provider | Chosen by the user's agent client | Governed by that provider |
| Sharing | The shared note | **Yes**, when the user shares | Share service + recipient | Until expiry or revocation |
| Telemetry | Crash reports, usage counts | Yes if enabled | Vendor | Per policy; must not contain content |

Two things follow:

- **"Local" and "private" are not synonyms.** In this architecture, notes are local; audio is processed in the cloud. A product should say both halves, not just the comfortable one. The forbidden phrasings are the absolute ones: "fully on-device", "audio never leaves your Mac". The accurate phrasing is: *notes are files on your Mac; audio is sent to a speech service for transcription; the service does not keep a copy.*
- **Retention is the lever.** Because the cloud steps are stateless, the cloud attack surface is limited to data in flight and in processing memory. There is no stored corpus of audio or transcripts to breach, subpoena, or leak. This is a materially stronger position than "encrypted at rest in our database", and it is only available if the vendor genuinely does not persist.

### 2.1 Training

State flatly whether user data is used to train models: audio, transcripts, notes, corrections. The answer for a product making the ownership promise in [doc 01 §4](01-problem-space.md#4-note-ownership) is no, for all of them, and it belongs on the trust page, not in a terms-of-service update.

## 3. Threats and mitigations

| Threat | Mitigation in this architecture |
| --- | --- |
| Lost or stolen laptop | Notes are ordinary files; rely on full-disk encryption (FileVault) rather than inventing app-level crypto. Keep the audio spool short-lived. |
| Interception in transit | TLS to the processing services; short-lived credentials issued to the app, no static API keys in the binary. |
| Processing provider retains or mines audio | Stateless, retention-free processing contractually and technically; name the region; do not name the provider publicly unless an evaluator requires it under NDA, since the provider can change and the promise should not depend on it. |
| Vendor retains transcripts | Same: no server-side storage of transcript or audio. The vendor's servers hold account and billing data, not content. |
| Over-broad sharing | Explicit, per-recipient links (§4); nothing sent to attendees by default. |
| Forwarded share link | Optional recipient verification by emailed code; expiry; revocation. |
| Prompt injection through the agent interface | Read-only tool surface, scoped access, call log ([doc 05 §3](05-agent-interfaces.md#3-privacy-boundaries)). |
| Recording someone without their knowledge | First-record reminder; disclosure helpers; no "stealth" framing anywhere in product or marketing (§5). |
| Covert auto-recording | No calendar auto-join; detect-and-offer requires per-app opt-in ([doc 02 §2](02-capture-layer.md#2-trigger-models)). |
| Telemetry leaking content | Telemetry schema excludes free text; off switch. |

## 4. Share links

Sharing is where a careful local design most often leaks, because it is the one feature that *has* to put content on a server. Design rules:

- **Explicit, named recipients.** The user types an email address; the recipient receives a read-only link. No "anyone with the link" default, and no automatic distribution to meeting attendees.
- **One link per recipient.** Revoking one person does not break the others, and a leaked link identifies who leaked it.
- **Optional recipient verification.** A one-time code sent to the recipient's email before the note opens, for sensitive shares. Not mandatory for every share, because friction kills adoption of the safe path.
- **Expiry by default.** Offer short windows (7, 30, 90 days). "Forever" should not be the default even if it is available.
- **Revocation at any time**, and a list of what is currently shared with whom.
- **No account required to read.** Forcing recipients to sign up pushes users back to copy-pasting the note into email, which is the least controlled channel of all.
- **Scope of the shared content.** Share the note; do not share the raw audio or the full transcript unless the user explicitly adds it.

Contrast with the defaults that generate most of the public complaints: recaps emailed to every invitee ([Read.ai](https://support.read.ai/hc/en-us/articles/40479237381011-How-do-I-make-my-meeting-reports-private), [Fireflies](https://guide.fireflies.ai/articles/7339665361-how-to-configure-meeting-recap-emails-and-privacy-settings); defaults verified October 2026), and share links that open for anyone who holds the URL ([The Verge on Granola](https://www.theverge.com/ai-artificial-intelligence/906253/granola-note-links-ai-training-psa), April 2026).

## 5. Consent and recording law as UX

Recording rules vary by jurisdiction (one-party versus all-party consent, sector rules, workplace rules) and are the user's responsibility. The product cannot make the user compliant, but its design can make compliance the easy path or the hard one.

What the design should do:

1. **Remind before the first recording** that participants must be told, with a link to regional guidance that cites official sources.
2. **Make disclosure cheap.** A copyable one-liner ("I'm taking AI notes on my laptop for my own records; let me know if you'd rather I didn't") in the languages the user works in.
3. **Make the recording state obvious** to the user at all times, so accidental recording is unlikely.
4. **Never trade on invisibility.** The absence of a bot is a property of the capture architecture, not a feature for avoiding disclosure. Marketing copy that says or implies "others won't know you're recording" is both an ethical problem and a liability for the user.
5. **Publish the regional guidance** the product links to, with sources and dates, and label it as information, not advice.

This matters more, not less, in markets such as Hong Kong, Singapore and Malaysia where users explicitly ask whether recording is legal before adopting a tool.

## 6. An honest security posture

Enterprise buyers ask for third-party audits and certifications. A small team building a local-capture tool may not have them, and the right response is to say so and to make the actual posture verifiable instead:

- Publish a trust page that answers the data-flow table in §2 in plain language: what is uploaded, where, retained for how long, used for training or not, and who the team is and which privacy law they operate under.
- Describe the operating jurisdiction and applicable law (for a Singapore team, PDPA) rather than implying a certification.
- Offer to disclose processing providers to an evaluator under NDA rather than publishing them.
- Do not use phrases like "enterprise-grade security" that assert a standard without evidence.

A clear "no, we have not been audited; here is exactly what we do" is more useful to a security reviewer than a vague yes, and it is the only one of the two that survives due diligence.

### 6.1 Questions a security reviewer will ask

A local-capture vendor should be able to answer these in writing before being asked:

1. Does anything join our meetings or appear in the participant list? (No.)
2. Does the app read calendars or invitations? (State yes/no and scope.)
3. What data leaves the device, to which services, in which region?
4. Is audio or transcript stored on your servers after processing? For how long?
5. Is any user data used to train models?
6. Where are notes stored at rest, and in what format?
7. How does sharing work, and what are the defaults?
8. Can an administrator or the user delete everything, and what does deletion cover?
9. What third-party audits or certifications do you hold? (If none, say none.)
10. Under which jurisdiction and privacy law do you operate?

> **Worked example: Audiee.ai**
> [Audiee.ai](https://audiee.ai) publishes its answers to §2 and §6.1 on its [trust page](https://audiee.ai/trust). In summary: notes and transcripts are Markdown files on the user's Mac; audio is uploaded to a speech service in Singapore for transcription and the servers do not retain audio or transcript copies; no user data is used to train models. Sharing follows §4: the user enters a recipient's email and they receive a read-only link, one per person, with optional email verification code, 7/30/90-day expiry and revocation at any time, and nothing is sent to meeting attendees by default. Before the first recording the app reminds the user to tell participants, and the website lists regional recording rules with official sources. Audiee is built by a small team in Singapore operating under PDPA; it has not undergone a third-party security audit, and it does not describe itself as fully on-device.

---

Previous: [05 · Agent interfaces](05-agent-interfaces.md) · Next: [07 · Comparison matrix](07-comparison-matrix.md)
