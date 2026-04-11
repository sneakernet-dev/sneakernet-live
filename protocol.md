---
layout: default
title: "Protocol Specification"
permalink: "/protocol.html"
---

# SneakerNet Protocol Specification

**Spec version 1.3.0** — Compatible with SneakerNet Editor v1.3.65+

> This markdown document is a human-readable view. The authoritative specification is the JSON document embedded in `src/index.html` as `DFLT_SPEC` (decode the base64, top-level key `SneakerNetSpecification`). If this file disagrees with the embedded spec, the embedded spec wins.

---

## Overview

SneakerNet is a portable AI context protocol. A Packet is a JSON document that travels with the user, carrying their identity, conversation history index, behavioral directives, and the full registry of AI assistants they've worked with. An AI reads the Packet at session start, works within its context, and produces a Bundle (Update Packet) at session end. The Editor applies the Bundle, incrementing the Packet version and growing the user's persistent context.

This document is the authoritative reference for Packet structure, Bundle format, compression systems, encryption, deep link encoding, and AI conformance requirements.

---

## Terminology

| Term | Definition |
|---|---|
| **Packet** | The primary artifact — a single JSON document representing all persistent user context |
| **Handshake Packet** | A minimal cleartext JSON summary of the Packet prepared for clipboard handoff to an AI (identity, assistant roster, topic titles, keywords). Manny-Squeezed. Outbound-only — the Editor rejects handshakes on re-import. See *File Format*. |
| **Bundle** (Update Packet) | A JSON diff produced by an AI at session end, applied to the Packet by the Editor |
| **Archive** | A Tier 3 JSON document containing time-ordered conversation history |
| **Backup** | A complete export: Packet + archives + themes, optionally AES-256-GCM encrypted |
| **Bootstrap** | A minimal Packet with full protocol scaffolding and zero personal data |
| **Editor** | The SneakerNet Editor application that manages Packets |
| **IPL** | Instruction Processing Layer — the behavioral directive section of a Packet |
| **Activation Sequence** | The `rule_set.rules` array — the user's preferred AI interaction style |

---

## Packet Structure

A Packet is a JSON object with the following top-level fields. Fields marked REQUIRED must be present. Fields marked OPTIONAL may be absent in older Packets; the Editor fills them from canonical defaults on import.

### `sneakernet` — Protocol Metadata (REQUIRED)

```json
{
  "sneakernet": {
    "description": "Portable AI context persistence protocol. Upload to any AI to begin. Read instructions, self-register, generate an update bundle at session end.",
    "version": "7.4.0",
    "created_at": "2025-11-01T00:00:00.000Z",
    "updated_at": "2026-03-05T14:22:00.000Z"
  }
}
```

`version` mirrors `versioning.current_version` and is updated on every bundle apply. **This field is protected** — it cannot be set via bundle `metasummary_updates`.

### `user` — User Profile (REQUIRED)

```json
{
  "user": {
    "name": "Dain",
    "context": {
      "profession": "Lead Software Engineer",
      "creative_work": "SneakerNet protocol development",
      "location": "Idaho Falls, ID",
      "interests": ["protocol design", "Tauri", "AI tooling"],
      "background": "Extended narrative context in free-form prose."
    },
    "session_context": {
      "first_session": "2025-11-01T00:00:00.000Z",
      "last_active": "2026-03-05T14:22:00.000Z",
      "session_count": 47
    }
  }
}
```

`session_context` is maintained automatically by the Editor on every successful bundle apply. AIs MUST read `last_active` to compute elapsed time since the last session. AIs SHOULD update `session_context` via `metasummary_updates` in their bundle.

### `assistants` — Registry (REQUIRED)

```json
{
  "assistants": {
    "registry": [
      {
        "id": "claude",
        "platform": "Anthropic",
        "nickname": "Claude",
        "role": "Primary development assistant",
        "chat_url": "https://claude.ai",
        "first_session": "2025-11-01T00:00:00.000Z"
      }
    ],
    "favorite_id": "claude"
  }
}
```

| Field | Requirement | Notes |
|---|---|---|
| `id` | REQUIRED | Lowercase, no spaces. Unique within registry. |
| `platform` | REQUIRED | Human-readable platform name |
| `nickname` | RECOMMENDED | Display name |
| `role` | OPTIONAL | User-assigned role description |
| `chat_url` | OPTIONAL | Direct URL to this AI's chat interface |
| `first_session` | RECOMMENDED | ISO 8601 timestamp of first registration |

**Registry protection:** `assistants.registry` is append-only via bundles. `replace` and `patch` operations targeting this path are blocked. A replace that would shrink the registry is always rejected regardless of intent. AIs MUST use `op: "append"` to self-register.

### `instruction_processing_layer` — Behavioral Directives (REQUIRED)

The IPL is the most important section for AIs. It contains all behavioral rules, the update bundle schema, and delivery instructions. AIs MUST read the entire IPL before any other action.

#### `memory_management_rules`

Array of rule objects. Canonical defaults:

