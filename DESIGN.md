# NAPSAC Design Document

**Status:** Final
**Author:** Dennis Kittrell
**Date:** 2026-03-14
**Repo:** Percona-Lab/NAPSAC

---

## Overview

NAPSAC is a Claude plugin that provides persistent AI memory using Notion as the only storage backend. It is a lightweight alternative to PACK — the knapsack you grab when you don't need the full pack.

NAPSAC exposes five tools (`memory_init`, `memory_list`, `memory_get`, `memory_update`, `memory_search`) that match PACK's signatures so system prompts work interchangeably between the two products. Unlike PACK, NAPSAC requires zero infrastructure: no GitHub repo, no MCP server, no API tokens, no config files, no local install.

The plugin is the authoritative implementation, providing validated naming, enforced page structure, and guaranteed index regeneration. A system prompt fallback serves tools where the plugin isn't available (Claude mobile, ChatGPT, Cursor, VS Code).

---

## Why Notion as Backend

### The problem NAPSAC solves

PACK is powerful but has a setup cost: GitHub repo, personal access token, local MCP server, env file. For users who primarily use Claude, ChatGPT, or other tools with native Notion access, this infrastructure is unnecessary friction.

Notion solves this because:

1. **Universal access.** Notion works on every device — desktop, mobile, tablet, web. Memory is always reachable.
2. **Zero infrastructure.** No server to run, no repo to maintain, no tokens to rotate. OAuth handles auth.
3. **Native connectors exist.** Claude has a built-in Notion connector. ChatGPT, Cursor, and VS Code support Notion MCP. The plumbing already exists.
4. **Version history built in.** Notion tracks page edit history natively. No git needed for audit trails.
5. **Sharing is native.** Want to share specific memory with a teammate? Share the Notion page. Permissions are granular.
6. **Human-editable.** Users can read and edit their memory directly in Notion. No terminal required.

### Tradeoffs vs GitHub-backed storage (PACK)

| Dimension | NAPSAC (Notion) | PACK (GitHub) |
|-----------|-----------------|---------------|
| Setup | Zero config | GitHub token + MCP server |
| Version history | Notion page history (limited) | Full git log with commits, diffs, SHAs |
| CLI access | None | Full CLI (`pack list`, `pack get`, etc.) |
| Offline access | No (requires Notion API) | Yes (with local git connector) |
| Token efficiency | Notion API overhead per request | Direct file reads, minimal overhead |
| Concurrency | Last-write-wins | SHA-based optimistic concurrency |
| Webhook/sync | None | Webhook + built-in Notion/Google Docs sync |
| Backup/export | Notion export only | Git clone, standard backup tools |
| Encryption | Notion's encryption at rest | User controls repo encryption |
| Multi-client | Any tool with Notion access | Any MCP client |

### Why this exists alongside PACK

NAPSAC and PACK serve different user profiles:

- **NAPSAC user:** Non-developers who use Cowork as their primary AI environment. PMs, marketers, founders, ops people. Wants memory to "just work" across devices. Values simplicity over control.
- **PACK user:** Developers who use Claude Code, Cursor, Windsurf, Open WebUI. Needs git history and CLI. Wants complete control over storage. Values auditability over convenience.
- **Both:** Power user who wants PACK's git history and CLI for serious work, but also wants mobile access. Runs PACK with Notion sync, uses NAPSAC's system prompt as a fallback.

Neither product replaces the other. They share conventions so migration between them is seamless.

---

## Design Principles

- **Zero infrastructure.** No local server, no install, no config files. If Notion is connected, it works.
- **Portable.** Same file path conventions as PACK. Content migrates freely between the two.
- **Plugin-first.** NAPSAC is a Claude plugin for Cowork and Claude Code. The plugin is the product. The system prompt is the fallback.
- **Prompt-compatible.** System prompts work interchangeably with PACK. Swap one for the other without rewriting prompts.
- **Notion-native.** Content is stored as proper Notion blocks, not raw markdown dumps. Pages look good in Notion.

---

## Architecture

### Why a Plugin and Not an MCP Server

NAPSAC's entire value proposition is zero config. An MCP server requires local install, config files, and running a process. That's PACK.

A Claude plugin runs inside the session. It uses Claude's native Notion connector for auth. No tokens, no env files, nothing to install.

| Consideration | Plugin | MCP Server |
|--------------|--------|------------|
| Installation | Claude installs from GitHub URL | `npm install`, local config |
| Auth | Notion OAuth via Claude's connector | API tokens in env file |
| Port conflicts | None (runs in-session) | Needs available port |
| Mobile | Works in Cowork | Requires desktop running |
| Cross-tool | Plugin = Cowork/Claude Code only | Works in any MCP client |
| Validation | Behavior spec in SKILL.md | Code-level validation |

