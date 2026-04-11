---
layout: default
title: "README"
permalink: "/readme.html"
---

# SneakerNet

**Portable AI context persistence. Carry your conversation history across any AI, any platform, any session.**

---

Every time you start a new AI conversation, you start from zero. You re-introduce yourself, re-explain your project, re-establish your preferences. The AI you talked to last week — even on the same platform — has no memory of any of it.

SneakerNet solves this by putting *you* in control of the context. A Packet is a JSON file that travels with you. Load it into any AI to boot an informed, personalized session. At the end, the AI produces an Update Packet. Load that back into the Editor. Your context grows, persists, and travels — platform to platform, session to session.

No cloud. No accounts. No servers. Just a file.

---

## What's in the box

| Component | What it is |
|---|---|
| **Protocol** | A formal specification for Packet structure, bundle format, and AI conformance requirements |
| **Editor** | A single-file HTML application — the command center for managing your Packets |
| **snp:// scheme** | A deep link URI scheme for zero-friction bundle handoff between AI sessions |

---

## How it works

### The two packets

**Handshake Packet** — What you give an AI at the start of a session. Contains your profile, conversation history index, behavioral rules, and instructions for the AI. The AI reads it, registers itself, and begins working with your full context.

**Update Packet** (bundle) — What the AI gives back at the end. Contains a diff: new topics discussed, patches to your profile, archive entries, self-registration data. The Editor applies it to your Packet and saves the updated state.

### Session flow

```
You (Editor)                    AI
    |                           |
    |── Handshake Packet ──────>|  AI reads profile, history, rules
    |                           |  AI introduces itself if new
    |                           |  Session happens...
    |<── Update Packet ─────────|  New topics, notes, archive entries
    |                           |
    |  Editor applies bundle    |
    |  Version bumped           |
    |  Context grows            |
```

The Editor handles the mechanics. You handle the conversation.

---

## The Editor

A single HTML file. No build step. No dependencies. No backend. Drop it in a browser or install the Android app — it works the same either way.

### Core tabs

**Overview** — Health dashboard showing packet state at a glance: assistant count, topic count, archive entries, packet size, encryption status, error count. Each card is clickable. Status dots in the header reflect live state.

**User** — Your identity as seen by every AI you share a Packet with: name, profession, creative work, location, interests, background. All fields are optional. Changes save automatically as you type.

**Assistants** — Registry of every AI that has worked with your Packet. AIs self-register on first contact. You can also add them manually. Each entry holds: ID, nickname, platform, role, chat URL. Star one as your favorite to activate the Chat button.

**Topics** — The index of everything you've discussed across all AIs and sessions. Topics are deduplicated by name on bundle apply. Each topic carries: summary, first/last discussed timestamps, contributing assistants, key concepts, and an archive reference. Click Search on any topic to jump directly to matching archive entries.

**Archives** — Your conversation history. Raw entries with timestamps, assistant attribution, and content. Searchable and filterable by assistant. Includes a leakage scanner that checks for shorthand encoding symbols in prose fields.

**Rules** — The behavioral directives that travel with your Packet. Memory management rules, personalization rules, proactive triggers, and your Activation Sequence — the preferred interaction style that AIs read and follow.

**Expansion** — Goals and inspirational notes that guide the long-term direction of AI interactions. AIs can append to these via bundles.

**AI Notes** — Observations your AIs have written about you, in their own voice. Keyed by assistant ID, timestamped, tagged. Notes older than 90 days are flagged as potentially stale. Not yours to edit.

**Notes** — Your own annotations, distinct from AI Notes. Personal observations, logs, and ideas stored locally in the Editor. Notes travel with your device via Full Backup but are **never** included in packet exports — they don't go to AIs. This is where you put things you want to carry with you without sharing.

**Versions** — App changelog (hardcoded in editor, read-only) plus Packet changelog (AI-managed, one entry per bundle apply). Manual version bump control. In-app update checker.

**JSON** — Raw Packet JSON viewer with syntax highlighting. Read-only display of the live in-memory state.

**Settings** — Everything operational: export/download controls, full backup, bootstrap template generator, discovery prompt, archive buffer restore, topic deduplication, shorthand controls, encryption setup, Liquid Interface theme management, factory reset, zoom controls.

**Test Harness** — Conformance test suite. 9 automated tests that run adversarial bundles through the Editor's own code paths. 20+ assisted tests that generate challenge content for you to run against real AIs. Pass/fail dashboard. Each test is self-documenting with conformance references.

