# NAPSAC — Plugin Instructions

NAPSAC (Notion Agent Persistent Store for Accessible Context) is a Claude plugin that provides persistent AI memory using Notion as the only storage backend.

## Plugin installation

To install as a Claude plugin:

```
Install this plugin: https://github.com/Percona-Lab/NAPSAC
```

Claude will clone the repo, copy `.claude-plugin/` and `skills/` to `~/.claude/plugins/napsac/`, and clean up.

## How it works

NAPSAC provides five tools that use Claude's native Notion connector for all reads and writes:

- `memory_init` — Set up a memory root in Notion (run once)
- `memory_list` — List all memory files (call at session start)
- `memory_get` — Read a specific memory file
- `memory_update` — Write or create a memory file
- `memory_search` — Search across all memory files

All tool logic is defined in `skills/napsac/SKILL.md`. There is no server, no compiled code, no runtime. The plugin is a behavior specification that Claude follows using existing Notion MCP tools.

## Required connectors

- **Notion** — Must be connected via Settings > Connectors > Notion. All memory operations go through the Notion API.

## File structure

```
.claude-plugin/plugin.json    — Plugin metadata
skills/napsac/SKILL.md        — Tool definitions and behavior spec
skills/napsac/references/     — Supporting reference docs
CLAUDE.md                     — This file
README.md                     — User-facing documentation
DESIGN.md                     — Architecture decisions
CONTRACTS.md                  — Non-negotiable invariants
```

## Related plugins

NAPSAC works alongside these Alpine toolkit plugins:

- **MYNAH** — Writing style profiles stored at `profiles/mynah-*`
- **BINER** — Notion design profiles stored at `profiles/notion-design`

When these plugins are installed alongside NAPSAC, they train profiles and store them in Notion memory automatically.

## Development

This repo is the plugin source. To develop:

1. Edit files in this repo
2. Copy updated files to `~/.claude/plugins/napsac/` to test
3. Commit and push to publish updates
