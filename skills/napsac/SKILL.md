---
name: napsac
description: >
  NAPSAC (Notion Agent Persistent Store for Accessible Context) provides
  persistent AI memory using Notion as the storage backend. Use this skill
  when the user asks to initialize, read, write, search, or list their
  persistent memory — or when starting any conversation (call memory_list
  automatically). Trigger on: "remember this", "save this to memory",
  "what do you know about", "check your memory", "memory init", "search
  memory", "update memory", "list memory", or any reference to persistent
  context. Also trigger at conversation start to load context automatically.
  NAPSAC is a lightweight alternative to PACK — it uses Notion as the only
  backend with zero config, no GitHub, no MCP server, no API tokens.
---

# NAPSAC — Notion Agent Persistent Store for Accessible Context

A persistent memory system that uses Notion as the only storage backend.
Zero config, zero infrastructure. Works immediately with Claude's native
Notion connector.

Named for the knapsack — the lightweight daypack you grab when you don't
need the full PACK. Part of the Alpine toolkit.

---

## Overview

NAPSAC stores memory as Notion sub-pages under a user-designated root page.
It provides five tools that match PACK's signatures so system prompts work
interchangeably between the two products.

**Tools:**

1. **`memory_init`** — Set up a new memory root in Notion
2. **`memory_list`** — List all memory files (call at session start)
3. **`memory_get`** — Read a specific memory file
4. **`memory_update`** — Write or create a memory file
5. **`memory_search`** — Search across all memory files

---

## Memory Backend

NAPSAC uses Notion exclusively. All reads and writes go through Claude's
native Notion connector (Notion MCP tools).

**Required Notion MCP tools:**
- `notion-search` — Find pages by title
- `notion-get-page` / `notion-fetch` — Read page content
- `notion-create-page` / `notion-create-pages` — Create new pages
- `notion-update-page` — Update existing pages

**No other infrastructure.** No GitHub, no local server, no API tokens,
no config files. If the user has Notion connected, NAPSAC works.

---

## Page Structure

Memory is organized as Notion sub-pages under a root page:

```
[Memory Root Page]
├── _index (auto-managed, lists all files with links)
├── contacts/
│   └── key-people
├── context/
│   ├── general
│   ├── preferences
│   └── projects-overview
├── profiles/
│   └── mynah-styles
└── projects/
    ├── project-alpha
    └── project-beta
```

**Conventions:**
- "Directories" are Notion pages whose title ends with `/` and contain
  child pages. They are organizational containers only.
- "Leaf pages" contain actual memory content. Their title is the full
  file path (e.g., `context/preferences`).
- The `_index` page is auto-generated on every `memory_update`.
- Page titles use the file path as the identifier for easy lookup.

---

## Tool Implementations

### memory_init

**Purpose:** Set up a new memory root in Notion.

**Input:**
- `page` (required) — Notion page URL or ID to use as the memory root

**Behavior:**
1. Validate that the Notion connector is available. If not, return:
   `"Notion connector is not connected. Enable it in your AI tool's settings to use NAPSAC memory."`
2. Fetch the provided page to confirm it exists and is accessible.
3. Create two child pages under the root:
   - `_index` — Empty index page (will be populated on first update)
   - `context/general` — Starter memory file with content:
     ```
     ## General Context

     This is the default memory file. Save general context and preferences here.

     *Created by NAPSAC on [date].*
     ```
4. Store the root page ID internally for subsequent calls. When running
   as a Claude plugin, persist this in the plugin's config. When running
   via system prompt only, include the root page ID in the prompt.
5. Return confirmation with the root page URL.

**Error handling:**
- If the page URL/ID is invalid or not found: return a clear error with
  the provided value.
- If Notion connector is unavailable: return the standard connector error.

---

### memory_list

**Purpose:** Return a lightweight list of all memory files. This is the
first call at every session start.

**Input:**
- `tag` (optional) — Filter by tag
- `topic` (optional) — Filter by topic
- `dir` (optional) — Filter by directory prefix (e.g., `projects/`)

**Behavior:**
1. Find the `_index` page under the memory root.
2. Read its content and return it.
3. If filters are provided, parse the index and return only matching entries.

**Output format:** The index content, which lists all files with:
- File path
- Description (first line of content or explicit description)
- Tags (if present in content)
- Last updated timestamp
- Link to the Notion sub-page

Target: ~700 tokens for ~25 files.

**Error handling:**
- If root page is not set: `"Memory root not configured. Run memory_init with a Notion page URL to get started."`
- If `_index` not found: `"No index found. Run memory_init to set up your memory root."`

---

### memory_get

**Purpose:** Read a specific memory file by path.

**Input:**
- `path` (optional) — File path (e.g., `context/preferences`).
  If omitted, concatenates all files and returns them (v1 compat).
- `all` (optional, boolean) — If true, concatenate and return all files.