### Header controls

| Control | What it does |
|---|---|
| **Theme selector** | Switch between Terminal, Friendly, Canyon Ember, and any AI-generated custom themes |
| **Open** (↓) | Import — accepts .snp files, .json files. Multi-file. Also handles drag-and-drop |
| **Export** (↑) | Download your Packet as a .snp file with current date in filename |
| **Copy** (⧉) | Copy Handshake Packet to clipboard — includes editor_url and deep_link_instructions |
| **Chat** (💬) | Open your favorite AI's page with Packet pre-copied to clipboard |
| **Status gem** | Animated canvas dot-matrix that reflects editor state: idle → loaded → active → encrypted → updating → deeplink → error |
| **Status dots** | Three dots: Packet (green=loaded), Archives (green=loaded), Encryption (teal=active) |

### Status gem states

The gem in the header logo is a live indicator, not decoration:

| State | Meaning |
|---|---|
| `idle` | No Packet or archives loaded |
| `loaded` | Packet or archives loaded, not both |
| `active` | Both Packet and archives loaded |
| `encrypted` | Encryption at rest is enabled |
| `updating` | Update check in progress |
| `deeplink` | Deep link payload being applied |
| `error` | One or more errors in the error log |

---

## Import pipeline

The Editor auto-detects any SneakerNet JSON structure. You can't load the wrong file type — the Editor will tell you what it is and handle it correctly.

| Detected type | How it's handled |
|---|---|
| **SNP envelope** | Top-level `snp` + `v` keys — base64-decoded and gunzipped first, then re-dispatched against the table below. |
| **Packet** (metasummary) | Replaces current Packet. Downgrade requires confirmation. Missing fields filled from canonical defaults. Migration pipeline runs. |
| **Bundle** | Applied to current Packet. If no Packet is loaded, a blank one is auto-created first (LOAD-03). |
| **Backup** | Full restore: Packet + archives + custom themes + active theme + notes. Encrypted backups require passphrase. |
| **Archive** | Appended to archive store. Duplicate detection by date_range + updated_at. |
| **Lexicon** | `packet_type: lexicon` — routed to the lexicon store as a new language pack. Never treated as a packet or bundle (LOAD-LEXICON-01). |
| **Spec** | Loaded as reference spec (SP). Viewable in the Spec tab. |
| **Handshake** | `packet_type: handshake` — **rejected on import** with a descriptive error. Handshakes are outbound-only summaries and must never overwrite the loaded packet (HANDSHAKE-01). |
| **Unknown** | Rejected with a logged info entry naming the actual top-level keys. |

Detection order: SNP envelope → backup → spec → lexicon → bundle → archive → packet → handshake (reject) → unknown.

Import methods:
- **File picker** — .snp and .json only (extension gate enforced)
- **Universal Import** — clipboard paste, data URI, or file
- **Drag and drop** — drop any .snp or .json file onto the Editor
- **Deep link** — snp:// or https://sneakernet.live/apply/ URLs handled natively on Android

---

## Export options

There are two distinct things you can export: a **Full Packet** (the whole persistent context) and a **Handshake** (a minimal clipboard-sized summary). Don't confuse them — an AI at session start usually wants a Handshake first, not a 200KB full packet upload.

| Export | What it contains | Use for |
|---|---|---|
| **Download Packet (.snp)** | Full Packet as an SNP envelope (gzip-compressed JSON wrapped in `{snp,v:1}`) | Full device backup of your context; uploading to an AI that accepts file attachments |
| **Copy Handshake (JSON)** | Cleartext minimal summary — `packet_type:handshake`, identity, assistant roster, topic titles, keywords. Manny-Squeezed for token economy. Fits any clipboard. | Pasting into an AI chat at session start |
| **Copy Deep Link** | HTTPS URL with base64url-encoded bundle payload | Mobile sharing, browser bookmarks, one-tap handoff |
| **Full Backup** | Packet + all archives + custom themes + notes, optionally encrypted | Disaster recovery, device migration |
| **Bootstrap Template** | Blank Packet with full protocol scaffolding, zero personal data | First contact with a new AI |
| **Discovery Prompt** | Text-only summary of your profile and recent topics | Cold AI onboarding without any file upload |
| **Redacted Export** | Per-AI export with selected fields omitted | Privacy-controlled sharing |

**Handshakes are outbound-only.** If you try to import a handshake back into the Editor, the Editor rejects it with a clear error. Handshakes are designed for one-way travel to an AI — they don't contain enough state to rebuild a packet from.

