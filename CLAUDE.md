# Project Instructions

## Versioning

All version numbers must stay in sync across these files:
- `.claude-plugin/plugin.json` → `version`
- `.claude-plugin/marketplace.json` → `metadata.version` and `plugins[0].version`
- Every skill frontmatter → `metadata.version`

**Before publishing or committing a release**, verify all versions match. Bump them together — never bump one without the others.
