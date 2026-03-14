# Cross-Tool Setup Reference

## How Each AI Tool Connects to NAPSAC Memory

Because memory lives in Notion, any AI tool with Notion access can use it.

### Claude (Mobile, Desktop, Web)

**Connection:** Native Notion connector (built into Claude)
**Access:** Full read/write
**Setup:**
1. Connect Notion in Claude settings (OAuth — no tokens needed)
2. Share the memory root page with Claude's integration
3. Add the NAPSAC system prompt to your project instructions

### Claude Code / Cowork (Recommended)

**Connection:** NAPSAC plugin + native Notion connector
**Access:** Full read/write via plugin tools with validation
**Setup:**
1. Install the NAPSAC plugin: `Install this plugin: https://github.com/Percona-Lab/NAPSAC`
2. Connect Notion in settings
3. Run `memory_init` with your Notion page URL
4. The plugin handles validation, naming enforcement, and index regeneration automatically

**Why this is the recommended path:** The plugin validates path naming, enforces page structure, and guarantees index regeneration after every write. The system prompt approach relies on AI compliance, which varies by tool.

### ChatGPT Pro

**Connection:** Notion MCP
**Access:** Full read/write
**Setup:**
1. In ChatGPT settings, connect Notion MCP
2. Authorize access to your memory root page
3. Add the NAPSAC system prompt for Notion MCP users

### Cursor

**Connection:** Notion MCP
**Access:** Full read/write
**Setup:**
1. Add Notion MCP server in Cursor settings
2. Authorize access to your memory root page
3. Add the NAPSAC system prompt to your rules

### VS Code

**Connection:** Notion MCP (via MCP extension)
**Access:** Full read/write
**Setup:**
1. Install an MCP-compatible extension
2. Configure Notion MCP server
3. Authorize access to your memory root page
4. Add the NAPSAC system prompt

### Gemini

**Connection:** No native Notion access
**Access:** Read-only
**Setup:**
1. Share the memory root page as a public link
2. Or export memory to Google Docs and attach as a knowledge source
3. Memory updates require a tool with Notion write access

### Open WebUI

**Connection:** None
**Access:** Not supported
**Alternative:** Use PACK instead (MCP server with GitHub backend)

## Compatibility Matrix

| Tool | Connection | Read | Write | Plugin | Setup Effort |
|------|-----------|------|-------|--------|-------------|
| Claude (all) | Native Notion | Yes | Yes | Optional | Minimal |
| Claude Code | Plugin + Notion | Yes | Yes | Yes | Minimal |
| Cowork | Plugin + Notion | Yes | Yes | Yes | Minimal |
| ChatGPT Pro | Notion MCP | Yes | Yes | No | Low |
| Cursor | Notion MCP | Yes | Yes | No | Low |
| VS Code | Notion MCP | Yes | Yes | No | Low |
| Gemini | Public link / Docs | Yes | No | No | Medium |
| Open WebUI | None | No | No | No | N/A |
