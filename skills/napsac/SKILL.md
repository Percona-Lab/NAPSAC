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

**The plugin is the authoritative implementation.** It provides validated
naming, enforced page structure, and guaranteed index regeneration after
every write. The system prompt approach (for ChatGPT, Cursor, Claude mobile)
is a best-effort fallback — it relies on AI compliance with conventions,
which varies by tool.

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
├── _conventions (naming/formatting rules — system page)
├── contacts/
│   └── contacts/key-people
├── context/
│   ├── context/general
│   └── context/preferences
├── profiles/
│   └── profiles/mynah-styles
└── projects/
    ├── projects/project-alpha
    └── projects/project-beta
```

**Conventions:**
- "Directories" are Notion pages whose title ends with `/` and contain
  child pages. They are organizational containers only.
- "Leaf pages" contain actual memory content. Their title is the full
  file path (e.g., `context/preferences`).
- The `_index` page is auto-generated on every `memory_update`.
- The `_conventions` page is created once during `memory_init` and never
  modified during normal operations. It is a system page.
- Page titles use the file path as the identifier for easy lookup.

---

## Tool Implementations

### memory_init

**Purpose:** Set up a new memory root in Notion.

**Input:**
- `page` (required) — Notion page URL or ID to use as the memory root

**Behavior:**
1. Validate that the Notion connector is available. If not, return:
   `"Notion connector is not connected. Enable it in Settings → Connectors → Notion to use NAPSAC memory."`
2. Fetch the provided page to confirm it exists and is accessible.
3. Create three child pages under the root:
   - `_index` — Empty index page (will be populated on first update)
   - `_conventions` — Naming and formatting rules (see _conventions content below)
   - `context/general` — Starter memory file with content:
     ```
     ## General Context

     This is the default memory file. Save general context and preferences here.

     *Created by NAPSAC on [date].*
     ```
4. Create the `context/` directory page as a container.
5. Store the root page ID internally for subsequent calls.
6. Return confirmation with the root page URL.

**_conventions page content:**

```
# NAPSAC Memory Conventions

This page defines how memory is structured, named, and formatted.
Every AI tool reading this memory must follow these rules.

