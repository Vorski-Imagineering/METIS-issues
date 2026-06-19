---
description: Find a person's LinkedIn profile URL by name, disambiguating by organization then country
allowed-tools: Read, mcp__claude-in-chrome__tabs_context_mcp, mcp__claude-in-chrome__tabs_create_mcp, mcp__claude-in-chrome__navigate, mcp__claude-in-chrome__javascript_tool
---

Find the LinkedIn profile URL for a person by searching their name, then disambiguating
by organization and country. Used standalone or as the lookup step when filling a
spreadsheet's empty LinkedIn column.

**Arguments:** `$ARGUMENTS` — `Name | Organization | Country` (org and country optional).
Example: `Alexander Weisberg | Rhizomatic Capital | United States`

> **Debug mode is ON.** If any step fails or returns an unexpected result, stop immediately
> and report the exact error. Do NOT fall back to screenshots — this command inspects the
> LinkedIn DOM with JavaScript only.

## Steps

### 1. Get browser context
Use `tabs_context_mcp` for a valid tab ID (create one with `tabs_create_mcp` if needed).
The user must already have an active LinkedIn session in Chrome.

### 2. Navigate to the people search
Navigate to:
`https://www.linkedin.com/search/results/people/?keywords=<URL-encoded name>`

If the "name" is an email address or obvious placeholder (e.g. `Test Test`), skip — it is
not searchable by name; report it as skipped.

### 3. Extract candidate cards (JavaScript only, no screenshots)
LinkedIn obfuscates CSS classes and wraps each card's text inside the profile anchor, so
extract by deduping `a[href*="/in/"]` links and parsing the text-rich anchor. Poll up to
12s for results to render:

```javascript
new Promise((resolve, reject) => {
  const STOP = ['Connect','Message','Follow','Pending','View profile'];
  let elapsed = 0;
  const check = () => {
    const byUrl = {};
    document.querySelectorAll('a[href*="/in/"]').forEach(a => {
      const url = a.href.split('?')[0];
      const text = (a.innerText || '').trim();
      if (!byUrl[url] || text.length > byUrl[url].length) byUrl[url] = text;
    });
    const results = Object.entries(byUrl).map(([url, text]) => {
      const lines = text.split('\n').map(l => l.trim()).filter(Boolean);
      const name = (lines[0] || '').replace(/\s*•.*$/, '').trim();
      const degIdx = lines.findIndex(l => /•/.test(l));
      const start = degIdx >= 0 ? degIdx + 1 : 1;
      let stopIdx = -1;
      for (let i = start; i < lines.length; i++) { if (STOP.includes(lines[i])) { stopIdx = i; break; } }
      const end = stopIdx >= 0 ? stopIdx : lines.length;
      const mid = lines.slice(start, end).filter(l => !/mutual connection/i.test(l) && !/followers/i.test(l));
      return { name, headline: mid[0] || '', location: mid[1] || '', profileUrl: url, lineCount: lines.length };
    }).filter(r => r.name && r.lineCount >= 3);
    if (results.length > 0) return resolve(JSON.stringify(results));
    elapsed += 500;
    if (elapsed >= 12000) return reject('no search result cards found after 12s');
    setTimeout(check, 500);
  };
  check();
}).catch(e => 'ERROR: ' + e);
```

### 4. Apply the match algorithm
1. **Name-narrow:** keep candidates whose last name equals the target's and whose first
   name matches or is a prefix/nickname (Alex↔Alexander). If none qualify, report
   "no confident match" and leave blank. If exactly one qualifies → that is the URL.
2. **Organization:** among the remaining, keep those whose headline/current company
   contains a *significant* token of the target org (length ≥ 4, ignoring generic words:
   capital, group, fund, ventures, foundation, the, and, impact, partners, llc, gmbh).
   If exactly one → that URL. If several → carry them forward. If none → keep the prior set.
3. **Country:** among the carried-forward set, keep those whose location contains the
   target country. If exactly one → that URL.
4. **Prefer the substantive profile:** if several remain but exactly one has a real,
   relevant headline while the rest are empty (headline `--`) or clearly unrelated
   (student/different field), choose that one. LinkedIn name searches commonly return a
   real profile plus near-empty duplicates — do not write the empty duplicate.
5. **Still genuinely multiple:** if 2+ remain that are all plausible, output those URLs
   (cap 3), to be written into one cell separated by newlines, and flag the row as ambiguous.

### 5. Report
Return the chosen URL(s) plus the candidates considered, so the caller can verify.

## Notes
- Only the first page of results (~10) is inspected; that is sufficient for name lookups.
- Org names rarely appear verbatim in headlines, so country is often the deciding factor.
- This command is read-only on LinkedIn; it never sends connection requests or messages.