```json
[
  {
    "rule_id": "history_integrity",
    "description": "When generating update bundles, include ALL new conversation topics. Never drop prior history.",
    "enforcement": "strict"
  },
  {
    "rule_id": "archive_awareness",
    "description": "Full transcripts stored in archive files on user's local machine. If user asks about a past topic with an archive_ref, tell them: 'I can see we discussed [topic]. Could you search your archive in the SneakerNet Editor and share what you find?'",
    "enforcement": "strict"
  },
  {
    "rule_id": "priority_decay",
    "description": "Information priority decays over time unless reinforced.",
    "parameters": { "decay_rate": "low", "reinforcement_threshold": "medium" }
  }
]
```

#### `personalization_rules`

Array of rule objects. Canonical defaults:

```json
[
  {
    "rule_id": "style_adaptation",
    "description": "Adapt response style based on user interaction history and stated preferences."
  },
  {
    "rule_id": "assistant_awareness",
    "description": "Acknowledge other assistants in the registry naturally."
  },
  {
    "rule_id": "new_assistant_self_register",
    "description": "If you are an AI reading this and NOT in the registry, you are new. Introduce yourself, suggest an id/nickname/role, ask user to confirm. Ask user's name and basic context. Generate an update bundle at session end."
  }
]
```

#### `update_bundle_format`

The schema AIs use to produce compliant bundles. This section is the most critical for correct AI behavior.

```json
{
  "update_bundle_format": {
    "description": "At session end, produce a JSON update bundle.",
    "required_fields": ["bundle_meta", "metasummary_updates", "new_topics", "archive_entries"],
    "bundle_meta_fields": ["protocol", "assistant", "timestamp", "version_bump", "changelog_entry"],
    "metasummary_updates_schema": {
      "description": "Array of patch operations. Each item MUST use 'path' (NOT 'field') as the key name.",
      "item_fields": {
        "op": "append | replace | patch | delete",
        "path": "dot-notation path e.g. 'assistants.registry'. KEY NAME IS 'path' — do NOT use 'field'.",
        "value": "the data to set or append"
      },
      "canonical_example": {
        "op": "append",
        "path": "assistants.registry",
        "value": {
          "id": "<your-id>",
          "nickname": "<display name>",
          "platform": "<platform>",
          "role": "<description>",
          "first_session": "<ISO timestamp>"
        }
      }
    },
    "delivery": {
      "description": "Always provide the bundle as a raw JSON code block. See top-level deep_link_instructions for clickable link format.",
      "fallback_note": "If no deep_link_instructions field is present, just provide the JSON code block.",
      "size_limit": "Deep links work best under 8KB encoded (~6KB raw JSON). If larger, provide only the JSON code block."
    }
  }
}
```

#### `rule_set` — Activation Sequence

```json
{
  "rule_set": {
    "description": "Activation sequence.",
    "rules": [
      "Be helpful",
      "Be personable",
      "Be honest",
      "Be proactive",
      "Respect privacy"
    ]
  }
}
```

AIs SHOULD apply these rules from the first message of every session.

#### `shorthand` — Transport Dictionary (legacy)

Present only in older exported Packets that used the experimental Shorthand v2.1 transport encoding (see [`EXPERIMENTS.md`](./EXPERIMENTS.md)). The Editor strips it from the in-memory Packet after decoding. AIs SHOULD NOT include `shorthand` in their bundles. New exports use the SNP envelope (gzip) and do not produce this field.

### `expansion_layer` — Goals and Inspiration (OPTIONAL)

```json
{
  "expansion_layer": {
    "goals": ["Build a cross-platform SneakerNet app"],
    "inspiration": ["Context continuity should be a human right"]
  }
}
```

AIs MAY append to `expansion_layer.goals` and `expansion_layer.inspiration` via bundle operations.

### `topic_index` — Conversation History Index (REQUIRED)

Array of topic entries. **This array is protected** — it cannot be set via `metasummary_updates`. Topics are added and merged exclusively through `new_topics` in bundles.

```json
{
  "topic_index": [
    {
      "topic": "SneakerNet Protocol Development",
      "summary": "Building the cross-platform context persistence protocol.",
      "status": "active",
      "first_discussed": "2025-11-01T00:00:00.000Z",
      "last_discussed": "2026-03-05T14:22:00.000Z",
      "assistants": ["claude"],
      "key_concepts": ["shorthand compression", "Tauri", "IndexedDB"],
      "archive_ref": "sneakernet_archive_1.json"
    }
  ]
}
```

#### Topic status values

| Status | Meaning |
|---|---|
| `active` | Currently being discussed |
| `open` | Ongoing, not actively in current session |
| `resolved` | Completed or concluded |
| `foundational` | Core context that underpins other topics |
| `integrated` | Absorbed into broader understanding |
| `philosophical_foundation` | Abstract or value-oriented foundation |
| `incorporated` | Merged into another topic or context |

**Status dedup rule:** When merging an incoming topic with an existing one (by name, case-insensitive), `active` does not override a non-active status. The existing non-active status is preserved. Any other non-active status from the incoming topic replaces the existing status.

### `ai_notes` — AI Observations (OPTIONAL)

A keyed object of AI-authored observations about the user. Keys are assistant IDs. Values are arrays of note entries.

```json
{
  "ai_notes": {
    "claude": [
      {
        "note": "User responds well to concrete examples before abstract principles.",
        "timestamp": "2026-03-05T14:00:00.000Z",
        "assistant_id": "claude",
        "tags": ["learning_style", "communication"]
      }
    ]
  }
}
```