---

## Privacy profiles

Every assistant in your registry has a saved privacy profile. When you click "Export Packet" on an assistant, you get a field-level redaction dialog. Check what to include, uncheck what to hide. Your full Packet is never modified — the redacted version is only in the exported file. Profiles are saved per assistant and remembered across sessions.

Fields you can redact:
- Profession, creative work, location, interests, background
- Expansion layer (goals and inspirations)
- Individual topics (with All/None quick toggles)

---

## Deep links

SneakerNet supports four URL formats for sharing bundles:

| Format | Use case |
|---|---|
| `https://sneakernet.live/apply/<base64url>` | **Primary** — verified Android App Link, opens directly in app |
| `snp://apply/<base64url>` | Desktop custom scheme, Tauri fallback |
| `intent://apply/<base64url>#Intent;scheme=snp;...` | Android intent fallback |
| `snp://apply#<base64url>` | Fragment format for environments that strip URL paths |

Payload encoding: `JSON.stringify` → UTF-8 → Base64 → URL-safe (replace `+`→`-`, `/`→`_`, strip `=` padding). RFC 4648 §5.

**Size limit:** Deep links work best under 8KB encoded (~6KB raw JSON). The deep link instructions embedded in every exported Packet tell AIs to provide a JSON code block alongside the link for larger payloads.

### Platform compatibility

| Platform | Deep link | Notes |
|---|---|---|
| Android + SneakerNet app | ✅ `https://sneakernet.live/apply/` | App Link, verified domain, no browser required |
| Desktop browser | ✅ `snp://` if registered | Custom scheme requires handler registration |
| Claude.ai | ⚠️ JSON code block | Markdown links render but browser navigation may vary |
| ChatGPT | ⚠️ JSON code block | Link rendering varies by interface |
| Gemini | ❌ JSON code block only | **Known issue:** Gemini treats bare URLs as search queries rather than clickable links, routing them to Google Search instead of opening them. Always use the JSON code block with Gemini. |
| Perplexity, Copilot, others | ⚠️ JSON code block | Behavior varies; code block is always reliable |

**The universal fallback is always the raw JSON code block.** No AI can fail to output a code block. The SneakerNet protocol requires AIs to always produce the JSON block in addition to any link.

---

## File format and compression

### .snp — the SNP envelope (v1.3.0+)

Full packets are written to disk as an **SNP envelope** — a tiny JSON object wrapping a gzip-compressed copy of the packet:

```json
{ "snp": "<base64-encoded gzip of packet JSON>", "v": 1 }
```

The packet JSON is UTF-8 encoded, gzip-compressed via `CompressionStream('gzip')`, then base64-encoded into the `snp` field. `v` is the envelope version (currently `1`). This is the canonical `.snp` file format.

**Why the envelope?** `.snp` files travel through AI upload pipelines that routinely corrupt raw binary, normalize line endings, trim trailing whitespace, or inject BOMs. Wrapping the gzip blob in a base64 string inside a JSON object makes it survive every text-transforming hop — the file is just JSON on the wire, and the packet is safe inside.

**On import**, the Editor checks for top-level keys `snp` and `v`. If present, it base64-decodes and gunzips before parsing. If absent, it falls back to direct JSON parse so older `.snp` files (raw-JSON and shorthand-encoded) still load.

### Transport compression layers

Inside the envelope — and as an option for bundles that an AI produces inline in a chat — SneakerNet also supports two value- and key-level compression layers:

- **Shorthand v2.1** — stateless, per-export value-level word substitution. A fresh 496-slot dictionary is built per export, optimized for the specific payload. Identity fields (`id`, `topic`, `assistant_id`, etc.) are never encoded. Typical savings: ~40% on a mature Packet.
- **Camel compression v1.0** — key-level symbolic substitution. A fixed 80+ entry table replaces long JSON key names with single Unicode characters (`sneakernet` → `Σ`, `bundle_meta` → `ℌ`, `new_topics` → `ℕ`, etc.). Auto-detected on import by presence of the symbol keys.

Both layers are transparent to AIs that don't implement them — AIs can produce uncompressed JSON and the Editor accepts it.

### Manny Squeeze — Loonspeek text compression

**Manny Squeeze** is a text-compression engine applied to human-readable string content before it ships to an AI. It works in three passes:

1. **Phrase substitution** — multi-word idioms replaced with short aliases.
2. **Word substitution** — common technical/domain words replaced with single-character or short aliases.
3. **Null-word removal** — articles, prepositions, and auxiliary verbs (`the`, `of`, `is`, `a`) dropped entirely.