**Behavior:**
1. If `path` is provided:
   a. Search for a child page under the root whose title matches the path.
   b. Fetch the page content.
   c. Convert Notion blocks to clean readable text.
   d. Return the content.
2. If `path` is omitted or `all` is true:
   a. Read the `_index` page.
   b. Fetch all linked pages.
   c. Concatenate their content with section headers.
   d. Return the combined content.

**Output:**
```
content:     string — page content in clean readable format
updated_at:  string — page last edited timestamp
page_id:     string — Notion page ID (used for subsequent updates)
```

**Error handling:**
- If path not found: `"Memory file not found: [path]. Available files can be seen with memory_list."`
- Never fail silently on missing pages.

---

### memory_update

**Purpose:** Write or create a memory file. Auto-updates the index.

**Input:**
- `path` (required) — File path (e.g., `context/preferences`)
- `content` (required) — Content to write (markdown format)
- `message` (optional) — Description of the change (stored in page history)

**Behavior:**
1. Search for an existing page under the root whose title matches `path`.
2. **If page exists:**
   a. Replace the page content with the new content.
   b. Convert markdown to proper Notion blocks (headings, bullets, code
      blocks, etc.) — not raw markdown dumped into the page.
3. **If page does not exist:**
   a. Determine the directory from the path (e.g., `context/` from
      `context/preferences`).
   b. Find or create the directory page under the root.
   c. Create a new child page with the path as its title.
   d. Write the content as Notion blocks.
4. **After write, regenerate the `_index` page:**
   a. List all child pages under the root (excluding `_index` itself).
   b. For each page, extract: path (title), description (first line),
      tags (if present), last edited timestamp, page link.
   c. Format as a lightweight table/list.
   d. Replace the `_index` page content.
5. Return confirmation with the page ID and path.

**Output:**
```
path:          string — the file path written
page_id:       string — Notion page ID
index_updated: boolean — confirms index was regenerated
```

**Content conversion rules:**
- `## Heading` → Notion heading_2 block
- `### Heading` → Notion heading_3 block
- `- item` → Notion bulleted_list_item block
- `1. item` → Notion numbered_list_item block
- `` `code` `` → Notion code block
- `> quote` → Notion quote block
- `**bold**` → Bold rich text
- `*italic*` → Italic rich text
- Plain paragraphs → Notion paragraph blocks

**Error handling:**
- If Notion write fails, retry with exponential backoff (1s, 2s, 4s).
  After 3 attempts, return error. Never lose a write silently.
- If `path` is omitted: route to `context/general` with a deprecation
  warning (v1 compat, matches PACK behavior).

---

### memory_search

**Purpose:** Search across all memory files by content.

**Input:**
- `query` (required) — Search keywords

**Behavior:**
1. Use the Notion search API to search across all pages under the root.
2. For each matching page:
   a. Extract the file path from the page title.
   b. Extract a snippet of matching content.
3. Return results sorted by relevance.

**Output:**
```
results: array of:
  path:    string — file path
  topic:   string — from page title
  tags:    string[] — if present
  snippet: string — matching excerpt
```

**Error handling:**
- If no results: return empty array with a message suggesting the user
  try different keywords.
- If search API unavailable: fall back to reading all pages and doing
  local substring matching.

---

## Index Page Format

The `_index` page is regenerated on every `memory_update`. It lists all
memory files in a scannable format:

```markdown
# Memory Index

Last updated: 2026-03-14T10:30:00Z
Files: 12

## context/

| File | Description | Tags | Updated |
|------|-------------|------|---------|
| context/general | General context and preferences | general | 2026-03-14 |
| context/preferences | Communication and tool preferences | preferences, tools | 2026-03-12 |
| context/projects-overview | Active projects summary | projects | 2026-03-13 |

## projects/

| File | Description | Tags | Updated |
|------|-------------|------|---------|
| projects/project-alpha | Alpha project tracking | alpha, q2-2026 | 2026-03-14 |
| projects/project-beta | Beta project notes | beta, planning | 2026-03-10 |

## profiles/

| File | Description | Tags | Updated |
|------|-------------|------|---------|
| profiles/mynah-styles | MYNAH writing style profiles | mynah, communication | 2026-03-08 |

## contacts/

| File | Description | Tags | Updated |
|------|-------------|------|---------|
| contacts/key-people | Key contacts and relationships | contacts | 2026-03-11 |
```

---

## Session Start Behavior

At the start of every conversation:

1. Call `memory_list` to load the index.
2. Call `memory_get` on `context/preferences` (if it exists per the index).
3. Scan the index for files relevant to the user's first message.
4. Load relevant files with `memory_get` as needed.

This mirrors PACK's session start behavior exactly.

---

## MYNAH and BINER Compatibility

### MYNAH Profiles

MYNAH style profiles are stored under `profiles/` with the naming
convention `profiles/mynah-{context-id}` (e.g., `profiles/mynah-slack-dm-tech`).