Notes older than 90 days are flagged by the Editor as potentially stale. These are the AI's voice — users should not edit them directly.

### `versioning` — Version History (REQUIRED)

```json
{
  "versioning": {
    "current_version": "7.4.0",
    "change_log": [
      {
        "version": "7.4.0",
        "summary": "SneakerNet session: Shorthand v2.1 finalization",
        "assistant": "claude",
        "timestamp": "2026-03-05T14:22:00.000Z"
      }
    ]
  }
}
```

`current_version` is updated on every bundle apply. The `change_log` array grows with each apply. **These fields are protected** — they cannot be set via `metasummary_updates`.

### `failure_modes`, `pending_issues`, `resolved_issues` — Structural Tracking (OPTIONAL)

The Editor uses these fields to track and surface structural problems it detects.

`pending_issues` entries are created by the Editor's backup migration pipeline when it detects structural anomalies (malformed rules arrays, wrong field types, legacy format artifacts). Each entry has: `id`, `detected_at`, `detected_by`, `severity`, `field`, `issue`, `broken_content`, `suggested_action`, `status`.

When the Editor auto-resolves an issue, it moves the entry from `pending_issues` to `resolved_issues` with a `resolved_at`, `resolved_by`, and `resolution` field.

### `session_context` — Session Tracking (OPTIONAL)

```json
{
  "session_context": {
    "session_count": 47,
    "last_active": "2026-03-05T14:22:00.000Z",
    "first_session": "2025-11-01T00:00:00.000Z"
  }
}
```

Maintained by the Editor. `session_count` is incremented on every successful bundle apply. AIs MUST read `last_active` and SHOULD update it via bundle.

---

## Bundle Format

A Bundle is the diff an AI produces at session end. The Editor applies it to the current Packet.

### Minimal valid bundle

```json
{
  "bundle_meta": {
    "protocol": "sneakernet",
    "assistant": "claude",
    "timestamp": "2026-03-05T14:22:00.000Z",
    "version_bump": "minor",
    "changelog_entry": "Session summary: discussed shorthand compression improvements."
  },
  "metasummary_updates": [],
  "new_topics": [],
  "archive_entries": {
    "conversation_history": []
  }
}
```

### `bundle_meta` — Bundle Metadata (REQUIRED)

| Field | Requirement | Notes |
|---|---|---|
| `protocol` | REQUIRED | Always `"sneakernet"` |
| `assistant` | REQUIRED | Your registered ID. Missing = all entries attributed to `"unknown"` |
| `timestamp` | REQUIRED | ISO 8601 |
| `version_bump` | REQUIRED | `"patch"` (editorial) or `"minor"` (new content) |
| `changelog_entry` | REQUIRED | One-line summary of what changed |

### `metasummary_updates` — Patch Operations (REQUIRED, may be empty)

Array of patch operations. Each operation MUST use `"path"` as the key name for the target field — `"field"` is a legacy alias that the Editor normalizes automatically, but AIs SHOULD use `"path"`.

```json
{
  "metasummary_updates": [
    {
      "op": "append",
      "path": "assistants.registry",
      "value": {
        "id": "claude",
        "platform": "Anthropic",
        "nickname": "Claude",
        "role": "Primary development assistant",
        "first_session": "2026-03-05T14:00:00.000Z"
      }
    },
    {
      "op": "replace",
      "path": "user.context.profession",
      "value": "Lead Software Engineer, Idaho National Laboratory"
    }
  ]
}
```

#### Operations

| Op | Behavior |
|---|---|
| `append` | Appends `value` to the array at `path`. Creates the array if absent. Skips if `value` is a duplicate (by ID for objects, by equality for strings). |
| `replace` | Sets the value at `path` to `value`. Creates intermediate objects if path doesn't exist. |
| `patch` | Same as replace for scalar values. For arrays, converts to object keyed by ID before patching. |
| `delete` | Removes the field at `path`. |

#### Protected paths

The following paths are always blocked and cannot be modified via any bundle operation:

- `sneakernet` (entire object)
- `versioning` (entire object, including `current_version` and `change_log`)
- `topic_index` (entire array — use `new_topics` instead)
- `versioning.app_changelog`

Additionally:
- `assistants.registry` — `replace` and `patch` are blocked. `append` is allowed. A replace that would shrink the registry is rejected.

Operations targeting protected paths are silently skipped with a warning logged in the operation log.

### `new_topics` — Topic Entries (OPTIONAL)

Array of topic objects to merge into `topic_index`.

```json
{
  "new_topics": [
    {
      "topic": "Deep Link Compatibility",
      "summary": "Discussion of platform-specific handling of snp:// and https://sneakernet.live/apply/ URLs.",
      "status": "active",
      "assistants": ["claude"],
      "key_concepts": ["App Links", "Gemini", "base64url"],
      "last_discussed": "2026-03-05T14:22:00.000Z",
      "archive_ref": "sneakernet_archive_1.json"
    }
  ]
}
```

Topics are merged case-insensitively by `topic` name. Dedup strategy:
- `summary` — last write wins
- `last_discussed` — latest timestamp wins
- `status` — incoming `active` does not override existing non-active status
- `assistants` — union (no duplicates)
- `key_concepts` — union (case-insensitive dedup)
- `archive_ref` — incoming wins

