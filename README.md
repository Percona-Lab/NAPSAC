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

### 1. Set up Notion

1. Create a **private** Notion page to use as your memory root. This is where all your memory files will live as sub-pages.
2. Connect your AI tool to Notion:
   - **Claude (all clients):** Settings > Connectors > Notion
   - **ChatGPT, Cursor, VS Code:** Connect Notion MCP in your tool's settings

> **Privacy notice**: Your memory will accumulate sensitive context over time. Keep the root page private. Only share specific sub-pages intentionally.

### 2. Connect your AI tool

**Claude Code or Cowork (plugin):**

1. Open Claude Code and prompt: `Install this plugin: https://github.com/Percona-Lab/NAPSAC`
2. Tell Claude: "Initialize my NAPSAC memory at [paste Notion page URL]"

**Claude Desktop, Mobile, or Web:**

1. Tell Claude: "Set up my memory on this Notion page: [URL]"

**ChatGPT, Cursor, or VS Code:**

1. Start using memory commands

### 3. Add the system prompt

Add the [system prompt](#system-prompt) below to your project instructions (Claude) or system prompt settings (ChatGPT, Cursor, VS Code). Replace `[MEMORY_ROOT_PAGE_URL]` with your Notion page URL.

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

## System prompt

Add this to your project instructions in any Claude client, or to your system prompt in ChatGPT/Cursor/VS Code. Replace `[MEMORY_ROOT_PAGE_URL]` with your actual Notion page URL.

```
CRITICAL -- MANDATORY FIRST STEP: Before responding to ANY user message,
you MUST attempt memory access. Do NOT respond until you have tried.

1. Fetch the memory index from Notion page [MEMORY_ROOT_PAGE_URL]. Look
   for the _index sub-page and read it.
2. Read context/preferences from the linked sub-pages.
3. If Notion is unavailable, fall back to Claude's built-in memory. State
   clearly: "Working from native memory only -- context may be incomplete."

Do NOT skip memory loading out of convenience. Try Notion first, always.

You have access to persistent memory stored as Notion sub-pages under the
memory root.
- At session start, read the _index page to see all available memory files
- To read specific context, fetch the relevant sub-page by following links
  in the index
- To save information, update the relevant sub-page or create a new one
  under the appropriate directory -- this is the user's personal memory and
  they decide what goes in it
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

This single prompt works across all clients — Claude (desktop, mobile, web), ChatGPT Pro, Cursor, and VS Code. The tools access Notion differently (native connector vs Notion MCP) but the memory structure and behavior are identical.

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
