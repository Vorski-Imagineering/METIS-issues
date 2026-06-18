# Claude Code Instructions

## Browser automation

Always use the Chrome connector (`mcp__claude-in-chrome__*` tools) when working with:
- `linkedin.com` — any LinkedIn page (search, connections, messaging, profiles, invitations)
- `docs.google.com` — any Google Docs/Sheets/Drive page

Never attempt to fetch or scrape these pages with `WebFetch` or `WebSearch`. Always navigate to them in the live browser via `mcp__claude-in-chrome__navigate` and interact using `mcp__claude-in-chrome__javascript_tool`, `mcp__claude-in-chrome__read_page`, etc.