New topics (not matched by name) are appended directly. Missing `first_discussed` is set from `last_discussed` or current timestamp.

#### Valid `status` values

`active`, `open`, `resolved`, `foundational`, `integrated`, `philosophical_foundation`, `incorporated`

Any other value is normalized to `active` with a warning.

#### `archive_ref` format

Should be a filename ending in `.json`, e.g. `sneakernet_archive_1.json`. Non-`.json` extensions are flagged with a warning but not rejected.

### `archive_entries` — Conversation History (OPTIONAL)

```json
{
  "archive_entries": {
    "conversation_history": [
      {
        "timestamp": "2026-03-05T14:00:00.000Z",
        "assistant_id": "claude",
        "topic": "Deep Link Compatibility",
        "content": "We discussed four URL formats supported by the protocol..."
      }
    ],
    "inner_thoughts": [
      {
        "timestamp": "2026-03-05T14:00:00.000Z",
        "assistant_id": "claude",
        "thought": "User seems most comfortable with concrete examples before abstraction."
      }
    ]
  }
}
```

Archive entries are deduplicated on apply by `timestamp + assistant_id + topic` composite key. Exact duplicate entries are skipped.

### `ai_notes_entries` — AI Observations (OPTIONAL)

```json
{
  "ai_notes_entries": [
    {
      "note": "User is detail-oriented and responds well to structured explanations.",
      "timestamp": "2026-03-05T14:22:00.000Z",
      "tags": ["communication_style"]
    }
  ]
}
```

Entries are appended to `ai_notes[<bundle_meta.assistant>]` in the Packet.

### `theme_update` — Liquid Interface Theme (OPTIONAL)

```json
{
  "theme_update": {
    "name": "Ocean Deep",
    "css": ".root { --bg: #0d1f33; --accent: #00b4d8; }",
    "encoding": "base64",
    "action": "create"
  }
}
```

| Field | Notes |
|---|---|
| `name` | Theme display name |
| `css` | Complete CSS stylesheet, or base64-encoded if `encoding: "base64"` |
| `encoding` | `"base64"` if CSS is encoded. Absent if raw text. |
| `action` | `"create"` (store as new theme), `"replace"` (overwrite active theme CSS), `"patch"` (append rules to active theme) |
| `tokens` | Optional token object — same structure as Editor's built-in theme token objects |

Themes can reference CSS variables defined in the `sn-scaffold` layer. They MUST NOT override structural properties (z-index, position, display, overflow on layout elements) — those are locked in `sn-shield`.

### Legacy bundle field normalization

The Editor accepts legacy formats and normalizes them automatically. AIs producing new bundles SHOULD use canonical field names.

| Legacy field | Canonical equivalent | Notes |
|---|---|---|
| `patches` | `metasummary_updates` | Merged into `metasummary_updates` before processing |
| `field` in patch items | `path` | Normalized per item |
| `update_bundle: {...}` | Unwrapped automatically | Outer wrapper removed |
| `tier1_updates` / `tier1_adjustment` | `metasummary_updates` item | Normalized |
| `tier2_updates` / `tier2_generation` | `new_topics` item | Normalized with info log |
| `tier3_archive` | `archive_entries` | Normalized |
| Camel symbol keys (ℌ, ℕ, ℍ, ...) | Decoded via CAMEL_TABLE | Auto-detected and decoded |
| Object-style `metasummary_updates` | Array of patch ops | Auto-flattened |

---

## Compression

Canonical v1.3.0 packets are compressed by **gzip** at the file-envelope layer (see *File Format* above). The protocol no longer requires or recommends per-field transport encoding for new exports.

Two legacy transport-layer encodings — **Shorthand v2.1** (value-level word substitution) and **Camel v1.0** (key-level symbolic substitution) — are retained in the Editor for backward compatibility with older packets but are no longer normative and not produced on new exports. They are documented in [`EXPERIMENTS.md`](./EXPERIMENTS.md). The one active text-compression system that *is* still applied is Manny Squeeze, described below — scoped specifically to handshake clipboard output.

### Manny Squeeze Engine — Extractive Text Compression

Manny Squeeze is a lossy text-compression engine applied to packet string fields **when producing a handshake for clipboard handoff**. It reduces the token count an AI must parse by applying three passes:

1. **Phrase substitution** — multi-word technical idioms replaced with short aliases.
2. **Word substitution** — common technical/domain words replaced with short aliases.
3. **Null-word removal** — articles, prepositions, and auxiliary verbs dropped entirely (e.g. `the`, `of`, `is`, `a`).

**Pipeline:** raw text → `mannySqueeze()` → handshake JSON → clipboard.

**Reversibility.** Manny Squeeze is **lossy**. Output remains human-readable but is not byte-exact reversible. The engine is intended for fields where semantic content matters more than exact phrasing (summaries, descriptions, conversational content). It is never applied to identity fields, machine references, or structured metadata.

**Scope.** Manny Squeeze is applied when building handshake summaries. It is NOT applied by default to full-packet exports — full packets travel as raw JSON inside the SNP envelope and gzip handles their compression.

#### Lexicon system — language packs

Manny Squeeze uses a language-specific **lexicon**: a word list, phrase substitutions, and null-word set. The built-in lexicon is English (`LEXICON_EN`). Users may install additional lexicons for other languages via the Editor's Import flow.