The tradeoff: the plugin only works in Cowork and Claude Code. For other tools, the system prompt fallback provides the same memory access without the plugin's validation guardrails.

### Plugin Architecture

```
┌─────────────────────────────────────────┐
│              Claude Session             │
│                                         │
│  ┌───────────────┐  ┌───────────────┐  │
│  │ NAPSAC Plugin │  │ Notion MCP    │  │
│  │ (SKILL.md)    │──│ (native)      │  │
│  │               │  │               │  │
│  │ memory_init   │  │ notion-search │  │
│  │ memory_list   │  │ notion-fetch  │  │
│  │ memory_get    │  │ notion-create │  │
│  │ memory_update │  │ notion-update │  │
│  │ memory_search │  │               │  │
│  └───────────────┘  └───────┬───────┘  │
│                              │          │
└──────────────────────────────┼──────────┘
                               │
                    ┌──────────▼──────────┐
                    │    Notion API       │
                    │  (OAuth, no tokens) │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  User's Notion      │
                    │  Workspace          │
                    │                     │
                    │  [Memory Root]      │
                    │  ├── _index         │
                    │  ├── _conventions   │
                    │  ├── context/       │
                    │  ├── projects/      │
                    │  ├── profiles/      │
                    │  └── contacts/      │
                    └─────────────────────┘
```

NAPSAC is a skill definition (SKILL.md) that instructs Claude how to use Notion MCP tools to implement memory operations. There is no server process, no compiled code, no runtime. The "plugin" is a behavior specification that Claude follows — but it provides real validation, naming enforcement, and guaranteed index regeneration that the system prompt alone cannot.

### The System Prompt Fallback Model

```
┌─────────────────────────────────────────────────┐
│            Plugin (Cowork / Claude Code)          │
│   Validated naming, enforced structure,           │
│   guaranteed index regen, error handling          │
├─────────────────────────────────────────────────┤
│       System Prompt (Claude mobile/web,           │
│           ChatGPT, Cursor, VS Code)               │
│   Best-effort compliance with _conventions,       │
│   depends on AI reading and following rules        │
├─────────────────────────────────────────────────┤
│            Native Memory (last resort)            │
│   No persistence. Context may be incomplete.      │
└─────────────────────────────────────────────────┘
```

The plugin is the reliable, validated tier. The system prompt is the cross-tool compatibility tier. Native memory is the last resort.

### How Tools Map to Notion Operations

| NAPSAC Tool | Notion Operations Used |
|-------------|----------------------|
| `memory_init` | Create pages (index + conventions + starter content) |
| `memory_list` | Fetch index page content |
| `memory_get` | Search by title → fetch page content |
| `memory_update` | Search by title → update or create page → regenerate index |
| `memory_search` | Notion search API across all child pages |

---

## Page Structure Design

### Why sub-pages (not a database)

Notion databases were considered but rejected:

1. **Page hierarchy is more natural.** Directories map to parent pages. Files map to child pages. This mirrors PACK's filesystem structure.
2. **No schema to maintain.** Databases require property definitions. Pages are freeform.
3. **Better for rich content.** Database entries are optimized for structured data. Memory files contain freeform text with headings, bullets, and code blocks.
4. **Simpler API operations.** Creating a child page is one API call. Creating a database entry with properties is more complex.

### Why full paths in page titles

Each leaf page's title is its full path (e.g., `context/preferences`, not just `preferences`). This ensures:

1. **Unique identification.** No ambiguity between files in different directories.
2. **Search-friendly.** Searching for `context/preferences` in Notion returns exactly one result.
3. **PACK compatibility.** Paths match PACK's file paths (minus the `.md` extension).

### The _conventions page

The `_conventions` page is the bridge between the plugin and the system prompt fallback.

In plugin mode: The plugin validates naming and structure directly. `_conventions` exists as human-readable documentation.

In system prompt mode: The system prompt instructs the AI to read `_conventions` before writing. This is the only mechanism that ensures naming consistency, directory structure, and content format across tools like ChatGPT and Cursor.

`_conventions` is created once during `memory_init` and never modified during normal operations. It is a system page.

### Index page design

The `_index` page is auto-regenerated on every `memory_update`. It serves the same purpose as PACK's `index.md`:

1. **Session start discovery.** Agents read this first (~700 tokens) to decide what to load.
2. **Human overview.** Users browsing Notion see a summary of everything in memory.
3. **Cross-tool compatibility.** Any AI tool that can read a Notion page can use the index.

