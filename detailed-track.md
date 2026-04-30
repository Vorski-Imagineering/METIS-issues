# Detailed Token Tracking Plan

## Overview

Hermes currently provides platform-level token aggregation but lacks per-user/per-channel tracking. This plan outlines how to implement detailed tracking to answer "which channel/person talking with you is spending how many tokens?"

## Current State Analysis

### What Hermes Tracks (via `hermes insights`)
- Total input/output/total tokens by platform
- Session counts and messages
- Model usage breakdown
- Time periods

### What Hermes Does NOT Track
- **Per-user token counts** — No attribution to individual Telegram users
- **Per-channel totals** — Can't see tokens spent in specific groups/channels
- **Per-message token counts** — Only last message's tokens stored in sessions.json

## Implementation Plan

### Phase 1: Gateway-Level Logging

**Goal:** Capture token usage with user/session metadata

**Approach:**
1. Modify Hermes gateway to log token usage per message
2. Include metadata: `chat_id`, `user_id`, `thread_id`, `platform`, `timestamp`
3. Store in a structured format (JSONL) for easy analysis

**Files to modify:**
- `gateway/run.py` — Add token logging middleware
- `gateway/platforms/telegram.py` — Add per-message tracking

### Phase 2: Token Attribution Service

**Goal:** Maintain running totals per user/channel

**Approach:**
1. Create a background service that reads token logs
2. Aggregates by chat_id and user_id
3. Exposes metrics via simple API or file output

**Schema:**
```json
{
  "chat_id": "-1003860423839",
  "user_id": "383220557",
  "date": "2026-04-30",
  "input_tokens": 12345,
  "output_tokens": 6789,
  "total_tokens": 19134,
  "messages": 5
}
```

### Phase 3: Reporting Integration

**Goal:** Include detailed tracking in daily-bip reports

**Approach:**
1. Add `token-tracking` skill that reads aggregated data
2. Update `daily-bip` to call token-tracking for per-channel breakdown
3. Show top users/channels by token usage

## Technical Details

### Option A: Log-Based Aggregation

```python
# ~/.hermes/scripts/token_logger.py
import json
from datetime import datetime
from pathlib import Path

LOG_FILE = Path.home() / '.hermes' / 'token_logs' / 'usage.jsonl'

def log_tokens(platform, chat_id, user_id, input_tokens, output_tokens):
    entry = {
        'timestamp': datetime.now().isoformat(),
        'platform': platform,
        'chat_id': chat_id,
        'user_id': user_id,
        'input_tokens': input_tokens,
        'output_tokens': output_tokens
    }
    with open(LOG_FILE, 'a') as f:
        f.write(json.dumps(entry) + '\n')
```

### Option B: Database Backend

For production use, consider SQLite for efficient querying:

```sql
CREATE TABLE token_usage (
    id INTEGER PRIMARY KEY,
    timestamp DATETIME,
    platform TEXT,
    chat_id TEXT,
    user_id TEXT,
    input_tokens INTEGER,
    output_tokens INTEGER
);

CREATE INDEX idx_chat_date ON token_usage(chat_id, date(timestamp));
```

## Pitfalls & Considerations

1. **Privacy:** Token tracking reveals conversation patterns. Consider anonymization.

2. **Performance:** Logging every token count adds overhead. Batch writes recommended.

3. **Migration:** Historical data won't be available. Clear in reports.

4. **Storage:** Token logs grow quickly. Implement rotation (keep 90 days).

## Estimated Effort

| Phase | Task | Hours |
|-------|------|-------|
| 1 | Gateway logging patch | 3-4 |
| 2 | Aggregation service | 2-3 |
| 3 | Reporting integration | 1-2 |
| **Total** | | **6-9 hours** |

## Next Steps

1. Review gateway code structure
2. Decide on log format and location
3. Prototype token logging in one platform (Telegram)
4. Test with sample data
5. Build aggregation and reporting

---

*Created: April 2026*
*Related: `token-usage-tracker` skill, `daily-bip` skill*