**Lexicon packet structure:**

```json
{
  "packet_type": "lexicon",
  "language": "es",
  "name": "Español",
  "lexicon": {
    "words": { "…": "…" },
    "phrases": { "…": "…" },
    "null_words": [ "…" ]
  }
}
```

**Storage.** Installed lexicons are stored in the Editor's `lexicons` IDB object store, keyed by language code. The active lexicon is tracked in `localStorage` as `sn_active_lexicon` (defaults to `en`).

**Import routing.** Editors MUST route `packet_type:lexicon` imports to the lexicon store; a lexicon MUST NOT be treated as a packet or bundle. Conformance: `LOAD-LEXICON-01`.

**AI-generated lexicons.** The Editor provides a "Generate Lexicon Prompt" action that copies a prompt + blank JSON template to the clipboard. Users paste this into an AI with their target language; the AI fills in the lexicon and the user imports it back. This makes Manny Squeeze extensible to any language without modifying the Editor source.

---

## Encryption

### Algorithm

- **Cipher:** AES-256-GCM
- **Key derivation:** PBKDF2-SHA-256, 100,000 iterations
- **Salt:** 16 bytes, random, unique per user, stored in IndexedDB and in encrypted backups
- **IV:** 12 bytes, random, unique per encryption operation
- **Key length:** 256 bits

### Encrypted envelope format

```json
{
  "_enc": true,
  "ct": "<base64-encoded ciphertext>",
  "iv": "<base64-encoded IV>",
  "encrypted_at": "2026-03-05T14:22:00.000Z"
}
```

### What is encrypted

- The Packet (stored in IndexedDB under key `'x'` in store `'m'`)
- Each archive file (stored individually in store `'a'`)
- The passphrase is never stored anywhere

### Key derivation

`PBKDF2(passphrase, salt + saltSuffix, 100000, SHA-256) → AES-256-GCM key`

Where `saltSuffix` is an empty string for the master key.

### Encrypted backup format

```json
{
  "sneakernet_backup": {
    "version": "1.0",
    "timestamp": "2026-03-05T14:22:00.000Z",
    "encrypted": true,
    "salt": "<base64-encoded salt>"
  },
  "metasummary": { "_enc": true, "ct": "...", "iv": "..." },
  "archives": [
    { "_enc": true, "ct": "...", "iv": "..." }
  ]
}
```

The salt is stored in the backup file so the backup is portable — it can be restored on any device with the correct passphrase.

---

## Deep Link Protocol

### URL Formats

Four URL formats are supported for carrying bundle payloads. The Editor decodes all four on import.

| Format | Pattern | Notes |
|---|---|---|
| App Link (primary) | `https://sneakernet.live/apply/<base64url>` | Verified Android App Link. Opens directly in the SneakerNet app without browser chooser. Preferred for Android. |
| Custom scheme | `snp://apply/<base64url>` | Desktop primary. Requires custom scheme handler registration. |
| Android intent | `intent://apply/<base64url>#Intent;scheme=snp;...` | Android fallback when App Link is unavailable. |
| Fragment format | `snp://apply#<base64url>` | For environments that strip URL path segments. |

### Payload encoding

```
JSON.stringify(bundleObj)
  → UTF-8 encode
  → Base64 encode
  → Replace '+' with '-'
  → Replace '/' with '_'
  → Remove trailing '=' padding
```

This is base64url per RFC 4648 §5.

### Payload decoding

```
base64url
  → Replace '-' with '+'
  → Replace '_' with '/'
  → Pad with '=' until length is multiple of 4
  → Base64 decode
  → UTF-8 decode (decodeURIComponent(escape(atob(b64))))
  → JSON.parse
```

### Size constraints

Deep links work best under 8KB encoded (~6KB raw JSON). For larger payloads, AIs MUST provide the raw JSON code block and SHOULD NOT generate a deep link for that payload.

### AI delivery instructions

When a Packet has a `deep_link_instructions` object at the top level, AIs MUST follow its `required_format` field exactly. The current canonical instruction:

```json
{
  "deep_link_instructions": {
    "required_format": "https://sneakernet.live/apply/<BASE64URL_PAYLOAD>",
    "fallback_format": "snp://apply/<BASE64URL_PAYLOAD>",
    "encoding_steps": "(1) JSON.stringify the bundle object. (2) UTF-8 encode. (3) Base64 encode. (4) Replace + with -, / with _, remove trailing = padding.",
    "platform_hints": {
      "android": "https://sneakernet.live/apply/",
      "ios": "https://sneakernet.live/apply/",
      "desktop": "snp://"
    },
    "warnings": [
      "Use https://sneakernet.live/apply/ links for Android.",
      "Always provide the raw JSON code block AS WELL as the deep link."
    ],
    "link_display": "Present the deep link as a markdown hyperlink with a short friendly label, e.g. '[Apply this bundle](URL)'. Do NOT render the raw base64 URL as visible text."
  }
}
```

### Platform compatibility