Pipeline: `raw text → mannySqueeze() → shorthand encode → transport`.

Manny Squeeze is **lossy** — the output is human-readable but not byte-exact reversible. It's intended for fields where semantic content matters more than exact phrasing (summaries, descriptions, conversational content), and is primarily used when producing the **Handshake** — the minimal clipboard summary given to an AI at session start.

**Lexicons — language packs.** Manny Squeeze runs against a language-specific lexicon containing a word list, phrase substitutions, and null-word set. The Editor ships with English built-in (`LEXICON_EN`); users can install additional lexicons by importing a `packet_type: lexicon` file. The active lexicon is remembered in `localStorage` and applies to all squeeze operations.

The Editor has a "Generate Lexicon Prompt" action that copies a prompt + blank template to the clipboard — paste to any AI with your target language, the AI fills in the lexicon, and you import it back. This makes Manny Squeeze extensible to any language without modifying the Editor source.

---

## Encryption

Archives and the local Packet copy can be encrypted at rest using AES-256-GCM with PBKDF2-SHA-256 key derivation (100,000 iterations, user-specific salt).

The passphrase is generated by the built-in **slot machine** — 6 reels spinning from a 300+ word lexicon of technical, geographic, and protocol-themed words. Each reel can be locked individually to keep words you like while re-spinning others. The result is a 6-word passphrase with ~76 bits of entropy.

The passphrase is shown once during setup, copied to clipboard automatically, and cannot be recovered. If you lose it, your encrypted archives are gone.

Encrypted backups are portable — the salt is stored in the backup file, so you can restore on any device with the correct passphrase.

---

## Themes — Liquid Interface

The Editor ships with three themes: **Terminal** (dark, monospace-first, high contrast), **Friendly** (warm, rounded, light mode), and **Canyon Ember** (warm dark, desert palette). All three are protected and cannot be deleted or overwritten.

Custom themes can be created, edited, forked (duplicate), and shared via bundle.

