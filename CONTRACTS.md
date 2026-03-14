# NAPSAC Contracts

Non-negotiable invariants for the NAPSAC project. These are the rules that every change must satisfy. Violating a contract is a bug, regardless of intent.

---

## C1: No Silent Data Loss

NAPSAC never deletes, overwrites, or loses user memory without explicit user action.

- `memory_update` writes to a specific page. It never modifies pages other than the target and `_index`.
- If a Notion API write fails after retries, the error message explicitly states that content was not saved. Never fail silently.
- `memory_init` never overwrites existing content in the root page. It only creates new child pages.

## C2: PACK Prompt Compatibility

System prompts written for PACK must work with NAPSAC (and vice versa), with only the CRITICAL FIRST STEP block differing.

- Tool names are identical: `memory_list`, `memory_get`, `memory_update`, `memory_search`.
- Input parameters are a superset of PACK's. NAPSAC accepts everything PACK accepts.
- Output fields use the same names where applicable (`content`, `path`, `updated_at`, `index_updated`).
- `memory_get` with no `path` returns a concatenated view of all files, matching PACK's v1 compat behavior.
- `memory_update` with no `path` routes to `context/general` with a deprecation warning, matching PACK's v1 compat behavior.

## C3: No Infrastructure Required

NAPSAC requires zero local infrastructure to function.

- No local server process. No MCP server. No background daemon.
- No config files on the user's machine. No `~/.napsac.env`.
- No API tokens for the user to manage. Notion OAuth handles auth.
- No npm install, no pip install, no binary to download.
- The plugin is a SKILL.md file that Claude follows. That's it.

## C4: Notion is the Only Backend

All reads and writes go through Notion. No other storage system.

- No GitHub. No local filesystem. No SQLite. No Redis.
- If Notion is unavailable, NAPSAC is unavailable. That's expected.
- The fallback path is Claude's native memory (clearly announced), not a second storage backend.

## C5: Index Consistency

The `_index` page is regenerated on every `memory_update`. This is mandatory, not optional.

- Every successful `memory_update` triggers index regeneration. No exceptions.
- If `_index` drifts from actual page state (e.g., manual edits in Notion), the next `memory_update` auto-corrects it.
- `_index` lists every memory page under the root with path, description, and timestamp.
- `_index` is lightweight enough to load at session start (~700 tokens for ~25 files).
- If index regeneration fails after a successful write, the write still succeeds. Index self-corrects on next write.

## C6: The User Owns Their Memory

NAPSAC never refuses to store information the user explicitly asks to save.

- The user decides what goes in their memory — personal facts, work context, preferences, style profiles, contacts, or anything else.
- No gatekeeping, no filtering, no "this doesn't seem appropriate" responses.
- Memory content is private by default (Notion page permissions).

## C7: Clean Content Format

Memory content is written as structured Notion blocks, not raw markdown dumped into a page.

- Headings, bullets, code blocks, and quotes use their native Notion block types.
- Pages are readable and editable in Notion's UI without looking broken.
- `memory_get` returns clean text, not raw Notion block JSON.

## C8: Clear Error Messages

Every error returns an actionable message. No stack traces, no cryptic codes.

- Notion connector unavailable → "Enable it in Settings → Connectors → Notion"
- Root page not set → "Run memory_init with a Notion page URL"
- File not found → Show the path that was requested
- Invalid path → Explain what's wrong and suggest the correct format
- Write failure → "Your content was not lost — try memory_update again"
- Never fail silently. Never return an empty response on error.

## C9: MYNAH and BINER Profile Compatibility

Profile storage format matches PACK exactly.

- MYNAH profiles stored at `profiles/mynah-{context-id}` use the same schema as PACK.
- BINER profiles stored at `profiles/notion-design` use the same schema as PACK.
- A profile trained via PACK works in NAPSAC without modification.
- A profile trained via NAPSAC works in PACK without modification.

## C10: Page Naming Enforcement

Page titles always use full file paths. The plugin validates this on every write.

- Leaf pages must include a directory prefix (`context/preferences`, not `preferences`).
- No file extensions (`.md`, `.txt`). Notion pages don't have extensions.
- No `_` prefix for user pages (reserved for `_index` and `_conventions`).
- Lowercase with hyphens for word separation.
- The plugin rejects writes that violate these rules with a clear error message.

## C11: _conventions is Never Modified During Normal Operations

The `_conventions` page is created once during `memory_init` and never touched again.

- `memory_update` must never target `_conventions`.
- The plugin must reject any attempt to write to `_conventions` with a clear error.
- Users can edit `_conventions` manually in Notion if they want to customize rules.
- The plugin does not read `_conventions` at runtime — it validates directly. `_conventions` exists for non-plugin environments and human reference.