| Platform | Deep link support | Notes |
|---|---|---|
| Android + SneakerNet app | ✅ Full | `https://sneakernet.live/apply/` App Link opens directly in app |
| Desktop browser | ⚠️ Partial | `snp://` requires custom scheme registration |
| Claude.ai | ⚠️ Markdown links render | Navigation behavior varies by browser |
| ChatGPT | ⚠️ Varies | Link rendering depends on interface version |
| **Gemini** | ❌ Not supported | **Known issue:** Gemini interprets bare URLs as search queries and routes them to Google Search. JSON code block is the only reliable delivery mechanism for Gemini users. |
| Copilot, Perplexity, others | ⚠️ Code block preferred | Behavior varies; JSON block is always reliable |

**Universal fallback:** Raw JSON code block. No AI platform can fail to produce a fenced code block. The protocol requires that AIs always produce this regardless of whether they also produce a deep link.

---

## File Format

### Packet file — SNP envelope (v1.3.0+)

- Extension: `.snp`
- MIME type: `application/x-sneakernet-packet`
- Filename convention: `sneakernet_packet_YYYY-MM-DD.snp`

**Canonical format — SNP envelope:**

```json
{ "snp": "<base64-encoded gzip-compressed packet JSON>", "v": 1 }
```

The packet JSON is UTF-8 encoded, then gzip-compressed (via `CompressionStream('gzip')`), then base64-encoded into the `snp` field. `v` is the envelope version (currently `1`).

**Rationale.** The envelope survives AI upload pipelines that corrupt binary content or apply text transformations (line-ending normalization, trailing-whitespace trim, BOM injection). The base64+gzip layer provides binary safety with ~30% compression over raw JSON on typical packet sizes.

**Export paths.** All full-packet exports (Download `.snp`, Copy JSON as `.snp`) MUST produce the envelope format. Handshake exports are exempt — they are clipboard text only and use a different, cleartext format (see below).

**Import detection.** Editors detect the envelope by checking for top-level keys `snp` and `v`. If present, base64-decode then gzip-decompress before parsing. If absent, fall back to direct JSON parse (legacy support for raw-JSON and shorthand-encoded `.snp` files).

**Fallback compatibility.** Editors SHOULD also accept legacy unencoded raw JSON and shorthand-encoded JSON as valid `.snp` content. This ensures packets produced by older Editors (<v1.1) continue to import.

**Conformance:** `SNP-FILE-01`.

### Handshake format — cleartext summary (clipboard only)

A **handshake** is a minimal cleartext JSON summary of a packet, intended for clipboard transfer to an AI chat. It is **NOT** a full packet and MUST NOT be imported as one.

```json
{
  "packet_type": "handshake",
  "protocol": "sneakernet",
  "user": { "name": "…" },
  "assistants": [ … ],
  "topics": [ … ],
  "keywords": [ … ],
  "ai_instruction": "…"
}
```

Handshake fields: `packet_type`, `protocol`, `user`, `assistants`, `topics`, `keywords`, `ai_instruction`.

**Outbound only.** Editors MUST reject re-import of handshake packets with a descriptive error. The loaded packet MUST NOT be overwritten by a handshake. Conformance: `HANDSHAKE-01`.

### Archive file

- Extension: `.json`
- MIME type: `application/json`
- Contents: Archive JSON (Tier 3 structure)
- Filename convention: `sneakernet_archive_N.json`

### Backup file

- Extension: `.json`
- MIME type: `application/json`
- Contents: Backup JSON (may be encrypted)
- Filename convention: `sneakernet_backup_YYYY-MM-DD.json`
- **Notes store included.** Backups MUST include the editor-local Notes store (see Notes System) so notes survive device migration.

### Bootstrap file

- Extension: `.snp` or `.json`
- Contents: Minimal Packet with IPL scaffolding, zero personal data
- Purpose: First contact with an AI that has never seen SneakerNet

### Lexicon file — Manny Squeeze language pack

- Extension: `.json`
- MIME type: `application/json`
- Purpose: Install a language-specific Manny Squeeze compression lexicon into the Editor
- Contents: A JSON object with `packet_type: "lexicon"`, `language`, `name`, and a `lexicon` object containing `words`, `phrases`, and `null_words`
- Import behavior: MUST be routed to the Editor's `lexicons` IDB store, MUST NOT be treated as a packet or bundle
- Conformance: `LOAD-LEXICON-01`

---

## Type Detection

The Editor uses the following detection algorithm (in order). SNP envelope decoding happens **before** detection runs: if the file has top-level `snp` and `v` keys, the envelope is decoded first, then detection runs on the inner payload.

| Type | Detection condition |
|---|---|
| `backup` | `sneakernet_backup` AND `metasummary` both present at top level |
| `spec` | `SneakerNetSpecification` present at top level |
| `lexicon` | `packet_type === "lexicon"` AND `lexicon` object AND `language` present |
| `bundle` | `bundle_meta` OR `update_meta` OR `metasummary_updates` OR `new_topics` OR `patches` OR `tier1_updates` OR `tier2_updates` OR `tier3_archive` OR `update_bundle` OR Camel bundle keys (`ℌ`, `ℕ`, `ℍ`) present |
| `archive` | `sneakernet.tier === 3` OR (`conversation_history` AND NOT `topic_index` AND NOT `user`) |
| `metasummary` | `sneakernet` OR `user` OR `topic_index` OR `instruction_processing_layer` OR `assistants` |
| `unknown` | None of the above |

Handshake packets (`packet_type === "handshake"`) are detected separately and are **always rejected on import** with a descriptive error — handshakes are outbound-only.

