# NAPSAC Page Structure Reference

## How Notion Pages Map to Memory Files

NAPSAC uses Notion's native page hierarchy to represent a file-directory structure:

```
Root Page (user-designated)
│
├── _index                          ← Auto-managed index
│
├── context/                        ← Directory page (title ends with /)
│   ├── context/general             ← Leaf page (title = full path)
│   ├── context/preferences
│   └── context/projects-overview
│
├── projects/
│   ├── projects/project-alpha
│   └── projects/project-beta
│
├── profiles/
│   └── profiles/mynah-styles
│
└── contacts/
    └── contacts/key-people
```

## Page Title Conventions

- **Directory pages**: Title is `dirname/` (e.g., `context/`, `projects/`)
- **Leaf pages**: Title is the full path without extension (e.g., `context/preferences`)
- **Index page**: Title is `_index` (underscore prefix keeps it sorted first)
- **Root page**: User's choice — not renamed by NAPSAC

## Why Full Paths in Titles

Using full paths as page titles (instead of just the filename) ensures:
1. Unique identification — no ambiguity between `context/general` and `projects/general`
2. Easy search — searching for "context/preferences" finds exactly one page
3. Portability — paths match PACK's file paths exactly
4. Human readable — users browsing Notion see the full context

## Directory Taxonomy

| Directory    | Purpose                                    | Example files                     |
|--------------|--------------------------------------------|-----------------------------------|
| `context/`   | Stable background: role, prefs, tooling    | `general`, `preferences`          |
| `projects/`  | Active and archived project memory         | `project-alpha`, `project-beta`   |
| `profiles/`  | Style/format profiles (MYNAH, BINER, etc)  | `mynah-styles`, `notion-design`   |
| `contacts/`  | People, accounts, org references           | `key-people`                      |

New directories can be added freely. The index discovers structure from page hierarchy, not from hardcoded paths.

## Content Format

Memory content is written as structured Notion blocks:

- `## Heading` → heading_2
- `### Heading` → heading_3
- `- item` → bulleted_list_item
- `1. item` → numbered_list_item
- `` `code` `` → code block
- `> quote` → quote block
- Plain text → paragraph

Content is NOT raw markdown dumped into a page. It uses native Notion block types for proper rendering and readability.
