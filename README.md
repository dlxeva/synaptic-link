# Synaptic Link

**A lightweight retrieval layer for AI agents over personal second brains.**

> Part of the [Institute of Artificial Absurdity](https://aizhiz.com/) notes on agent memory and personal knowledge bases. 相关随笔：[AI 时代的个人知识库，正在变成吃灰的收藏夹](https://aizhiz.com/zh/posts/2026-10-18-personal-knowledge-base-saved-stuff/)

[中文版](README.zh-CN.md)

---

## The Problem

AI agents have a finite context window. Personal knowledge bases grow indefinitely.

You can't inject your entire Obsidian vault into every prompt — it's too expensive, too noisy, and causes context overflow. But you also can't rely on blind file search every time.

What's missing is a **middle layer**: fast enough to query on every turn, precise enough to surface what's actually relevant, transparent enough that both you and your agent understand exactly what was retrieved and why.

Synaptic Link is that layer.

---

## Where It Fits

A complete second-brain agent system has four layers:

```
┌─────────────────────────────────────────────┐
│  Host Layer        Obsidian Vault            │
│                    Projects · Library ·      │
│                    Archive · Daily · System  │
└───────────────┬─────────────────────────────┘
                │
┌───────────────▼─────────────────────────────┐
│  Write Layer       Archivist Agent           │
│                    Captures decisions,       │
│                    conversations, reviews    │
│                    into structured notes     │
└───────────────┬─────────────────────────────┘
                │
┌───────────────▼─────────────────────────────┐  ← this project
│  Retrieval Layer   Synaptic Link             │
│                    index / search / read     │
│                    SQLite FTS5, zero deps    │
└───────────────┬─────────────────────────────┘
                │
┌───────────────▼─────────────────────────────┐
│  Behavior Layer    Agent prompt / policy     │
│                    When to search, how to    │
│                    incorporate results       │
└─────────────────────────────────────────────┘
```

**Synaptic Link covers only the retrieval layer.** It does not define when to call search, how to use the results, or how to write notes. Those belong to the layers above and below it.

---

## Progressive Disclosure

The retrieval layer implements a three-tier access pattern:

```
Always in context
    Hot Cache ── SOUL / MEMORY / recent logs (injected into system prompt)
        │
        │ not enough?
        ▼
    Synapse Index ── SQLite FTS5 over all notes
        │               index   →  build this layer
        │               search  →  query this layer
        │ hit?
        ▼
    Deep Knowledge ── full Markdown files
                        read    →  access this layer
```

**Only retrieve what you need, when you need it.** The agent queries the index first (< 50 ms), reads the full file only on a confirmed hit, and skips everything else.

---

## Interface

Three commands. That's the core.

```bash
# Build the index
python synapse.py index [--vault PATH] [--db PATH]

# Search — returns title, section, path, and a highlighted snippet
python synapse.py search "query" [--limit N] [--json]

# Read a full note after a search hit
python synapse.py read "/absolute/path/to/note.md"
```

### Search output

Human-readable (default):
```
## Note Title § Section Heading
   /path/to/note.md
   ...matched **snippet** with highlights…
```

Machine-readable (`--json`):
```json
[
  {
    "title": "Note Title",
    "section": "Section Heading",
    "path": "/path/to/note.md",
    "snippet": "...matched **snippet**…"
  }
]
```

The `§ Section` field gives paragraph-level context. Your agent can use it to decide whether the snippet is already sufficient or whether a full `read` is needed.

---

## Design Principles

**Local-first.** Your notes never leave your machine. No cloud sync, no embeddings API required for core functionality. The index is a single SQLite file you can open with any database browser.

**Human-machine readable.** Every note is plain Markdown. The index is inspectable. Search output is readable in a terminal and consumable as JSON. Nothing is locked in opaque binary formats. Both you and your agent work with the same data.

**Tool, not policy.** Synaptic Link provides retrieval capability. When to invoke it, how to incorporate results, what threshold triggers a `read` — those decisions belong to the agent's system prompt. The tool stays out of the way.

**Progressive disclosure.** Don't load more than needed. `search` first, `read` only on a confirmed hit. Information unfolds with demand, not upfront.

**Lightweight by design.** Zero dependencies for `index`, `search`, `read`. No embedding model, no vector store, no external server. SQLite FTS5 trigram tokenizer handles CJK and Latin without extra configuration.

---

## Setup

**Requirements:** Python 3.8+, SQLite 3.35+ (ships with Python 3.9+ on most platforms)

```bash
# 1. Point it at your vault
export OBSIDIAN_VAULT=/path/to/your/vault
export SYNAPTIC_DB=~/.synaptic-link/synapse.db   # optional, defaults to script dir

# 2. Build the index
python scripts/synapse.py index

# 3. Search
python scripts/synapse.py search "your query"

# 4. Keep the index current (optional, requires: pip install watchdog)
python scripts/synapse.py watch
```

**Notes:**
- Files prefixed with `_draft_` are excluded from the index
- Queries must be 3+ characters (SQLite trigram limitation)
- Use `--full` to force a complete rebuild: `synapse.py index --full`

---

## Integrations

### OpenClaw

Copy `SKILL.md` and `scripts/synapse.py` to your OpenClaw skills directory:

```
~/.openclaw/skills/synaptic-link/
├── SKILL.md
└── synapse.py
```

The agent will use the skill description in `SKILL.md` to decide when to invoke search.

### Claude Code

Add to your project's `CLAUDE.md`:

```markdown
## Knowledge Base (Synaptic Link)

Before answering questions about past decisions, project history, or personal
notes, search the local knowledge base:

    python /path/to/synapse.py search "<keywords>" --json

If the snippet is insufficient, read the full note:

    python /path/to/synapse.py read "<path from search result>"

Do not answer questions about project history from memory alone.
```

### Any agent with shell access

If your agent can run a subprocess, it can use Synaptic Link. The `--json` flag
produces structured output suitable for any programmatic consumer.

---

## What This Is Not

- Not a complete memory system (no write path, no conversation capture)
- Not a vector / semantic search tool (keyword retrieval only in v1.0)
- Not a cloud service or SaaS product
- Not an Obsidian plugin
- Not tied to any specific agent runtime

---

## Roadmap

| Version | Scope |
|---------|-------|
| **v1.0** | `index` / `search` / `read`, FTS5 trigram, section-level chunking, `--json` output |
| v1.1 | Incremental indexing, file watcher (`watch` command) |
| v1.2 | Optional vector search via local embedding API (Ollama / compatible) |
| v1.3 | Hybrid retrieval (BM25 + vector, RRF fusion) |
| future | MCP server wrapper |

---

## Background

This project grew out of a personal multi-agent system built on [OpenClaw](https://github.com/OpenClaw/openclaw), where one agent handles conversations and another (an archivist) maintains a structured Obsidian vault. The retrieval problem — how does the conversation agent access vault knowledge without loading the whole vault into context — led to this tool.

The three-layer access pattern (hot cache → synapse index → deep knowledge) is a practical application of progressive disclosure to agent memory design.