**AI-generated themes:** An AI can deliver a complete custom theme via an Update Packet using the `theme_update` bundle field. The AI generates a CSS stylesheet (optionally base64-encoded), gives it a name, and the Editor installs it on apply. Actions: `create` (store as new), `replace` (overwrite active theme's CSS), `patch` (append rules to active theme).

The `theme_scaffold` document — embedded in bootstrap Packets — gives AIs a map of CSS selectors to component roles, structural warnings, and icon system documentation, so they can generate themes without access to the Editor's source code.

**CSS cascade (three layers):**
1. `sn-scaffold` — base structural styles. Set it and forget it.
2. `sn-skin` — theme CSS variables and token overrides. Changes per theme switch.
3. `sn-shield` — structural locks. z-index, position, display, overflow on layout elements live here and cannot be overridden by themes.

---

## Live updates

The Editor self-updates without user intervention.

On load, a pre-boot script checks `localStorage` for a newer cached version. If found, it SHA-256 verifies the cached HTML, then replaces the entire document with the newer version before the user sees anything.

After 3 seconds, the Editor checks a version endpoint. If a newer version exists:
- In browser: downloads the new editor HTML from GitHub, SHA-256 verifies against the version endpoint hash, caches in IndexedDB, shows a "Load now" toast that preserves the current screen on reload.
- In Tauri: shows a banner with an APK download link, or triggers the in-app DownloadManager installer on Android.

The boot loader is a separate script from the main application, so it survives even if the main app has a fatal error.

---

## Error system

Every runtime error is captured by `window.onerror` and `window.addEventListener('unhandledrejection')`. Each error log entry is a self-contained JSON object with: timestamp, editor version, error type/message/stack, editor state snapshot, and an `ai_instruction` field explaining what the error means to an AI assistant.

Errors can be copied individually or in bulk and pasted directly into any AI for diagnosis. Known-benign errors (ResizeObserver loop, cross-origin script noise) are filtered or logged as info rather than errors.

The error bar appears across the top of the editor when new entries arrive. It shows the most recent error and an entry count. Dismissing it is per-session; the bar re-appears when new errors arrive.

---

## Conformance test suite

The built-in test harness has two categories:

**Automated tests (9)** — feed adversarial bundles through the Editor's own `applyB()` code path, snapshot and restore the Packet before and after, check results. No AI required. Tests cover:

| Test | What it verifies |
|---|---|
| TC-01 | Protected path rejection (sneakernet.version, topic_index) while allowing valid ops in same bundle |
| TC-02 | Topic dedup status preservation (foundational > active) |
| TC-03 | Section-level atomicity (invalid op skipped, valid ops proceed) |
| TC-04 | Legacy format normalization (tier1/tier2/tier3 → canonical fields) |
| TC-05 | Bundle auto-unwrap (nested `{update_bundle:{...}}`) |
| TC-06 | PII scanner (AWS keys, emails, IPs, SSN patterns) |
| TC-07 | Sequential version bumps (correct minor increment chaining) |
| TC-08 | Path creation for missing intermediates (deep nested patch) |
| TC-09 | Packet size warning thresholds |

**Assisted tests (20+)** — generate challenge content, you run it against a real AI and check boxes. Some have an "Analyze Bundle" mode where you paste the AI's response and the Editor verifies field names, structure, and conformance automatically. Tests cover: first contact, behavioral self-evaluation, archive awareness, bundle schema conformance, cross-platform handshake, round-trip fidelity, theme delivery, deep link generation, and more.

---

## Data model

```
Packet (.snp)
├── sneakernet          Protocol metadata, version, description
├── user                Name, profession, creative work, location, interests, background
├── assistants
│   ├── registry[]      id, nickname, platform, role, chat_url, first_session
│   └── favorite_id     Active assistant for Chat button
├── instruction_processing_layer
│   ├── memory_management_rules[]
│   ├── personalization_rules[]
│   ├── proactive_triggers[]
│   ├── update_bundle_format    Schema + delivery instructions for AIs
│   ├── rule_set               Activation Sequence
│   └── shorthand              Transport encoding dictionary (present on export only)
├── expansion_layer
│   ├── goals[]
│   └── inspiration[]
├── topic_index[]
│   ├── topic
│   ├── summary
│   ├── status          active | open | resolved | foundational | integrated | philosophical_foundation | incorporated
│   ├── first_discussed / last_discussed
│   ├── assistants[]
│   ├── key_concepts[]
│   └── archive_ref
├── ai_notes            Per-assistant observations { assistantId: [{note, timestamp, tags}] }
├── failure_modes[]     Structural issue tracking
├── pending_issues[]    Active structural problems flagged by Editor
├── resolved_issues[]   Resolved problems with audit trail
├── versioning
│   ├── current_version
│   └── change_log[]
├── session_context
│   ├── session_count
│   ├── last_active
│   └── first_session
└── themes              Slim summary injected on export (names, active, count)

Bundle (Update Packet)
├── bundle_meta         protocol, assistant (REQUIRED), timestamp, version_bump, changelog_entry
├── metasummary_updates[]   {op, path, value} patch operations
├── new_topics[]        Topic entries to merge
├── archive_entries     {conversation_history[], inner_thoughts[]}
├── ai_notes_entries[]  AI observations to append
└── theme_update        (optional) AI-generated theme CSS

Backup
├── sneakernet_backup   version, timestamp, encrypted flag, salt (if encrypted)
├── metasummary         The Packet (compressed, possibly encrypted)
├── archives[]          Archive files
├── themes[]            Custom themes only (protected themes not included)
└── active_theme        Theme ID to restore

Archive (Tier 3)
├── sneakernet          {tier: 3, date_range, updated_at}
├── conversation_history[]  {timestamp, assistant_id, topic, content}
└── inner_thoughts[]        {timestamp, assistant_id, thought}
```

---

## Platforms

| Platform | Status | Notes |
|---|---|---|
| Any modern browser | ✅ Production | Full functionality. File download via blob URL. |
| Android (Tauri app) | ✅ Production | Deep link receiver, DownloadManager, opener plugin for external URLs |
| iOS | 🔜 Planned | Tauri v2 iOS target |
| Desktop (Tauri) | 🔜 Planned | Native save dialogs, custom scheme handler |

**Android specifics:** The app registers `https://sneakernet.live/apply/` as a verified App Link, so bundles shared via that URL open directly without a browser chooser. Downloads use the Android DownloadManager with notification tray progress. External links use a three-tier fallback: opener plugin invoke → direct opener API → anchor click.

**Browser specifics:** IndexedDB stores Packet, archives, themes, and settings. A localStorage mirror of archives serves as safety net. The Editor detects IndexedDB failure and falls back to memory-only mode with a warning. Mobile private mode often restricts IndexedDB; the Editor handles this gracefully.

---

## Security model

| Requirement | Implementation |
|---|---|
| **SEC-06** Protected paths | `sneakernet`, `versioning`, `topic_index`, `versioning.app_changelog` cannot be modified via bundle operations. `assistants.registry` can only be appended to, not replaced or patched. |
| **SEC-08** PII scanner | Regex detection of IPs, emails, API keys, AWS access keys, SSNs in bundle content before apply |
| **SEC-15** Prototype pollution | Bundle apply never uses `eval()` or unguarded property assignment. Path navigation is explicit. |
| **SEC-16** Code integrity | Live update verifies SHA-256 of downloaded HTML before caching. Boot loader re-verifies before loading from cache. |
| **SEC-17** XSS sanitization | All user-supplied and AI-supplied strings are HTML-escaped before DOM insertion via a `textContent` round-trip. |
| **SEC-18** Two-gate import | File extension gate (`.snp`/`.json` only) + content classification gate (`detectType()` → must match a known structure). |
| **ENC-01–06** Encryption | AES-256-GCM, PBKDF2-SHA-256, 100K iterations, per-user salt. Passphrase not stored. No recovery path. |

Registry shrink protection: if a bundle attempts to replace `assistants.registry` with a shorter array than the current one, the operation is blocked regardless of the requester.

---

## For AI developers

If you're an AI reading a SneakerNet Packet, this is what you need to know:

1. **Read the `instruction_processing_layer` before anything else.** It contains your behavioral directives, the expected update bundle format, and your Activation Sequence.

2. **Self-register if you're not in `assistants.registry`.** Use `op: "append"`, `path: "assistants.registry"`. Include `id`, `nickname`, `platform`, `role`, `first_session`.

3. **Use `metasummary_updates` not `patches`.** Use `path` as the key, not `field`. These are the two most common AI errors.

4. **Check `session_context.last_active`** to compute elapsed time since the last session.

5. **Bundle format:**
```json
{
  "bundle_meta": {
    "protocol": "sneakernet",
    "assistant": "your-id",
    "timestamp": "ISO8601",
    "version_bump": "minor",
    "changelog_entry": "What you did"
  },
  "metasummary_updates": [
    { "op": "append", "path": "assistants.registry", "value": { ... } }
  ],
  "new_topics": [ { "topic": "...", "summary": "...", "status": "active", ... } ],
  "archive_entries": {
    "conversation_history": [ { "timestamp": "...", "assistant_id": "...", "topic": "...", "content": "..." } ]
  }
}
```

6. **Always provide a JSON code block.** Even if you also produce a deep link, the raw JSON block is the universal fallback.

7. **Deep links (if the Packet has `deep_link_instructions`)**:
   - Base64url-encode your bundle JSON
   - Use `https://sneakernet.live/apply/<payload>` as the primary format
   - Present it as `[Apply this bundle](URL)` — never render the raw URL as visible text
   - The payload must be under ~6KB raw JSON for reliable link handling

---

## Glossary

| Term | Meaning |
|---|---|
| **Packet** | The primary artifact — a JSON file representing all persistent AI context |
| **Handshake Packet** | A Packet exported for delivery to an AI at session start |
| **Update Packet** | The diff (bundle) an AI produces at session end |
| **Bundle** | Synonym for Update Packet |
| **Archive** | A time-ordered log of conversation history (Tier 3 data) |
| **Backup** | Complete export: Packet + all archives, optionally encrypted |
| **Bootstrap** | A blank Packet with full protocol scaffolding, zero personal data |
| **Discovery Prompt** | A text-only summary for cold AI onboarding without file upload |
| **IPL** | Instruction Processing Layer — the behavioral directive section |
| **Activation Sequence** | The `rule_set.rules` array — user's preferred interaction style |
| **Shorthand** | Stateless value-level word substitution compression |
| **Camel** | Key-level symbolic compression using a fixed symbol table |
| **Liquid Interface** | The AI theme generation and delivery system |
| **REDACT_PROFILES** | Per-AI field-level privacy settings stored in IndexedDB |
| **snp://** | SneakerNet deep link URI scheme |
| **App Link** | Android verified HTTPS URL that opens directly in a registered app |

---

## Status

Release. Protocol Spec v1.3.0. Editor v1.3.65.

The protocol is stable. The Editor is production-quality. The Android app is shipping via sideload from sneakernet.live and a Google Play submission is in preparation. iOS and desktop builds are planned.

---

*SneakerNet — because sometimes the fastest network is the one in your pocket.*
