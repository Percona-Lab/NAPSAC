# PACK Compatibility Reference

## Tool Signature Mapping

NAPSAC's tools are designed to be drop-in compatible with PACK's tool signatures. A user switching between the two should not need to change their system prompt (other than the CRITICAL FIRST STEP block).

### memory_list

| | PACK | NAPSAC |
|---|------|--------|
| Input | `{ tag?, topic?, dir? }` | `{ tag?, topic?, dir? }` |
| Output | `{ content, mode }` | `{ content }` |
| Source | Reads `index.md` from GitHub | Reads `_index` page from Notion |

### memory_get

| | PACK | NAPSAC |
|---|------|--------|
| Input | `{ path? }` | `{ path?, all? }` |
| Output | `{ content, sha, updated_at }` | `{ content, updated_at, page_id }` |
| Source | Reads file from GitHub | Reads page from Notion |
| No path | Returns concatenated view | Returns concatenated view |

**Difference:** NAPSAC returns `page_id` instead of `sha`. There is no SHA-based concurrency because Notion handles versioning natively via page history.

### memory_update

| | PACK | NAPSAC |
|---|------|--------|
| Input | `{ path, content, sha?, message? }` | `{ path, content, message? }` |
| Output | `{ sha, commit_url, index_updated, path }` | `{ page_id, index_updated, path }` |
| Storage | Writes file to GitHub with git commit | Writes page to Notion |
| No path | Routes to `context/general.md` | Routes to `context/general` |

**Difference:** No `sha` parameter. Notion's last-write-wins model replaces PACK's optimistic concurrency. Page history provides the audit trail instead of git commits.

### memory_search

| | PACK | NAPSAC |
|---|------|--------|
| Input | `{ query }` | `{ query }` |
| Output | `{ results: [{ path, topic, tags, snippet }] }` | `{ results: [{ path, topic, tags, snippet }] }` |
| Method | Server-side substring match | Notion search API + fallback to local match |

## File Path Conventions

Both products use identical file path conventions:

- `context/general` — general context
- `context/preferences` — user preferences
- `projects/{name}` — project-specific memory
- `profiles/mynah-{context}` — MYNAH style profiles
- `profiles/notion-design` — BINER design profiles
- `contacts/{name}` — contact/people files

PACK uses `.md` extensions; NAPSAC omits them (Notion pages don't have extensions). When migrating, strip or add `.md` as needed.

## Migration Paths

### NAPSAC → PACK
1. Export each Notion sub-page to markdown
2. Add YAML frontmatter (topic, tags, created, updated, sync)
3. Commit to a GitHub repo
4. Point PACK at the repo

### PACK → NAPSAC
1. If PACK has Notion multi-page sync enabled, the page structure already exists
2. Point NAPSAC at the sync parent page
3. Run `memory_init` to adopt it
4. Switch system prompts

### Running Both
- PACK owns writes via GitHub
- Notion pages are read-only mirrors (via PACK's Notion sync)
- Don't write to Notion via NAPSAC if PACK sync is active — PACK will overwrite on next sync
