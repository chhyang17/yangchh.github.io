---
layout: post
title: "Building a Cross-Device, AI-Organized Obsidian Note System"
date: 2026-08-17
---
# Building a Cross-Device, AI-Organized Obsidian Note System (Syncthing + Local LLM Wiki)

I wanted a note system that follows me across my work PC, home PC, and phone, lets me write in a messy stream-of-consciousness style, and still ends up organized — without me doing the organizing. Here's what I actually built, including the mistakes along the way.

## The goal

- One Obsidian vault, edited from three devices (work laptop, home PC, phone)
- Write raw, unstructured notes ("flow" notes — todos, references, random thoughts, mixed languages)
- Let an LLM turn that mess into something structured and queryable later, instead of hand-filing everything into folders as I write

## Part 1 — Syncing the vault across devices with Syncthing

The sync layer is just [Syncthing](https://syncthing.net/) on the two PCs, plus a Syncthing-compatible Android client for the phone, all pointed at the same folder ID — which is the same folder Obsidian opens as its vault.

### The mistake: same Folder ID ≠ connected

The first time I set this up, I created the folder with the same Folder ID on all three devices and assumed that alone would sync them. It doesn't. A shared Folder ID only tells Syncthing "these are meant to be the same folder" — it does **not** establish a sync relationship by itself. Two separate things have to happen on top of that:

1. **Devices must be mutually added.** Each device needs the others' Device ID added under "Remote Devices," and each side has to accept the connection.
2. **The folder must be explicitly shared with each device**, and the receiving device has to **accept the share invitation** that pops up. Skipping this (or dismissing the popup) leaves the folder sitting there un-shared even with identical Folder IDs.

If a new note isn't showing up anywhere else, and a file's context menu says something like *"this file is not fully available on any connected device,"* that's Syncthing telling you it has no peer to push to — check both of the above first.

### Android gotcha: storage permissions

On Android, scoped storage can silently limit Syncthing to its own app-private directory instead of the public folder Obsidian actually writes to. If the sync folder looks connected but stays empty, check that the Syncthing app has full/"manage external storage" permission, and that the folder path you picked is the real Obsidian vault path, not a phantom empty one.

### Excluding `.obsidian` from sync

Once notes were syncing, editing on two devices around the same time started producing conflict files — not on the notes themselves, but on `.obsidian/core-plugins.json`, `workspace.json`, etc. These are per-device app state (which plugins are on, current window layout), not content, so they don't belong in sync at all. Fix: add an ignore pattern per device in the folder's Syncthing settings:

```
.obsidian
```

(Ignore patterns are per-device, so this has to be added on each machine separately.) This eliminated the recurring "Conflict Resolver" popups entirely.

## Part 2 — Folder structure: less is more

I originally over-designed the vault with folders like `01-personal`, `02-projects`, `03-todo`, `04-reference`, `05-daily`. In practice that just added friction to the "write raw" goal — I'd hesitate over where something belonged instead of just writing it down.

Simplified to:

```
vault/
├── 00-inbox/        # everything goes here first, no upfront categorization
├── 04-reference/     # a small set of things worth keeping long-term
└── attachments/
```

The plan: write daily, unsorted, mixed-language notes into `00-inbox`, and periodically have an LLM re-organize the backlog rather than sorting by hand.

## Part 3 — Getting an LLM to actually use the vault

There are a few different ways to connect an LLM assistant to a local vault, and they trade off differently depending on which client you're in:

- **Claude Code**, run from the vault directory, gets direct read/write access with zero extra config — good for hands-on editing sessions.
- **Claude Desktop + a filesystem MCP connector** lets you point a chat-style conversation at the vault, but this only works in the desktop client, not the web/mobile app.
- **An in-Obsidian plugin** that ingests notes and builds a queryable structure works everywhere Obsidian runs (including mobile), independent of which chat client you're using.

Since I use Claude across web, mobile, and desktop interchangeably, and Claude's own memory (which persists across chats) already covers a lot of "what did we talk about last time," the missing piece was specifically: **detailed content I never actually typed into a chat with Claude, but did write down in the vault.** For that, an in-vault plugin made the most sense.

### Karpathy LLM Wiki plugin

I went with the [Karpathy LLM Wiki plugin](https://community.obsidian.md/plugins/karpathywiki) — an implementation of [Andrej Karpathy's "LLM wiki" idea](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f). It reads your notes, has an LLM extract entities/concepts into linked wiki pages (`[[wiki-links]]`), and retrieves via graph expansion (Personalized PageRank over the link graph) rather than embeddings/vector search — no vector DB, no separate indexing pipeline.

Workflow: **Ingest single source** on a note → it generates entity/concept pages under `wiki/` → **Query wiki** opens a chat panel that answers from those pages. Your original notes are never modified.

### Running it against a local model: two real bugs

I wanted this fully local via Ollama rather than sending notes to a cloud API. Ran into two separate issues getting there:

**1. Ollama models not showing up in the plugin, despite showing in Ollama's own GUI**

This is a CORS issue, not a missing-model issue. The plugin runs inside Obsidian's app context (`app://obsidian.md` origin) and calls Ollama's local HTTP API to list models — but Ollama only allows cross-origin requests from an explicit allowlist, and `app://obsidian.md` isn't on it by default. Ollama's own GUI/CLI don't go through this restriction, which is why the model was visible there but not in the plugin.

Fix — set an environment variable and restart Ollama:

```
OLLAMA_ORIGINS=app://obsidian.md*
```

On Windows: System Properties → Environment Variables → add `OLLAMA_ORIGINS` as a user variable → fully quit and restart Ollama (not just close the tray icon — if it's installed as a background service, the service needs restarting, not just the tray app).

**2. Raw model "thinking" text leaking into every response**

Using a community-quantized GGUF (`hf.co/unsloth/gpt-oss-20b-GGUF`) pulled directly via Ollama's HuggingFace integration, responses came back with visible internal reasoning mixed into the output — text like *"The user is asking X... so we should respond with Y"* appearing before the actual answer.

Root cause: `gpt-oss` models use OpenAI's "Harmony" response format, which separates an internal `analysis` channel (reasoning) from the `final` channel (the actual answer) via a specific chat template. Ollama's **official** library build of the model (`ollama pull gpt-oss:20b`) has this template correctly wired in. Some independently-uploaded community GGUFs don't carry (or don't correctly expose) that template, so the two channels never get separated and the reasoning leaks straight into the visible output. This isn't just cosmetic — the plugin's entity/concept extraction depends on getting clean structured output from the model, so the same bug was also causing ingestion to silently fail (queries came back saying the wiki was empty even after "successfully" ingesting).

Fix: pull the official model instead of the community GGUF —

```
ollama pull gpt-oss:20b
```

— then point the plugin at that model. Ingestion and query output were clean afterward.

## Where it landed

- Notes sync reliably across all three devices via Syncthing, with `.obsidian` excluded to stop config-file conflicts
- The vault stays a flat, low-friction `inbox/` for daily writing
- A local, fully-offline LLM (via Ollama + the Karpathy LLM Wiki plugin) periodically turns that inbox into linked entity/concept pages I can query — e.g. asking "what do I need to do this week" pulls from notes I never manually filed anywhere
- Cross-language duplicate entities (e.g. "CRA" vs its Chinese name) are a known rough edge — the plugin's **Lint wiki** command is meant to catch and merge these, worth running periodically

## Takeaways

- "Same Folder ID" in Syncthing is necessary but not sufficient — device trust *and* explicit folder sharing (with acceptance on both ends) are separate steps people commonly miss
- Exclude app config directories (`.obsidian`, `.git`, etc.) from file sync tools; they're per-device state, not content
- If a local-model integration in an Electron-based app (Obsidian, and plenty of others) can't see your Ollama models, suspect CORS before anything else
- For "reasoning" models like gpt-oss, prefer the inference tool's official/first-party model build over a community re-upload — chat template correctness for channel-separated output isn't guaranteed to survive the round-trip