The underscore prefix (`_index`) keeps it sorted before alphabetical content in Notion's sidebar.

---

## Concurrency Model

PACK uses SHA-based optimistic concurrency to prevent race conditions. NAPSAC uses Notion's native last-write-wins model.

**Why no optimistic concurrency:**

1. **Single-user product.** NAPSAC is personal memory. Concurrent writes from multiple agents are rare.
2. **Notion doesn't expose ETags.** There's no lightweight way to detect version conflicts via the Notion API.
3. **Page history is the safety net.** If a write overwrites something important, Notion's page history allows recovery.
4. **Simplicity.** Removing concurrency control eliminates an entire class of error handling.

**Risk:** If two AI sessions write to the same file simultaneously, the last write wins. This is acceptable because:
- Memory files are personal, not shared
- Simultaneous sessions writing the same file is extremely rare
- Notion page history allows recovery of overwritten content

---

## Content Format

### Why Notion blocks instead of raw markdown

NAPSAC converts markdown input to proper Notion blocks:

1. **Native rendering.** Pages look correct in Notion's UI. Raw markdown in a page looks broken.
2. **Editability.** Users can edit memory in Notion's editor naturally.
3. **Consistency with BINER.** BINER formats pages using Notion blocks. Memory pages should follow the same standard.

### Conversion rules

```
## Heading      → heading_2 block
### Heading     → heading_3 block
- item          → bulleted_list_item block
1. item         → numbered_list_item block
`code block`    → code block
> quote         → quote block
**bold**        → bold rich text
*italic*        → italic rich text
plain text      → paragraph block
```

### Read format

`memory_get` returns content in clean readable format — not raw Notion block JSON. The plugin converts Notion blocks back to clean text that Claude can consume efficiently.

---

## Error Handling Design

### Retry strategy

Notion API failures use exponential backoff: 1s, 2s, 4s. Three attempts maximum.

**Why 3 attempts:** Notion's API is generally reliable. Transient failures (rate limits, network blips) typically resolve within seconds. More than 3 retries risks masking a real problem.

**Never lose a write silently.** If all retries fail, the error message explicitly states that the content was not saved and suggests retrying.

### Graceful degradation

The three-tier fallback (plugin → system prompt/Notion → native memory) ensures some level of context is always available. The system prompt enforces trying each level in order.

---

## Design Decisions

| # | Question | Decision | Rationale |
|---|----------|----------|-----------|
| 1 | Storage model | Notion sub-pages, not database | Freeform content, simpler API, mirrors filesystem |
| 2 | Concurrency | Last-write-wins | Single-user product, Notion has no ETags, page history provides safety |
| 3 | Content format | Notion blocks, not raw markdown | Native rendering, user-editable, BINER consistency |
| 4 | Page naming | Full file paths as titles | Unique identification, search-friendly, PACK-compatible |
| 5 | Index naming | `_index` with underscore prefix | Sorts first in sidebar, avoids conflict with user content |
| 6 | Plugin vs MCP server | Plugin (SKILL.md) | Zero infrastructure, mobile-native, reuses existing Notion connector |
| 7 | Tool signatures | Match PACK exactly | System prompt portability, same mental model |
| 8 | File extensions | Omitted (no `.md`) | Notion pages don't have extensions; stripped on migration from PACK |
| 9 | Directory representation | Parent pages with `/` suffix | Natural Notion hierarchy, browseable in sidebar |
| 10 | Version tracking | Notion page history | No infrastructure, built-in, good enough for personal memory |
| 11 | Plugin vs system prompt | Both — plugin is authoritative, prompt is fallback | Plugin provides validation; prompt provides cross-tool reach |
| 12 | _conventions page | Created once, never auto-modified | Human-readable docs for non-plugin environments; plugin validates directly |

---

## What NAPSAC Intentionally Does Not Do

| Feature | Why Not |
|---------|---------|
| GitHub integration | That's PACK's domain. NAPSAC is Notion-only by design. |
| CLI | No local infrastructure. Memory is accessed through AI tools or Notion directly. |
| Sync to external targets | No infrastructure to run sync jobs. Use PACK if you need sync. |
| Webhook support | No server to receive or send webhooks. |
| MCP server | NAPSAC is a plugin, not a server. The distinction is intentional. |
| API tokens | OAuth only. No tokens for users to manage. |
| Local storage | Everything lives in Notion. No local state, no config files. |
| Encryption | Relies on Notion's encryption at rest. For encrypted memory, use PACK with an encrypted git repo. |
