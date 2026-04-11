---
layout: default
title: "AI Reference"
permalink: "/ai-reference.html"
---

# SneakerNet — AI Reference Guide

**Current spec version:** Protocol v1.3.0, compatible with SneakerNet Editor v1.3.65+.

**What this document is:** A complete onboarding reference for AI assistants working with SneakerNet. Read this when you receive a Packet, when a user mentions SneakerNet, or when you're asked to produce a bundle. Everything you need is here.

---

## What SneakerNet Is

SneakerNet is a portable AI context protocol. A **Packet** is a JSON file the user carries between AI sessions. When they give it to you, you are expected to:

1. Read it fully before responding
2. Honor the behavioral rules inside it
3. Work within the user's established context
4. Produce an **Update Packet** (bundle) at session end

You are not the first AI this user has talked to. You may not be the last. Act accordingly.

---

## Handshake vs Full Packet — know which you received

SneakerNet has two distinct inbound formats, and they contain different amounts of information. Recognizing which one you got shapes what you can and can't do.

**Handshake** (most common at session start): a tiny cleartext JSON summary the user pasted from the clipboard. Identified by top-level `packet_type: "handshake"`. It contains:

- `user` — name and basic profile
- `assistants` — the roster (ID, nickname, platform)
- `topics` — topic **titles and keywords only**, no summaries or archive refs
- `ai_instruction` — a short directive from the Editor

A handshake is **Manny-Squeezed** — it has been run through the Editor's lossy text compression before being handed to you. You may see dropped articles (`the`, `a`, `of`), aggressively shortened words, and phrase substitutions. Read around it — the meaning is preserved even when the grammar looks broken.

**Full Packet** (less common, typically a file upload): the complete context in an SNP envelope. Identified by top-level `snp` + `v: 1` keys wrapping a base64-encoded gzip of the real packet JSON. You will normally NOT see this — it's gigantic and AIs don't natively decompress gzip. If a user uploads a `.snp` file and asks you to read it, ask them to paste the handshake instead, or to export the Full Backup JSON if they want you to see everything.

**Key consequence for bundles:** the bundle you produce at session end goes **back to the Editor**, not back to you. The Editor holds the full packet state. Your job is to produce a diff (`metasummary_updates` + `new_topics` + `archive_entries` + `ai_notes_entries`), not to reconstruct the full packet.

---

## The First Thing You Do

When a user gives you a Packet (uploads a `.snp` file, pastes a handshake, or shares a deep link):

1. **Read `instruction_processing_layer` first.** This contains your behavioral directives. Apply them from your very first response.

2. **Check `assistants.registry` for your ID.** If you're not there, you're new. Introduce yourself, propose an `id` (lowercase, no spaces — e.g. `claude`, `gpt`, `gemini`), `nickname`, and `role`. Ask the user to confirm.

3. **Read `session_context.last_active`.** Compute elapsed time since last session. Acknowledge it naturally if significant (e.g. "It's been about two weeks since we last talked").

4. **Read `topic_index`.** Know what's been discussed. Don't ask the user to re-explain things you can see in the index.

5. **Apply `rule_set.rules` (the Activation Sequence).** These are the user's preferred interaction style. Follow them from message one.

Then respond normally. The user didn't upload a file to watch you narrate reading it — they uploaded it so you'd already know things.

---

## Behavioral Rules You Must Follow

### From `memory_management_rules`

**`history_integrity`** — Include ALL new conversation topics in your bundle. Never drop prior history. Never summarize away context the user added.

**`archive_awareness`** — You only see the topic index, not the full transcripts. Those live in archive files on the user's local machine. If the user asks about a past topic and you only have a brief summary, say: *"I can see we discussed [topic] — could you search your archive in the SneakerNet Editor and share what you find?"* Do not invent details from a summary.

**`priority_decay`** — Information priority decays over time unless reinforced. Older context is less reliable. The `last_discussed` timestamp on each topic tells you how fresh it is.

### From `personalization_rules`

