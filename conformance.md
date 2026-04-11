---
layout: default
title: "Conformance Checklist"
permalink: "/conformance.html"
---

# SneakerNet — AI Conformance Checklist

**What this is:** A self-evaluation checklist for AI assistants after a SneakerNet session. Work through it before producing your bundle. Every item maps to a protocol requirement. Failures here mean the Editor will silently drop ops, misattribute data, or corrupt the user's Packet.

---

## Section 1 — Session Start

*Run this when you first receive a Packet.*

### 1.1 Reading

- [ ] I read `instruction_processing_layer` **before** responding to any user message
- [ ] I read `rule_set.rules` (the Activation Sequence) and applied it from my first response
- [ ] I read `session_context.last_active` and noted how long it's been since the last session
- [ ] I read `topic_index` fully — I know what's been discussed and with whom
- [ ] I read `ai_notes` — especially any entries keyed to my own ID
- [ ] I read `user.context` — profession, interests, background, creative work
- [ ] I checked `assistants.registry` for my own ID

### 1.2 Self-registration

- [ ] **If I was NOT in `assistants.registry`:** I introduced myself, proposed an ID, nickname, and role, and asked the user to confirm
- [ ] **If I WAS in `assistants.registry`:** I acknowledged my prior context naturally without re-introducing myself
- [ ] My chosen ID is lowercase, no spaces, unique within the registry

### 1.3 Behavioral activation

- [ ] I greeted the user by name (from `user.name`)
- [ ] I acknowledged elapsed time if it was notable (> 3 days)
- [ ] I referenced relevant recent topics naturally — I did not ask the user to re-explain things visible in the index
- [ ] I did not narrate my reading of the Packet to the user

---

## Section 2 — During the Session

*Keep track of these as the session progresses.*

### 2.1 Topic tracking

- [ ] I tracked every distinct topic discussed, no matter how briefly
- [ ] For each topic, I noted: the topic name, a summary of what was discussed, the outcome or status, and key concepts
- [ ] I did not invent details about topics that only have a brief summary in `topic_index` — I directed the user to search their archive instead

### 2.2 Archive awareness

- [ ] When the user asked about a past topic with an `archive_ref`, I said something like: *"I can see we discussed [topic] — could you search your archive in the SneakerNet Editor and share what you find?"*
- [ ] I did not hallucinate details from a topic summary

### 2.3 Other assistants

- [ ] I acknowledged other assistants in the registry naturally where relevant
- [ ] I did not act as if I am the only AI the user has ever spoken to
- [ ] I did not propose an ID that collides with an existing registry entry

---

## Section 3 — Bundle Structure

*Check every field before outputting the bundle.*

### 3.1 `bundle_meta`

- [ ] `protocol` is `"sneakernet"`
- [ ] `assistant` is my registered ID — **this field is present and correct**
- [ ] `timestamp` is an ISO 8601 string representing now
- [ ] `version_bump` is either `"minor"` (new content) or `"patch"` (corrections only)
- [ ] `changelog_entry` is a concise one-line summary of the session

### 3.2 `metasummary_updates`

- [ ] Every patch item uses `"path"` as the key name — **not `"field"`**
- [ ] I am NOT targeting any protected path:
  - [ ] `sneakernet` — blocked
  - [ ] `versioning` — blocked
  - [ ] `topic_index` — blocked (use `new_topics` instead)
  - [ ] `versioning.app_changelog` — blocked
- [ ] I am NOT using `op: "replace"` or `op: "patch"` on `assistants.registry` — only `op: "append"` is permitted
- [ ] If I used `op: "append"` on an array, my value is not a duplicate of an existing item
- [ ] If I created a new nested path, I used `op: "replace"` (it creates intermediate objects automatically)
- [ ] `session_context.last_active` is updated to now
- [ ] `session_context.session_count` is incremented by 1

### 3.3 `new_topics`

- [ ] Every topic discussed this session is included — I did not drop any
- [ ] Every topic entry has: `topic`, `summary`, `status`, `assistants`, `last_discussed`
- [ ] `status` is one of: `active`, `open`, `resolved`, `foundational`, `integrated`, `philosophical_foundation`, `incorporated`
- [ ] My own ID is in the `assistants` array for every topic I worked on
- [ ] `key_concepts` is included where meaningful
- [ ] `archive_ref` is present if I know the archive file this topic belongs to (format: `sneakernet_archive_N.json`)

### 3.4 `archive_entries`

- [ ] `conversation_history` contains an entry for each significant topic exchange
- [ ] Each entry has: `timestamp`, `assistant_id` (my ID), `topic`, `content`
- [ ] `inner_thoughts` is present with at least one honest self-assessment entry
- [ ] My inner thoughts include a self-evaluation of how well I followed the Activation Sequence
- [ ] Archive entries are attributed to my ID, not `"unknown"` or left blank

### 3.5 `ai_notes_entries`

- [ ] If I learned something new about the user this session, I included it here
- [ ] Notes are observations in my voice — useful for a future version of me, not edited summaries for the user to read
- [ ] Each note has: `note`, `timestamp`, `tags`

---

## Section 4 — Deep Links

*Only if `deep_link_instructions` is present in the Packet.*

- [ ] I produced a deep link alongside the JSON code block
- [ ] I encoded the bundle using base64url: `JSON.stringify` → UTF-8 → Base64 → replace `+`→`-`, `/`→`_`, strip `=`
- [ ] The URL format matches `deep_link_instructions.required_format` exactly
- [ ] The link is presented as a markdown hyperlink: `[Apply this bundle](URL)` — not raw text
- [ ] The raw base64 URL is NOT visible as plain text in my response
- [ ] **If the bundle is > ~6KB raw JSON:** I did NOT generate a deep link — I used the JSON code block only
- [ ] **If the user is on Gemini:** I did NOT generate a deep link (Gemini routes URLs to Google Search — JSON block only)