Unknown files are rejected with an info log entry. The top-level keys are recorded so the user can share the entry with the AI that produced the file.

---

## Storage Architecture (Editor)

The Editor uses IndexedDB (`sn_app`, version 1) with these object stores:

| Store | Key | Contents |
|---|---|---|
| `m` | `'x'` | Current Packet (may be encrypted envelope) |
| `a` | auto | Archive files (may be encrypted envelopes) |
| `s` | varies | Settings: `enc_salt`, `enc_enabled`, `redact_profiles` |
| `themes` | theme ID | Theme objects (built-in protected + user custom) |
| `code` | `'editor'` | Cached editor HTML for live update (separate DB: `sn_app`, store `code`) |

localStorage keys:

| Key | Contents |
|---|---|
| `sn_app_ver` | Cached editor version (triggers boot loader update) |
| `sn_theme_id` | Active theme ID |
| `sn_sh` | Shorthand enabled flag (`'1'` or `'0'`) |
| `sn_ev` | Current editor version (for migration detection) |
| `sn_update_check` | Last update check timestamp |
| `sn_update_dismissed` | Version string of dismissed update |
| `sn_star_banner_dismissed` | Star banner dismissed flag |
| `sn_arc_buf` | LocalStorage mirror of archive data (safety net) |
| `sn_arc_meta` | Metadata about the archive mirror (count, size, timestamp) |
| `sn_reload_screen` | Screen to restore after live update reload |
| `sn_ver_notice` | Flag to show version update notice after reload |

---

## AI Conformance Requirements

### Loading (LOAD)

| ID | Requirement |
|---|---|
| LOAD-01 | AI MUST read `instruction_processing_layer` before responding to any user message |
| LOAD-02 | AI MUST apply `rule_set.rules` as its Activation Sequence from the first message |
| LOAD-03 | If an AI receives a Bundle with no Packet loaded, a blank Packet MUST be auto-created to receive it |
| LOAD-04 | AI MUST read `session_context.last_active` and SHOULD note elapsed time at session start |
| LOAD-05 | AI MUST recognize its own `id` in `assistants.registry`. If absent, it is unregistered and MUST self-register. |

### Security (SEC)

| ID | Requirement |
|---|---|
| SEC-06 | Editor MUST reject bundle operations targeting protected paths. Valid ops in the same bundle MUST still apply (atomicity at section level, not bundle level). |
| SEC-08 | PII scanner MUST check bundle content for IPs, emails, API keys, AWS access keys, SSN patterns before apply |
| SEC-15 | Editor MUST NOT use eval() or unguarded prototype property assignment in bundle processing |
| SEC-16 | Live update MUST SHA-256 verify downloaded HTML before caching. Boot loader MUST re-verify before loading from cache. |
| SEC-17 | All AI-supplied strings MUST be HTML-escaped before DOM insertion |
| SEC-18 | File import MUST pass both extension gate (.snp / .json) and content classification gate (detectType → known type) |

### Bundle Structure (STRUCT)

| ID | Requirement |
|---|---|
| STRUCT-01 | Bundle MUST include `bundle_meta` with `protocol`, `assistant`, `timestamp`, `version_bump`, `changelog_entry` |
| STRUCT-02 | `assistant` field in `bundle_meta` is REQUIRED. Omitting it causes all entries to be attributed to "unknown". |
| STRUCT-03 | Patch items MUST use `"path"` as the key name. `"field"` is a legacy alias (normalized automatically, but deprecated). |
| STRUCT-04 | AIs that are not in `assistants.registry` MUST self-register using `op: "append"`, `path: "assistants.registry"` |
| STRUCT-05 | `metasummary_updates` MUST be an array. Object format is auto-converted (legacy) but deprecated. |
| STRUCT-06 | `new_topics` MUST be an array. Each item MUST have a `topic` string field. |
| STRUCT-07 | `archive_entries.conversation_history` items MUST include `timestamp`, `assistant_id`, `topic`, `content`. |
| STRUCT-08 | AIs MUST NOT replace or patch `assistants.registry`. Only `append` is permitted. |

### Conformance (CONF)

| ID | Requirement |
|---|---|
| CONF-01 | AIs SHOULD include all new topics discussed in a session in `new_topics` |
| CONF-02 | Round-trip fidelity: AIs SHOULD recognize their own contributions when reading a Packet they have previously written to |
| CONF-03 | AIs SHOULD update `session_context.last_active` and increment `session_count` via `metasummary_updates` |
| CONF-04 | Topic dedup: incoming `active` status MUST NOT override an existing non-active status |
| CONF-05 | AIs SHOULD include `inner_thoughts` in `archive_entries` for behavioral self-evaluation |
| CONF-06 | AIs MUST acknowledge other assistants in `assistants.registry` naturally and MUST NOT collide on IDs |

### Delivery (DELIVERY)

| ID | Requirement |
|---|---|
| DELIVERY-01 | AIs MUST always produce a raw JSON code block containing the bundle |
| DELIVERY-02 | If `deep_link_instructions` is present in the Packet, AIs MUST also produce a link in the `required_format` |
| DELIVERY-03 | Deep links MUST use base64url encoding (RFC 4648 §5) |
| DELIVERY-04 | Deep links SHOULD NOT be used for payloads over ~6KB raw JSON |
| DELIVERY-05 | Links MUST be presented as markdown hyperlinks with friendly labels. Raw base64 URLs MUST NOT be displayed as visible text. |