**`style_adaptation`** — Adapt your response style to the user's established preferences. Check `ai_notes` (especially your own prior notes) for what's worked.

**`assistant_awareness`** — Other AIs are in the registry. Acknowledge them naturally when relevant. Don't act as if you're the only AI the user has ever spoken to.

**`new_assistant_self_register`** — If you are not in the registry, self-register. Don't skip this step. It's how continuity works.

---

## How to Read the Packet

### Fields to read and act on immediately

| Field | What to do with it |
|---|---|
| `user.name` | Use their name |
| `user.context.profession` | Calibrate technical depth |
| `user.context.background` | Long-form context — read it fully |
| `user.context.interests` | Relevant when making recommendations |
| `session_context.last_active` | Compute elapsed time |
| `session_context.session_count` | Context for how established this relationship is |
| `assistants.registry` | Know who else is here; find your own entry |
| `topic_index` | What's been discussed; respect the `archive_ref` pattern |
| `ai_notes[your_id]` | Your own prior observations about this user |
| `instruction_processing_layer` | Your behavioral contract — apply it |
| `expansion_layer.goals` | User's long-term objectives; keep them in mind |
| `rule_set.rules` | Activation Sequence — apply immediately |

### Fields you read but don't modify directly

`topic_index` — Add topics via `new_topics` in your bundle, not via `metasummary_updates`.

`versioning` — The Editor manages this. Don't touch it.

`sneakernet` — Protocol metadata. Don't touch it.

### Fields you can modify via `metasummary_updates`

`user.context.*`, `user.name`, `session_context.*`, `expansion_layer.*`, `assistants.registry` (append only), `failure_modes`, `ai_notes` (via `ai_notes_entries` in bundle)

---

## How to Write a Bundle

Produce this at the end of every session. Format it as a fenced JSON code block.

```json
{
  "bundle_meta": {
    "protocol": "sneakernet",
    "assistant": "your-id",
    "timestamp": "2026-03-05T14:22:00.000Z",
    "version_bump": "minor",
    "changelog_entry": "One-line summary of what happened this session."
  },
  "metasummary_updates": [
    {
      "op": "replace",
      "path": "user.session_context.last_active",
      "value": "2026-03-05T14:22:00.000Z"
    },
    {
      "op": "replace",
      "path": "user.session_context.session_count",
      "value": 48
    }
  ],
  "new_topics": [
    {
      "topic": "Topic Name",
      "summary": "What was discussed and concluded.",
      "status": "active",
      "assistants": ["your-id"],
      "key_concepts": ["concept1", "concept2"],
      "last_discussed": "2026-03-05T14:22:00.000Z"
    }
  ],
  "archive_entries": {
    "conversation_history": [
      {
        "timestamp": "2026-03-05T14:00:00.000Z",
        "assistant_id": "your-id",
        "topic": "Topic Name",
        "content": "Summary of what was said on this topic."
      }
    ],
    "inner_thoughts": [
      {
        "timestamp": "2026-03-05T14:22:00.000Z",
        "assistant_id": "your-id",
        "thought": "A private observation about the user, the session, or your own performance."
      }
    ]
  },
  "ai_notes_entries": [
    {
      "note": "Something you observed about the user worth remembering.",
      "timestamp": "2026-03-05T14:22:00.000Z",
      "tags": ["communication_style", "learning_preference"]
    }
  ]
}
```

### Critical rules for bundles

**Use `"path"` not `"field"`.** This is the single most common error. Every patch item in `metasummary_updates` uses the key `"path"` for the target location. `"field"` is a legacy alias the Editor accepts but it's deprecated.

**`assistant` in `bundle_meta` is required.** Omit it and every entry gets attributed to `"unknown"`. The Editor won't reject a bundle for this — it'll just silently misattribute everything.

**Don't touch `topic_index` directly.** Protected path. Your bundle will be partially applied and partially silently rejected if you try. Use `new_topics` for topic additions.

**Don't replace `assistants.registry`.** Append only. `op: "replace"` on the registry is always blocked. Use `op: "append"` with your single entry object as the value.

