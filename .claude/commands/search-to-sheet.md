---
description: Extract LinkedIn people search results and save them to a CSV file
allowed-tools: Read, Write, mcp__claude-in-chrome__tabs_context_mcp, mcp__claude-in-chrome__tabs_create_mcp, mcp__claude-in-chrome__navigate, mcp__claude-in-chrome__javascript_tool, mcp__claude-in-chrome__read_page
---

Extract the first page of LinkedIn people search results and save them to a CSV file.

**Arguments:** `$ARGUMENTS` — a LinkedIn people search URL. If missing, stop and tell the user:
`"Please provide a LinkedIn search URL: /search-to-sheet <linkedin-search-url>"`

> **Debug mode is ON.** If any step fails or returns an unexpected result, stop immediately and report the exact error to the user. Do NOT try fallbacks, alternative approaches, or workarounds.

## Steps

### 1. Parse arguments

Extract the LinkedIn search URL from `$ARGUMENTS`. If missing, stop and tell the user.

### 2. Get browser context

Use `tabs_context_mcp` to find available tabs. Use an existing tab or create a new one with `tabs_create_mcp`.

### 3. Navigate to the LinkedIn search URL

Navigate to the LinkedIn search URL and wait for the page to load.

### 4. Extract search result cards

LinkedIn uses obfuscated CSS class names, so find cards by locating action buttons (Connect / Follow / Message / Pending) and walking up the DOM. Use `javascript_tool` to poll until cards appear (up to 10 seconds), then extract all visible people:

```javascript
new Promise((resolve, reject) => {
  let elapsed = 0;
  const check = () => {
    const actionBtns = Array.from(document.querySelectorAll('button')).filter(b =>
      ['Connect', 'Follow', 'Message', 'Pending'].includes(b.innerText.trim())
    );
    if (actionBtns.length > 0) {
      const results = actionBtns.map(btn => {
        // Walk up to card container (has a profile /in/ link)
        let card = btn;
        for (let j = 0; j < 10; j++) {
          card = card.parentElement;
          if (card.querySelector('a[href*="/in/"]')) break;
        }

        // Profile URL: use the second /in/ link (first is avatar-only)
        const profLinks = Array.from(card.querySelectorAll('a[href*="/in/"]'));
        const nameLink = profLinks.length > 1 ? profLinks[1] : profLinks[0];
        const profileUrl = nameLink ? nameLink.href.split('?')[0] : '';

        // Parse card text line by line
        const lines = (card.innerText || '').split('\n').map(l => l.trim()).filter(l => l);

        const name = (lines[0] || '').replace(/\s*•.*$/, '').trim();
        const degreeRaw = lines.find(l => /•/.test(l));
        const degree = degreeRaw ? degreeRaw.replace(/^[^•]*•/, '').trim() : '';
        const followers = lines.find(l => /followers/i.test(l)) || '';
        const mutualConnections = lines.find(l => /mutual connection/i.test(l)) || '';
        const currentJobLine = lines.find(l => /^Current:/i.test(l));
        const currentJob = currentJobLine ? currentJobLine.replace(/^Current:\s*/i, '') : '';

        // Headline and location: lines between degree and action button
        const actionText = btn.innerText.trim();
        const degreeIdx = lines.findIndex(l => /•/.test(l));
        const actionIdx = lines.indexOf(actionText);
        const startIdx = degreeIdx >= 0 ? degreeIdx + 1 : 1;
        const endIdx = actionIdx >= 0 ? actionIdx : lines.length;
        const middleLines = lines.slice(startIdx, endIdx).filter(l =>
          !/^Current:/i.test(l) && !/followers/i.test(l) && !/mutual connection/i.test(l)
        );
        const headline = middleLines[0] || '';
        const location = middleLines[1] || '';

        return { name, degree, headline, location, currentJob, followers, mutualConnections, profileUrl };
      }).filter(r => r.name && r.profileUrl);
      return resolve(JSON.stringify(results));
    }
    elapsed += 500;
    if (elapsed >= 10000) return reject('no search result cards found after 10s');
    setTimeout(check, 500);
  };
  check();
}).catch(e => e);
```

If this fails, use `read_page` to inspect the DOM, adapt the selectors once, then stop if still failing.

Report: `Found {N} results: {name1}, {name2}, ...`

### 5. Write CSV file

Use the `Write` tool to write the results to `/Users/vvorski/dev/auto-li/output/search-results.csv`.

- If the file does not exist, write a header row first: `Name,Degree,Headline,Location,Current Job,Followers,Mutual Connections,Profile URL`
- If the file already exists, use `Read` to get the existing content and append the new rows.
- Properly escape CSV fields: wrap any field containing commas, quotes, or newlines in double quotes, and double any internal quotes.

### 6. Final summary

Report: `Done — wrote {N} rows to output/search-results.csv. Columns: Name | Degree | Headline | Location | Current Job | Followers | Mutual Connections | Profile URL.`

## Notes

- Only the first visible page of results is captured (~10 results). This command does not paginate.
- Cards are found by action buttons (Connect/Follow/Message/Pending) since LinkedIn obfuscates CSS class names.
- Running the command multiple times appends additional rows to the same CSV file.
