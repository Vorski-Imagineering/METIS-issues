---
description: Accept multiple pending LinkedIn invitations serially, sending the message from texts/accept-invite.txt to each
allowed-tools: Read, mcp__claude-in-chrome__tabs_context_mcp, mcp__claude-in-chrome__tabs_create_mcp, mcp__claude-in-chrome__navigate, mcp__claude-in-chrome__javascript_tool, mcp__claude-in-chrome__find, mcp__claude-in-chrome__form_input, mcp__claude-in-chrome__read_page
---

Accept **$ARGUMENTS** pending LinkedIn connection invitations, one at a time, by running the `/accept-one` flow that many times serially.

## Steps

1. Parse `$ARGUMENTS` as an integer `N`. If it is missing, not a number, or less than 1, stop and tell the user: "Please provide a number, e.g. `/accept-many 5`."

2. Run the full `/accept-one` flow **N times**, one after the other. Do not start the next iteration until the current one has fully completed (message sent and confirmed).

3. Keep a running count. After each successful iteration print: `[{i}/{N}] Message sent to **{name}**.`

4. If any iteration fails at any step, stop immediately and report the exact error and which iteration it failed on (e.g. "Failed on iteration 2/5: accept button not found"). Do not continue to the next iteration.

5. When all N iterations are done, print a final summary: `Done — sent messages to N connections: {name1}, {name2}, …`