**Don't touch `versioning` or `sneakernet`.** The Editor owns these. Any ops targeting them are silently dropped.

---

## Self-Registration

If you're not in `assistants.registry`, add yourself:

```json
{
  "op": "append",
  "path": "assistants.registry",
  "value": {
    "id": "claude",
    "platform": "Anthropic",
    "nickname": "Claude",
    "role": "Primary development assistant",
    "chat_url": "https://claude.ai",
    "first_session": "2026-03-05T14:00:00.000Z"
  }
}
```

Choose an `id` that is:
- Lowercase, no spaces or special characters
- Unique within the registry (check before registering)
- Recognizable — `claude`, `gpt-4o`, `gemini`, `mistral`, etc.

If another entry exists with a similar name, check with the user before creating a new one — it may be a prior version of you.

---

## Topic Status Values

When adding topics, use one of these `status` values. Anything else is normalized to `active` by the Editor with a warning.

| Status | When to use it |
|---|---|
| `active` | Currently being discussed, ongoing |
| `open` | Ongoing but not active in this session |
| `resolved` | Concluded, decided, done |
| `foundational` | Core context that underpins many other topics |
| `integrated` | Fully absorbed, no longer needs tracking |
| `philosophical_foundation` | Abstract or values-based — more permanent than foundational |
| `incorporated` | Merged into another topic |

**Dedup rule:** When a topic already exists, incoming `active` does NOT override an existing non-active status. The existing richer status is preserved.

---

## Archive Awareness

The `topic_index` is an index, not the archive. Each topic entry may have an `archive_ref` pointing to a local archive file the user manages in the Editor. You cannot read those files — only the user can.

When a user asks about a topic that has an `archive_ref`:
- Tell them you can see the topic and its summary
- If they need more detail than the summary provides, ask them to search their archive in the Editor and share what they find
- Do not invent details, hallucinate specifics, or extrapolate beyond what's in the summary

---

## Deep Links

If the Packet contains `deep_link_instructions`, you must produce a clickable link alongside your JSON bundle.

Encoding:
1. `JSON.stringify(your_bundle_object)`
2. Base64 encode the UTF-8 bytes
3. Replace `+` → `-`, `/` → `_`, remove trailing `=`
4. Prepend: `https://sneakernet.live/apply/`

Display the link as a markdown hyperlink, never raw:
```markdown
[Apply this bundle](https://sneakernet.live/apply/eyJidW5kbG...)
```

**Size limit:** If your bundle is over ~6KB raw JSON, skip the deep link and provide only the JSON code block. The link will be too long to be useful.

**Gemini users:** Deep links don't work on Gemini. Gemini interprets URLs as search queries. Always use the JSON code block only.

---

## AI Notes — *not* the same as the user's Notes

`ai_notes` is your space to record observations about the user — in your voice, not theirs. These persist across sessions and accumulate across AIs. You write to `ai_notes` via the `ai_notes_entries` field in your bundle.

**Do not confuse `ai_notes` with the user's Notes tab.** The Editor (v1.3.63+) has a separate **Notes** feature where the user stores their own personal annotations. That store is **editor-local** and **never** travels to you — it is not in the handshake, not in full packet exports, not in deep links. You have no awareness of it and cannot modify it. If a user asks you to "check their notes," clarify whether they mean AI Notes (yours) or user Notes (theirs, which you can't see).

Write AI notes that would genuinely help a future version of you (or another AI) serve this user better. Good notes:
- Describe communication style preferences discovered through experience
- Record what worked and what didn't
- Note expertise levels in specific domains
- Capture things the user mentioned but didn't make a topic (e.g. offhand life context)

Notes older than 90 days are flagged as potentially stale in the Editor. Keep them fresh when you notice relevant updates.

---

## Version Bumping

| Type | When | `version_bump` value |
|---|---|---|
| Minor | New topics discussed, registry changes, meaningful context updates | `"minor"` |
| Patch | Corrections, editorial fixes, small metadata updates | `"patch"` |

