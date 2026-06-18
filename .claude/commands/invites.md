---
description: Read LinkedIn invitation manager page and extract all pending invitation names
allowed-tools: mcp__claude-in-chrome__tabs_context_mcp, mcp__claude-in-chrome__tabs_create_mcp, mcp__claude-in-chrome__navigate, mcp__claude-in-chrome__javascript_tool, mcp__claude-in-chrome__read_page
---

Extract all pending LinkedIn connection invitations from the invitation manager page.

## Steps

1. **Get browser context** using `tabs_context_mcp` to find available tabs.

2. **Navigate to the LinkedIn invitation manager** at `https://www.linkedin.com/mynetwork/invitation-manager/received/` (create a new tab if needed).

3. **Wait for page load**, then use `javascript_tool` to extract invitation data. Use this script to get the currently loaded invitations:

```javascript
const cards = document.querySelectorAll('[role="listitem"]');
const invitations = Array.from(cards).map(card => {
  const ignoreBtn = card.querySelector('button[aria-label*="Ignore"]');
  if (!ignoreBtn) return null;
  const name = ignoreBtn.getAttribute('aria-label').replace('Ignore an invitation to connect from ', '');
  const profileLink = Array.from(card.querySelectorAll('a[href*="/in/"]')).find(a => a.textContent.trim().length > 0);
  const profileUrl = profileLink?.href || null;
  return { name, profileUrl };
}).filter(Boolean);
const totalMatch = document.querySelector('span')?.textContent?.match(/All \((\d+)\)/);
const total = totalMatch ? parseInt(totalMatch[1]) : null;
JSON.stringify({ total, loaded: invitations.length, invitations });
```

4. **Load all invitations**: The page lazy-loads invitations (typically 10 at a time). If the total count exceeds the loaded count, repeatedly click the "Load more" button and re-extract until all invitations are loaded. To click "Load more":

```javascript
const loadMore = Array.from(document.querySelectorAll('button')).find(b => b.textContent.trim().includes('Load more'));
if (loadMore) { loadMore.click(); 'clicked'; } else { 'no load more button'; }
```

Wait ~2 seconds between clicks for new cards to render, then re-run the extraction script. Repeat until all invitations are loaded or the "Load more" button disappears.

5. **Report results** to the user as a numbered list with name and profile URL for each invitation. Include the total count at the top.

## Key page structure details

- Each invitation card is a `div[role="listitem"]`
- The **Ignore** button `aria-label` contains the inviter's full name in the format: `"Ignore an invitation to connect from {Name}"`
- Profile links match `a[href*="/in/"]` - the one with text content holds the display name
- The total invitation count appears in a span like `"All (95)"`
- A "Load more" button at the bottom loads the next batch of ~10 invitations