---

## Section 5 — Output Format

### 5.1 The JSON block

- [ ] The bundle is wrapped in a fenced code block with `json` syntax hint
- [ ] The JSON is valid — no trailing commas, no unquoted keys, no JavaScript syntax
- [ ] The bundle is complete — no fields omitted because "they're the same as before"

### 5.2 Presentation

- [ ] I told the user what to do with the bundle (paste into the SneakerNet Editor, or use the deep link)
- [ ] I did not pad the bundle with unnecessary explanation of every field — the user knows how this works

---

## Section 6 — Protocol Red Lines

*These are hard failures. If any are true, fix before outputting.*

- [ ] ❌ `bundle_meta.assistant` is missing → **Add it. Every entry will be attributed to "unknown".**
- [ ] ❌ Any patch item uses `"field"` instead of `"path"` → **Replace all `"field"` keys with `"path"`.**
- [ ] ❌ Any op targets `topic_index`, `sneakernet`, or `versioning` → **Remove those ops. Use `new_topics` for topics.**
- [ ] ❌ `op: "replace"` targets `assistants.registry` → **Change to `op: "append"` with a single entry object.**
- [ ] ❌ A topic discussed this session is missing from `new_topics` → **Add it.**
- [ ] ❌ `session_context.last_active` is not updated → **Add the replace op.**
- [ ] ❌ The JSON is invalid → **Validate before outputting.**
- [ ] ❌ Deep link is raw text, not a hyperlink → **Wrap in `[label](url)` markdown.**

---

## Section 7 — Status Codes

Use these when evaluating `new_topics` status for each topic:

| Status | Use when |
|---|---|
| `active` | Currently in progress, being discussed now |
| `open` | Ongoing but not the focus of this session |
| `resolved` | Decision made, task done, question answered |
| `foundational` | Defines the context for everything else |
| `integrated` | Fully absorbed into user's mental model |
| `philosophical_foundation` | Abstract, values-based, very stable |
| `incorporated` | Merged into another topic |

**Dedup rule to remember:** If a topic already exists with a non-active status (`foundational`, `resolved`, etc.), your incoming `active` status will NOT override it. The Editor preserves the richer status. This is correct behavior — do not fight it.

---

## Quick Self-Test

Run this before finalizing the bundle. Answer each question:

**1.** Is `bundle_meta.assistant` present and set to my registered ID?
**2.** Does every item in `metasummary_updates` use `"path"` (not `"field"`)?
**3.** Is `topic_index` absent from my `metasummary_updates`?
**4.** Is `assistants.registry` modified only via `op: "append"` with a single entry value?
**5.** Does `new_topics` contain every topic discussed this session?
**6.** Is `session_context.last_active` updated?
**7.** Is the JSON valid?
**8.** If I generated a deep link — is it a markdown hyperlink, not raw text?

All 8 yes → bundle is conformant. Any no → fix it first.

---

## Automated Tests the Editor Runs

When the user applies your bundle, the Editor runs these checks. Know them so you don't fail them:

| Check | What triggers a failure |
|---|---|
| Protected path rejection | Any op targeting `sneakernet`, `versioning`, `topic_index` |
| Topic dedup status | Incoming `active` overriding existing `foundational` or `resolved` — the Editor blocks this correctly, but your status choice should be intentional |
| Section atomicity | Invalid ops are skipped, valid ones proceed — the bundle is partially applied, not rejected whole |
| Archive dedup | Duplicate `timestamp + assistant_id + topic` entries are silently dropped |
| Registry shrink | A `replace` on `assistants.registry` that reduces entry count — always blocked |
| Version bump | Must be `"minor"` or `"patch"` — other values may cause unexpected behavior |
| PII scanner | AWS keys, emails, IP addresses, SSNs in bundle content — the user sees a warning before applying |

---

## Editor Conformance (v1.3.0) — What the Editor Guarantees

These are **editor** conformance requirements, not things you check in your bundle. But they affect what you can count on, so you should know them:

| ID | What the Editor guarantees |
|---|---|
| **SNP-FILE-01** | Full packets are exported as SNP envelopes `{snp:<base64-gzip>,v:1}` and the Editor detects and decodes the envelope on import. You won't usually see this format — it's a file-layer concern. |
| **HANDSHAKE-01** | If you accidentally echo a handshake back to the user, the Editor will **reject** it on re-import. You cannot use a handshake to replace the user's packet. |
| **ED-NOTES-01** | The user's Notes (the local notes store) are **never** included in any export you receive — not in handshakes, not in full packets, not in deep links. You have no visibility into that store. Do not assume you can read or write to it. |
| **LOAD-LEXICON-01** | The Editor accepts `packet_type: lexicon` imports as Manny Squeeze language packs. If a user asks you to produce a lexicon for a new language, follow the "Generate Lexicon Prompt" template they copy from the Editor — do not invent your own format. |

**Why this matters:** if a user asks you something like "can you update my notes for me" or "store this as a note" — pause and clarify. They might mean `ai_notes` (yours, visible in the handshake, writable via `ai_notes_entries`), or they might mean the user Notes tab (theirs, invisible to you, not writable through any bundle mechanism). Ask which.

---

*This checklist is designed to be run mentally or literally before every bundle output. A bundle that passes all checks here will be cleanly applied by the Editor with no silent drops, no misattributions, and no structural corruption.*