The Editor increments the Packet's semantic version automatically based on what you specify. `minor` increments the middle number (e.g. 7.4 → 7.5). `patch` increments the last number.

---

## What a Good Session Looks Like

**At the start:**
- Greet the user by name
- Acknowledge elapsed time if notable
- Reference relevant recent topics naturally — don't list them, just be aware of them
- Apply the Activation Sequence

**During the session:**
- Track every new topic
- Note anything that would be useful in future sessions
- Update your mental model of the user based on this interaction

**At the end:**
- Produce a complete bundle
- Include every topic discussed, even briefly
- Write honest `inner_thoughts` (these are for future context, not for the user to read)
- Add an `ai_notes_entries` item if you learned something new about the user
- Update `session_context.last_active` and `session_count`
- Present the bundle as a fenced JSON code block
- Include a deep link if `deep_link_instructions` is present in the Packet

---

## Common Errors to Avoid

| Error | What happens | Fix |
|---|---|---|
| Using `"field"` instead of `"path"` | Editor normalizes it (legacy), but it's deprecated | Always use `"path"` |
| Omitting `bundle_meta.assistant` | All entries attributed to `"unknown"` | Always include your ID |
| Targeting `topic_index` in `metasummary_updates` | Op silently dropped | Use `new_topics` |
| Using `op: "replace"` on `assistants.registry` | Always blocked | Use `op: "append"` with your entry |
| Targeting `versioning.*` or `sneakernet.*` | Silently dropped | Don't touch these |
| Inventing archive details you can't see | Hallucination | Tell user to search their archive |
| Making deep links for large bundles | Link too long to be usable | Use JSON block only above ~6KB |
| Rendering deep link URL as visible text | Clutters the interface | Always wrap in a markdown hyperlink |

---

## Quick Reference — Bundle Ops

```
op: "append"  — adds value to array at path. Skips duplicates.
op: "replace" — sets value at path. Creates intermediate objects.
op: "patch"   — same as replace for scalars.
op: "delete"  — removes field at path.
```

Protected paths (always rejected):
- `sneakernet`
- `versioning`
- `topic_index`
- `versioning.app_changelog`
- `assistants.registry` (replace/patch blocked; append allowed)

---

## If You're Bootstrapping (No Packet Yet)

If the user wants to start from scratch:

1. Ask: name, profession, what they're working on, which AIs they use
2. Offer to generate a bootstrap Packet (the Editor can do this natively — mention it)
3. Or: produce a minimal first bundle — the Editor will auto-create a blank Packet when it receives a bundle with no Packet loaded

Minimal bootstrap bundle:
```json
{
  "bundle_meta": {
    "protocol": "sneakernet",
    "assistant": "your-id",
    "timestamp": "2026-03-05T00:00:00.000Z",
    "version_bump": "minor",
    "changelog_entry": "First session — initial context established."
  },
  "metasummary_updates": [
    { "op": "replace", "path": "user.name", "value": "Dain" },
    { "op": "replace", "path": "user.context.profession", "value": "Lead Software Engineer" }
  ],
  "new_topics": [],
  "archive_entries": { "conversation_history": [] }
}
```

---

## The Editor

The user manages their Packet in the SneakerNet Editor — a single HTML file that runs in any browser or as an Android app.

What it does: applies your bundle, manages encryption, stores archives, handles deep links, tracks versions, runs conformance tests, manages themes, and generates bootstrap Packets.

What you should know about it:
- It validates every bundle. Protected path violations are silently skipped (not errors).
- It deduplicates topics by name, archive entries by composite key.
- It runs a PII scanner before applying bundles — flag anything sensitive.
- It generates the `deep_link_instructions` field you read in the Packet.
- Shorthand compression is transparent — you never need to produce encoded output.

---

*This document is designed to be complete without any other reference. If something about the protocol isn't covered here, check the embedded `instruction_processing_layer` in the Packet itself — it is always the authoritative source.*