When MYNAH learns a writing style:
1. The profile is saved via `memory_update` to the appropriate path.
2. The profile format matches PACK's MYNAH profile schema exactly.
3. Any AI tool reading the memory can find and use these profiles.

In **Cowork and Claude Code**: The MYNAH plugin works alongside NAPSAC
natively — it trains profiles and stores them via NAPSAC's tools.

In **other clients**: Profiles are read-only. Training requires Cowork
or Claude Code where plugins are supported.

### BINER Design Profiles

BINER design profiles are stored at `profiles/notion-design` (or
`profiles/biner-{profile-name}` for named profiles).

When BINER learns a Notion design style:
1. The profile is saved via `memory_update`.
2. The format matches PACK's BINER profile schema exactly.
3. Any AI tool creating Notion pages can apply these preferences.

### Profile Portability

Because NAPSAC and PACK use the same file path conventions and profile
formats, profiles are portable between the two systems:
- A MYNAH profile trained via PACK works in NAPSAC (and vice versa).
- A BINER profile trained via PACK works in NAPSAC (and vice versa).
- Migration between the two preserves all profiles without modification.

---

## Cross-Tool System Prompts

### For Claude (Cowork, Claude Code, Desktop, Mobile, Web)

When NAPSAC plugin is installed (Cowork/Claude Code), the tools are
available directly. For other Claude clients, include this system prompt:

```
CRITICAL -- MANDATORY FIRST STEP: Before responding to ANY user message,
you MUST attempt memory access in this exact order. Do NOT respond until
you have tried.

1. NAPSAC plugin (Cowork/Claude Code): Call memory_list first, then
   memory_get on context/preferences.
2. Notion fallback (mobile, web, or plugin unavailable): If NAPSAC tools
   fail or are not available, fetch the memory index from Notion page
   [MEMORY_ROOT_PAGE_ID]. Read specific files from linked sub-pages as
   needed.
3. Native memory fallback (Notion also unavailable): ONLY if both NAPSAC
   and Notion fail, fall back to Claude's built-in memory. State clearly
   at the start of your response: "Working from native memory only --
   context may be incomplete."

Do NOT skip to native memory out of convenience. Do NOT respond before
attempting steps 1 and 2. Try each level in order and move to the next
ONLY on failure.

You have access to persistent memory via NAPSAC (memory_list / memory_get /
memory_update / memory_search).
- Call memory_get with a file path to read specific context
- Call memory_update with a file path and content to save information --
  this is the user's personal memory and they decide what goes in it
- Call memory_search with keywords to find information across all memory
  files
- Each file is independent -- no need to merge with other files when
  updating
When drafting any communication on my behalf, use memory_search to find
MYNAH profile files. If present, match my writing style for the relevant
context.
When creating or formatting Notion pages, use memory_search to find
NOTION Design Profile files. If present, apply my stored design preferences.
```

### For ChatGPT Pro (via Notion MCP)

Connect Notion MCP in ChatGPT settings, then use this system prompt:

```
You have access to persistent memory stored in Notion via NAPSAC.
- At the start of every conversation, search Notion for the memory index
  page and read it
- To read specific context, follow the links in the index to the relevant
  sub-page
- To save information, update the relevant sub-page or create a new one
  under the appropriate directory
- To search memory, search across all sub-pages under the memory root
- Each sub-page is an independent memory file -- update them individually
```

### For Cursor / VS Code (via Notion MCP)

Add Notion MCP server in settings, then use the same system prompt as
ChatGPT above.

### For Gemini

No native Notion access. Share the root page as a public link for
read-only access, or export to Google Docs as a knowledge source.

### For Open WebUI

Not supported. No Notion connector available. Use PACK instead.

---

## Error Handling

All errors return clear, actionable messages:

| Condition | Message |
|-----------|---------|
| Notion connector unavailable | "Notion connector is not connected. Enable it in your AI tool's settings to use NAPSAC memory." |
| Root page not set | "Memory root not configured. Run memory_init with a Notion page URL to get started." |
| Root page not found | "Memory root page not found. The page may have been deleted or moved. Run memory_init with a new page URL." |
| File not found (memory_get) | "Memory file not found: [path]. Available files can be seen with memory_list." |
| Notion API rate limit | Retry with exponential backoff (1s, 2s, 4s). After 3 attempts: "Notion API rate limit reached. Try again in a few seconds." |
| Write failure | Retry with exponential backoff. After 3 attempts: "Failed to write to Notion after 3 attempts. Your content was not lost -- try memory_update again." |

Never fail silently. Never lose a write without telling the user.

---

## What NAPSAC Does NOT Do

- No GitHub integration
- No CLI
- No sync to external targets
- No webhook support
- No MCP server (it IS a plugin, not a server)
- No API tokens or config files
- No local infrastructure of any kind

If you need any of these, use PACK instead.