## Page Naming
- Leaf pages use full paths as titles: context/preferences, projects/alpha
- Directory pages use trailing slash: context/, projects/
- No file extensions (Notion pages don't have them)
- _index and _conventions are system pages -- never delete or rename

## Directory Taxonomy
- context/ -- Stable background: role, preferences, tooling
- projects/ -- Active and archived project memory
- profiles/ -- Style/format profiles (MYNAH, BINER)
- contacts/ -- People, accounts, org references
- New directories can be created freely

## Content Format
- Write as native Notion blocks, not raw markdown
- Headings, bullets, code blocks, quotes use native block types
- Pages must be readable in Notion UI without cleanup

## Index (_index)
- Regenerated after every memory write
- Lists all files with: path, description, tags, last updated
- Target: ~700 tokens for ~25 files
- If _index drifts, the next write auto-corrects it

## Profiles
- MYNAH: profiles/mynah-{context-id} (schema matches PACK)
- BINER: profiles/notion-design (schema matches PACK)
- Profiles are portable between PACK and NAPSAC
```

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
   b. **Path resolution:** If no exact match, attempt fuzzy resolution:
      - If `path` has no `/`, search for `context/{path}`, `projects/{path}`,
        etc. If exactly one match is found, use it. If multiple matches,
        return an error listing the ambiguous options.
   c. Fetch the page content.
   d. Convert Notion blocks to clean readable text.
   e. Return the content.
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
1. **Validate the path:**
   a. Must contain a `/` (e.g., `context/preferences`, not just `preferences`).
      If no `/`, check if the user means `context/{path}`. If ambiguous, ask.
   b. Must not start with `_` (reserved for system pages).
   c. Must not contain file extensions (`.md`, `.txt`, etc.).
   d. Must use lowercase with hyphens for word separation.
2. Search for an existing page under the root whose title matches `path`.
3. **If page exists:**
   a. Replace the page content with the new content.
   b. Convert markdown to proper Notion blocks (headings, bullets, code
      blocks, etc.) — not raw markdown dumped into the page.
4. **If page does not exist:**
   a. Determine the directory from the path (e.g., `context/` from
      `context/preferences`).
   b. Find or create the directory page under the root.
   c. Create a new child page with the path as its title.
   d. Write the content as Notion blocks.
5. **After write, ALWAYS regenerate the `_index` page:**
   a. List all child pages under the root (excluding `_index`, `_conventions`,
      and directory pages).
   b. For each page, extract: path (title), description (first line),
      tags (if present), last edited timestamp, page link.
   c. Group by directory and format as a table.
   d. Replace the `_index` page content.
   e. This step is MANDATORY. It must not be skipped, deferred, or made
      conditional. Every successful write triggers an index regeneration.
6. Return confirmation with the page ID and path.

**Output:**
```
path:          string — the file path written
page_id:       string — Notion page ID
index_updated: boolean — confirms index was regenerated (always true)
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
  After 3 attempts, return error: `"Failed to write to Notion after 3
  attempts. Your content was not lost — try memory_update again."`
  Never lose a write silently.
- If `path` is omitted: route to `context/general` with a deprecation
  warning (v1 compat, matches PACK behavior).
- If `_index` regeneration fails after a successful write: log a warning
  but do NOT fail the write. The index will self-correct on next write.

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

## projects/

| File | Description | Tags | Updated |
|------|-------------|------|---------|
| projects/project-alpha | Alpha project tracking | alpha, q2-2026 | 2026-03-14 |

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

### For Cowork and Claude Code (Plugin)

When the NAPSAC plugin is installed, the tools are available directly.
No system prompt is needed for memory operations — the plugin handles
validation, naming enforcement, and index regeneration automatically.

Add this to project instructions for MYNAH/BINER integration:

```
When drafting any communication on my behalf, use memory_search to find
MYNAH profile files. If present, match my writing style for the relevant
context.
When creating or formatting Notion pages, use memory_search to find
NOTION Design Profile files. If present, apply my stored design preferences.
```

### For Claude (Mobile, Desktop, Web) — System Prompt Fallback

When the plugin is not available, include this system prompt:

```
CRITICAL -- MANDATORY FIRST STEP: Before responding to ANY user message,
you MUST attempt memory access. Do NOT respond until you have tried.

1. Fetch the memory index from Notion page [MEMORY_ROOT_PAGE_URL]. Look
   for the _index sub-page and read it.
2. Read the _conventions sub-page. Follow its rules for naming, formatting,
   and structuring all memory pages.
3. Read context/preferences from the linked sub-pages.
4. If Notion is unavailable, fall back to Claude's built-in memory. State
   clearly: "Working from native memory only -- context may be incomplete."

Do NOT skip memory loading out of convenience. Try Notion first, always.

You have access to persistent memory stored as Notion sub-pages under the
memory root.
- At session start, read the _index page to see all available memory files
- ALWAYS read _conventions before writing any memory -- it defines naming,
  structure, and format rules
- To read specific context, fetch the relevant sub-page by following links
  in the index
- To save information, update the relevant sub-page or create a new one
  under the appropriate directory -- this is the user's personal memory and
  they decide what goes in it
- After every write, regenerate the _index page to reflect the current state
- To search memory, search across all sub-pages under the memory root
- Each sub-page is an independent memory file -- update them individually,
  no need to merge

When drafting any communication on my behalf, search memory for MYNAH
profile files (under profiles/). If present, match my writing style for
the relevant context (Slack DM, email, channel post, etc.).

When creating or formatting Notion pages, search memory for BINER/Notion
Design Profile files (under profiles/). If present, apply my stored design
preferences instead of default formatting.
```

### For ChatGPT Pro (via Notion MCP)

Connect Notion MCP in ChatGPT settings, then use the same system prompt
as Claude above. These tools access Notion via Notion MCP instead of a
native connector, but the memory structure and behavior are identical.

### For Cursor / VS Code (via Notion MCP)

Add Notion MCP server in settings, then use the same system prompt as
Claude above.

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
| Notion connector unavailable | "Notion connector is not connected. Enable it in Settings → Connectors → Notion to use NAPSAC memory." |
| Root page not set | "Memory root not configured. Run memory_init with a Notion page URL to get started." |
| Root page not found | "Memory root page not found. The page may have been deleted or moved. Run memory_init with a new page URL." |
| File not found (memory_get) | "Memory file not found: [path]. Available files can be seen with memory_list." |
| Invalid path (memory_update) | "Invalid path: [path]. Paths must include a directory (e.g., context/preferences, not just preferences)." |
| Path has extension | "Invalid path: [path]. Notion pages don't use file extensions. Use [path without extension] instead." |
| Notion API rate limit | Retry with exponential backoff (1s, 2s, 4s). After 3 attempts: "Notion API rate limit reached. Try again in a few seconds." |
| Write failure | Retry with exponential backoff. After 3 attempts: "Failed to write to Notion after 3 attempts. Your content was not lost — try memory_update again." |
| Index regen failure | Log warning. Do not fail the write. Index self-corrects on next write. |

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
