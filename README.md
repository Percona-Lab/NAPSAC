# NAPSAC — Notion Agent Persistent Store for Accessible Context

Zero-config persistent AI memory backed by Notion. Works across Claude, ChatGPT, Cursor, and VS Code.

Leave the PACK at base camp. Grab the NAPSAC.

> 100% vibe coded with [Claude](https://claude.ai/).

## What it does

NAPSAC gives any AI tool persistent memory using Notion as the only storage backend. No GitHub repo, no MCP server, no API tokens, no config files. Connect Notion, point to a page, and start remembering.

Five tools, matching [PACK](https://github.com/Percona-Lab/PACK)'s signatures:

- **`memory_init`** — Takes a Notion page URL. Sets it as the memory root. Creates an index and starter content.
- **`memory_list`** — Lists all memory files with paths, descriptions, and tags. Call at session start.
- **`memory_get`** — Reads a specific memory file by path. Returns clean, structured content.
- **`memory_update`** — Writes or creates a memory file. Auto-updates the index. Notion page history serves as version tracking.
- **`memory_search`** — Searches across all memory files by keywords. Returns matching excerpts with file paths.

Part of the [Alpine toolkit](https://github.com/Percona-Lab) family alongside [PACK](https://github.com/Percona-Lab/PACK), [MYNAH](https://github.com/Percona-Lab/MYNAH), [BINER](https://github.com/Percona-Lab/BINER), and [IBEX](https://github.com/Percona-Lab/IBEX).

## How it works

Memory is organized as Notion sub-pages under a root page you choose:

```
[Memory Root Page]
├── _index (auto-managed)
├── context/
│   ├── general
│   └── preferences
├── projects/
│   ├── project-alpha
│   └── project-beta
├── profiles/
│   └── mynah-styles
└── contacts/
    └── key-people
```

- **Notion is the only backend.** All reads and writes go through Notion's API via your AI tool's native connector or Notion MCP.
- **Zero config.** No tokens, no env files, no local server. If your AI tool can access Notion, NAPSAC works.
- **OAuth-only auth.** Notion's standard OAuth flow handles permissions. You authorize once and it works everywhere.
- **Version history for free.** Notion tracks page history natively. Every edit is recoverable.

## Quick start

### In Claude Code or Cowork (plugin)

1. Install the NAPSAC plugin
2. Connect Notion in your settings
3. Tell Claude: "Initialize my NAPSAC memory at [paste Notion page URL]"
4. Done. Claude will use memory automatically from now on.

### In Claude Desktop, Mobile, or Web (system prompt)

1. Connect Notion in Claude settings
2. Create a Notion page to use as your memory root
3. Add the system prompt below to your project instructions
4. Tell Claude: "Set up my memory on this Notion page: [URL]"

### In ChatGPT, Cursor, or VS Code (Notion MCP)

1. Connect Notion MCP in your tool's settings
2. Create a Notion page to use as your memory root
3. Add the system prompt for Notion MCP users (see below)
4. Start using memory commands

## Tool reference

### memory_init

Sets up a new memory root. Run once.

```
Input:  { page: "https://notion.so/your-page-id" }
Output: { root_page_id, status: "initialized" }
```

Creates `_index` and `context/general` under your chosen page.

### memory_list

Lists all memory files. Call at session start.

```
Input:  { tag?, topic?, dir? }
Output: { content: "index listing ~700 tokens" }
```

### memory_get

Reads a memory file.

```
Input:  { path: "context/preferences" }
Output: { content, updated_at, page_id }
```

Omit `path` to get all files concatenated (v1 compat).

### memory_update

Writes a memory file. Creates the file and directory if they don't exist. Auto-regenerates the index.

```
Input:  { path: "projects/alpha", content: "## Project Alpha\n\n- Status: active\n- Lead: Sarah" }
Output: { path, page_id, index_updated: true }
```

Content is converted to proper Notion blocks (headings, bullets, code blocks) — not raw markdown.

### memory_search

Searches across all memory files.

```
Input:  { query: "project alpha status" }
Output: { results: [{ path, topic, tags, snippet }] }
```

## System prompts

### For Claude (all clients)

```
CRITICAL -- MANDATORY FIRST STEP: Before responding to ANY user message,
you MUST attempt memory access in this exact order. Do NOT respond until
you have tried.

1. NAPSAC plugin (Cowork/Claude Code): Call memory_list first, then
   memory_get on context/preferences.
2. Notion fallback (mobile, web, or plugin unavailable): If NAPSAC tools
   fail or are not available, fetch the memory index from Notion page
   [MEMORY_ROOT_PAGE_ID]. Read specific files from linked sub-pages as needed.
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
- Call memory_search with keywords to find information across all memory files
- Each file is independent -- no need to merge with other files when updating
When drafting any communication on my behalf, use memory_search to find
MYNAH profile files. If present, match my writing style for the relevant context.
When creating or formatting Notion pages, use memory_search to find NOTION
Design Profile files. If present, apply my stored design preferences.
```

Replace `[MEMORY_ROOT_PAGE_ID]` with your actual Notion page URL.

### For ChatGPT, Cursor, VS Code (Notion MCP)

```
You have access to persistent memory stored in Notion via NAPSAC.
- At the start of every conversation, search Notion for the memory index
  page and read it
- To read specific context, follow the links in the index to the relevant
  sub-page
- To save information, update the relevant sub-page or create a new one
  under the appropriate directory
- To search memory, search across all sub-pages under the memory root
- Each sub-page is an independent memory file — update them individually
```

## Cross-tool compatibility

| Tool | Connection | Read | Write | Setup |
|------|-----------|------|-------|-------|
| Claude (all clients) | Native Notion | Yes | Yes | Minimal |
| Claude Code / Cowork | Plugin + Notion | Yes | Yes | Minimal |
| ChatGPT Pro | Notion MCP | Yes | Yes | Low |
| Cursor | Notion MCP | Yes | Yes | Low |
| VS Code | Notion MCP | Yes | Yes | Low |
| Gemini | Public link / Docs export | Yes | No | Medium |
| Open WebUI | Not supported | No | No | Use PACK |

## MYNAH and BINER integration

NAPSAC stores profiles for two companion plugins from the Alpine toolkit:

**[MYNAH](https://github.com/Percona-Lab/MYNAH)** (My Natural Authoring Helper) learns your writing style. Profiles are stored at `profiles/mynah-{context}`. Train once in Cowork or Claude Code, and every AI tool reading your NAPSAC memory can match your voice.

**[BINER](https://github.com/Percona-Lab/BINER)** (Beautiful Intelligent Notion Enhancement & Reformatting) learns your Notion design preferences. Profiles are stored at `profiles/notion-design`. Any AI tool creating Notion pages can apply your stored design preferences.

**In Cowork and Claude Code:** MYNAH and BINER plugins work alongside NAPSAC natively — they train profiles and store them in Notion memory.

**In other clients:** Profiles are read-only. Training requires Cowork or Claude Code where plugins are supported.

## PACK vs NAPSAC

### When to use NAPSAC (Notion-native memory)

- You want persistent AI memory with zero setup
- You use Claude and/or ChatGPT as your primary AI tools
- You want to read and write memory from any device, including mobile
- You want to share memory selectively with teammates
- You don't need git history, CLI access, or commit-level auditability
- You're not comfortable with terminal setup or managing API tokens
- You want one memory that works across Claude, ChatGPT, Cursor, and VS Code via Notion MCP

### When to use PACK (GitHub-backed memory)

- You want full git history with commit messages, diffs, and SHAs
- You need CLI access to memory outside of AI sessions
- You use Open WebUI, Windsurf, or other MCP clients without Notion connectors
- You want token-efficient directory-based storage that stays flat as memory grows
- You want complete control over storage, backup, and encryption
- You're comfortable with terminal setup and managing a local MCP server
- You need webhook-based sync to fan out to multiple targets

### When to use both

- PACK as source of truth with git history and CLI
- PACK's multi-page Notion sync enabled (`NOTION_SYNC_MODE=multi`) for mobile reads
- NAPSAC's system prompt as the fallback for mobile and non-MCP clients
- Both use the same file path conventions and page structure, so content is portable

### Migration paths

**From NAPSAC to PACK:** Export Notion pages to markdown files, add YAML frontmatter, commit to a GitHub repo. A script for this is on the roadmap.

**From PACK to NAPSAC:** PACK's Notion sync already creates the page structure. Just switch your system prompt to point at the Notion page instead of the MCP server.

**Running both:** PACK owns writes via GitHub. Notion pages are read-only mirrors. Don't write to Notion directly via NAPSAC if PACK sync is active — PACK will overwrite on next sync.

## Project structure

```
├── .claude-plugin/
│   └── plugin.json           # Plugin metadata
└── skills/
    └── napsac/
        ├── SKILL.md           # Main skill spec (tool definitions, behavior)
        └── references/
            ├── page-structure.md     # Notion page layout conventions
            ├── pack-compatibility.md # PACK tool mapping and migration
            └── cross-tool-setup.md   # Per-tool connection guides
```

## License

MIT
