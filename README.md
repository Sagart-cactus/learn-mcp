# Learn MCP

This repository contains a practical knowledgebase for the Model Context Protocol.

Current target: **MCP specification revision `2026-07-28`** (the stateless protocol core).

- `MCP-Complete-Knowledgebase.md` is the long-form Markdown reference.
- `html/index.html` is the published GitHub Pages version.
- `.github/workflows/pages.yml` publishes the contents of `html/` to GitHub Pages after pushes to `main`.

The article focuses on clear explanations, careful source wording, and practical MCP server design guidance.

## Keeping it current

Both files track the official specification. When a new revision lands, update:

1. The "last updated" line and the stated current revision.
2. The protocol-revisions table in section 1.
3. Any spec URLs, which are versioned by revision date (`/specification/<revision>/...`).
4. The migration checklist, so it covers the newest hop.
