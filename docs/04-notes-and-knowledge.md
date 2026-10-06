# 04 · Notes and knowledge

> Part of [AI Meeting Note-Taker: System Design Notes](../README.md). Previous: [03 · Transcription pipeline](03-transcription-pipeline.md). Next: [05 · Agent interfaces](05-agent-interfaces.md).

A meeting note-taker that only produces per-meeting summaries is a transcription utility. The step that turns it into something people keep using is treating the accumulated notes as a personal knowledge base: linked, searchable, and able to answer "what did we quote them last time?" with a source. This document takes the position that **Markdown files on the user's machine are the source of truth** and works through what that implies for structure, linking, retrieval and cite-back.

## 1. Markdown as the source of truth

### 1.1 Why files

| Property | Hosted database as truth | Local Markdown as truth |
| --- | --- | --- |
| Readable without the vendor's app | No | Yes, by anything |
| Survives uninstalling the app | No | Yes |
| Survives a lapsed subscription | Depends on retention policy | Yes |
| Backed up by the user's existing tooling | No | Yes (Time Machine, any sync folder, git) |
| Diffable, scriptable | No | Yes |
| Vendor can change the schema under you | Yes | Only additively, in front matter |
| Multi-device sync | Built in | Has to be added (and is a privacy decision) |
| Full-text and semantic search | Server-side | Must be built locally (§3) |

The last two rows are the real cost of the file-first position, and the rest of this document is mostly about paying it.

### 1.2 Layout

A workable layout, one folder per note type and one file per unit:

```
Knowledge/
├── meetings/
│   ├── 2026-10-06-weekly-sync-with-acme.md
│   └── 2026-10-06-weekly-sync-with-acme.transcript.md
├── voice/
│   └── 2026-10-05-1432-idea-for-pricing-page.md
├── screenshots/
│   └── 2026-10-05-1120-dashboard-error.md     # OCR text + link to image
├── people/
│   └── Mei Lin.md                              # auto-created from mentions
├── topics/
│   └── Pricing.md
└── .index/                                     # rebuildable: search index, link graph, embeddings
```

Rules:

- **Everything under `.index/` is derived and can be deleted.** If the app can rebuild its index from the Markdown, the Markdown is really the source of truth. If it cannot, it is not.
- **Transcript and notes are separate files.** The transcript is immutable after finalisation ([doc 03 §6](03-transcription-pipeline.md#6-the-immutable-transcript-and-the-fallible-summary)); the notes file is editable by the user. Keeping them apart prevents an edit to the notes from ever touching the transcript.
- **Front matter carries structured metadata**; the body stays human-readable.

### 1.3 A meeting note

```markdown
---
id: m_01JA3Z2K9Q
type: meeting
title: Weekly sync with Acme
date: 2026-10-06T10:00:00+08:00
duration_min: 42
platform: zoom
participants: [Me, Mei Lin, Daniel Ho]
transcript: ./2026-10-06-weekly-sync-with-acme.transcript.md
languages: [en, zh-yue, zh-cmn]
---

# Weekly sync with Acme

## Summary
- Acme will move the pilot start to November because their finance sign-off slipped. [[^t:s0412]]
- Pricing discussed at HK$180k for the first year; Mei Lin to confirm whether that includes onboarding. [[^t:s0587]] [[^t:s0591]]

## My action items
- [ ] Send revised SOW with November start date by Friday. [[^t:s0633]]

## Waiting on
- [ ] Mei Lin: confirmation that HK$180k includes onboarding. [[^t:s0591]]

## Decisions
- No decision on the support SLA; parked to next week. [[^t:s0702]]

## Links
[[Mei Lin]] · [[Daniel Ho]] · [[Acme]] · [[Pricing]]
```

The `[[^t:s0412]]` tokens are **cite-back anchors**: references to transcript sentence identifiers. Rendered, they become a timestamp (`10:34`) that opens the transcript at that sentence and seeks the audio position if audio is retained. The notation is a convention, not a standard; what matters is that the anchor is stable, machine-parseable and survives a round-trip through any Markdown editor.

### 1.4 Export and deletion

If the files are the truth, "export" is nearly a no-op: zip the folder, or open it in another editor. The product should still offer a one-click export so that the promise is visible rather than implicit. Uninstalling the app must leave the folder untouched.

## 2. Bi-directional links

Wiki-style `[[links]]` are the lowest-friction way to connect notes, and they are plain text, so they survive the file-first rule.

- **Forward links** are written into the body (by the user or by the structuring pass, which can recognise people, companies and recurring topics).
- **Backlinks** are derived: the indexer scans all files, builds the graph and writes it to `.index/`. A person page therefore lists every meeting they appeared in without anyone maintaining it.
- **Auto-created entity pages** (people, organisations, topics) should be created lazily on first mention and be ordinary Markdown files, so the user can add to them.
- **Graph view** is a visualisation of the same index. It is more useful for noticing clusters ("these five meetings all touch Pricing") than for navigation.

Resolving a link by title is convenient but fragile under renames. Resolve by `id` in front matter first, falling back to title, and rewrite stale title links opportunistically.

## 3. Retrieval

Users ask two kinds of questions: *find* ("the meeting where Daniel mentioned the audit") and *answer* ("what did we quote Acme last time?"). Both need retrieval; the second additionally needs a language model and, critically, sources.

### 3.1 Index design

A local retrieval layer over a few thousand notes and transcripts is well within the capability of a laptop. A reasonable design:

- **Chunking.** Notes chunk by heading; transcripts chunk by a window of consecutive sentences (say 10 to 20) with overlap, each chunk remembering its sentence identifier range so a hit can be cited precisely.
- **Hybrid retrieval.** Lexical search (BM25 or similar) catches names, numbers and codes; vector similarity catches paraphrase. Union the candidate sets and rerank. Lexical search matters more than usual in mixed-language text, where a product name is the same token in any language.
- **Metadata filters.** Date range, participants, platform, note type. "What did Mei Lin say about pricing in September?" is a filter plus a query, and the filter does most of the work.
- **Multilingual embeddings.** Queries may be in English about a meeting held in Cantonese. The embedding model has to place both in the same space; monolingual embeddings quietly fail here.
- **Incremental indexing.** Watch the folder; reindex changed files. Because the user may edit Markdown in another editor, the indexer cannot assume it is the only writer.

Where the embedding and reranking computation runs is a product decision with the same honesty requirement as transcription: say what leaves the device. Keeping the index itself on the user's machine is the minimum for the "your notes are yours" promise to hold.

### 3.2 Answering with sources

The answer path is standard retrieval-augmented generation with one non-negotiable: **every claim in the answer cites a chunk, and every chunk resolves to a note heading or a transcript sentence range.**

```
Q: What did we quote Acme for the first year?

A: HK$180,000 for the first year, discussed in the weekly sync on 6 October 2026.
   Mei Lin was going to confirm whether onboarding is included; there is no
   record of that confirmation yet.

   Sources:
   [1] Weekly sync with Acme · 2026-10-06 · transcript 10:34–10:51
   [2] Weekly sync with Acme · 2026-10-06 · "Waiting on"
```

Rules that keep this trustworthy:

- If retrieval returns nothing relevant, the answer says so. It does not generalise from unrelated notes.
- The model is instructed that the retrieved chunks are the *only* admissible evidence, and the UI renders the sources as first-class links, not a footnote.
- "There is no record of X" is a legitimate and valuable answer. A knowledge base that cannot say "I don't have that" is a liability.

### 3.3 Inputs beyond meetings

The same store and index should accept other captured material: voice memos (transcribed through the same pipeline), screenshots (OCR'd, with the image stored alongside), and pasted text. The point of a personal knowledge base is that the user does not have to decide up front which tool a piece of information belongs to.

## 4. Cite-back end to end

Putting the pieces together, a single fact travels through the system with its provenance attached:

1. Capture produces a segment with offsets ([doc 02](02-capture-layer.md)).
2. Transcription produces sentences with stable identifiers and timestamps ([doc 03](03-transcription-pipeline.md)).
3. Structuring produces bullets, each carrying `[[^t:…]]` anchors to those sentences.
4. The note file stores bullets and anchors in plain text.
5. The index chunks notes and transcripts, keeping identifier ranges.
6. A question retrieves chunks and the answer cites them, resolving back to the same identifiers.
7. An agent tool call ([doc 05](05-agent-interfaces.md)) returns the same identifiers, so an agent's answer can also be checked.

If any step drops the identifier, provenance is lost from that point on. Treat the identifier as a required field everywhere.

> **Worked example: Audiee.ai**
> In [Audiee.ai](https://audiee.ai), meetings, voice notes and screenshots all land in a Knowledge store made of Markdown files on the user's Mac, with bi-directional links and a graph view, and semantic search runs over a local index. Every bullet in a meeting note carries a timestamp that opens the transcript at the corresponding sentence. Asking a question such as "what did we quote last time?" returns an answer with its sources attached. Notes can be exported in one click, deleting the app leaves the files in place, and user data is not used to train models. Knowledge storage is not capped on any plan, and reading, search and questions do not consume transcription hours.

---

Previous: [03 · Transcription pipeline](03-transcription-pipeline.md) · Next: [05 · Agent interfaces](05-agent-interfaces.md)