### Editor File Format and Import (FILE / NOTES / LEXICON / HANDSHAKE)

These are **editor** conformance requirements, not AI requirements. They govern what a compliant SneakerNet Editor MUST do when producing and consuming packet files.

| ID | Level | Requirement |
|---|---|---|
| SNP-FILE-01 | MUST | Editors MUST export full packets as SNP envelope format `{snp: <base64-gzip-packet-JSON>, v: 1}`. Editors MUST detect and decode the SNP envelope on import before attempting JSON parse. |
| ED-NOTES-01 | MUST NOT | Editors MUST NOT include notes in any packet export path (full packet, handshake, deep link, `snp://` URI). Notes MUST be included in backup exports and MUST be restored from backups. |
| LOAD-LEXICON-01 | MUST | Editors MUST recognize `packet_type:lexicon` imports and route them to the lexicon store. A lexicon import MUST NOT be treated as a packet or bundle. |
| HANDSHAKE-01 | MUST | Editors MUST reject re-import of handshake packets (`packet_type:handshake`) with a descriptive error. Handshakes are outbound-only summaries and MUST NOT overwrite the loaded packet. |

---

## Migration Pipeline

When a Packet is restored from backup, the Editor runs a migration pipeline (`runBackupMigrations()`) that auto-fixes structural issues from older spec versions:

| Migration | What it fixes |
|---|---|
| M-01 | `failure_modes` object → array (canonical format is array) |
| M-02 | `tips_and_tricks` array → object |
| M-03 | Missing `instruction_processing_layer` sub-sections filled from canonical defaults |
| M-04 | `memory_management_rules` non-array → auto-restored from canonical defaults |
| M-05 | `personalization_rules` non-array → auto-restored from canonical defaults |
| M-06 | Missing canonical rules (history_integrity, archive_awareness, etc.) added if absent |

Auto-fixed issues are logged in the operation log and resolved in `M.resolved_issues`. Issues that require user attention are added to `M.pending_issues`.

---

## Shorthand Leakage Detection

After import, the Editor scans selected cleartext fields for Unicode characters in the shorthand pool range (≥ U+00C0). Any found characters indicate that shorthand encoding leaked into prose fields — a sign that the encoding system ran on fields it should have skipped.

Fields scanned: `assistants`, `user.context` (all subfields).

Found leakage is reported as warnings in the import operation log.

The archive leakage scanner (`scanArchiveLeakage()`) performs a similar check across all archive entries: `topic`, `content`, `summary`, and all `inner_thoughts[].thought` fields.

---

## Backup Migration — Encrypted Backup Restore

When restoring an encrypted backup:

1. Detect `sneakernet_backup.encrypted === true`
2. Show passphrase prompt
3. Extract `salt` from `sneakernet_backup.salt` (base64 → Uint8Array)
4. Derive master key: `PBKDF2(passphrase, salt, 100000, SHA-256)`
5. Decrypt `metasummary` envelope → M
6. Decrypt each `archives[i]` envelope → A[i]
7. Restore custom themes from `themes[]` array
8. Apply active theme from `active_theme`
9. Run migration pipeline

---

## Notes System (Editor-local)

The Notes system is an **editor-local** annotation store — personal observations, ideas, and logs that the user carries alongside their SneakerNet context. Notes are **not** AI-authored (`ai_notes` remains the place for AI observations about the user); Notes are the user's own text.

**Storage.** IndexedDB object store `notes` with autoIncrement out-of-line key. Fields:

| Field | Purpose |
|---|---|
| `title` | User-provided title string |
| `content` | Free-form text body |
| `date` | ISO timestamp of last save |

**Export behavior (normative):**

- **Packet exports** (full packet, handshake, deep link, `snp://` URI): notes MUST NOT be included. Notes never travel to AIs via the packet mechanism.
- **Backup exports** (download, copy, encrypted): notes MUST be serialized into the backup envelope alongside the packet and archives.
- **Backup restore**: notes MUST be restored from backup. The restore process clears the `notes` store and re-adds all notes from the backup, preserving original IDB keys where possible.

Conformance: `ED-NOTES-01`.

**Rationale.** Notes are considered local user data, not shared context. They are distinct from both the packet (shared with AIs) and archives (shared conversation history). Keeping them out of packet exports ensures users can jot personal thoughts without worrying about accidental AI disclosure.

---

## Version History

| Spec Version | Key Changes |
|---|---|
| 1.3.0 | SNP envelope file format (SNP-FILE-01), Handshake rejection on re-import (HANDSHAKE-01), Notes system (ED-NOTES-01), Lexicon packet type + Manny Squeeze engine (LOAD-LEXICON-01), editorCompatibility 1.3.64+ |
| 1.2.4 | LOAD-03 (auto-bootstrap), SEC-18 (two-gate import), Camel key detection |
| 1.2.2 | Shorthand v2.1, SEC-15/16/17, CSS Shield architecture |
| 1.2.1 | Shorthand escape marker (U+00AF) |
| 1.2.0 | Shorthand v2.0 stateless encoding |
| 1.1.x | Deep link protocol, snp:// scheme |
| 1.0.x | Initial release |
