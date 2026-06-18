# Design: `message-person` LinkedIn skill

**Date:** 2026-06-18  
**Status:** Approved

---

## Purpose

A slash command to send a LinkedIn message to a single existing connection, with the message text provided inline as an argument. Intended for targeted, one-off outreach (e.g. after looking someone up in METIS).

---

## Command signature

```
/message-person <profile-url> <message text>
```

- First whitespace-delimited token → `profile_url`
- Everything after the first token → `message_text`
- Both are required. If either is missing, stop and tell the user the correct usage.

**Example:**
```
/message-person https://www.linkedin.com/in/guyvankoolwijk/ Hey Guy, wanted to reach out!
```

---

## Steps

### 1. Parse arguments
Split `$ARGUMENTS` on the first whitespace. First token = `profile_url`, remainder = `message_text`. If either is empty, error out with usage instructions.

### 2. Navigate to the profile
Use `mcp__claude-in-chrome__navigate` to open the profile URL in the current tab.

### 3. Click the Message button
Poll up to 10 seconds for a button or link whose visible text is exactly "Message" (not "Connect", "Follow", etc.). Click it.

```javascript
new Promise((resolve, reject) => {
  let elapsed = 0;
  const check = () => {
    const btn = Array.from(document.querySelectorAll('button, a'))
      .find(el => el.textContent.trim() === 'Message');
    if (btn) { btn.click(); return resolve('clicked'); }
    elapsed += 300;
    if (elapsed >= 10000) return reject('Message button not found after 10s');
    setTimeout(check, 300);
  };
  check();
}).catch(e => e);
```

If not found, stop and report. The person may not be a 1st-degree connection.

### 4. Wait for compose editor
Poll up to 10 seconds for the messaging compose editor to appear.

```javascript
new Promise((resolve, reject) => {
  let elapsed = 0;
  const check = () => {
    const el = document.querySelector(
      '.msg-form__contenteditable, [contenteditable="true"][role="textbox"], textarea[name="message"]'
    );
    if (el) return resolve('editor ready');
    elapsed += 300;
    if (elapsed >= 10000) return reject('editor not found after 10s');
    setTimeout(check, 300);
  };
  check();
}).catch(e => e);
```

### 5. Type the message
Focus the editor and insert the message text.

```javascript
const el = document.querySelector(
  '.msg-form__contenteditable, [contenteditable="true"][role="textbox"], textarea[name="message"]'
);
el.focus();
document.execCommand('insertText', false, MESSAGE_TEXT_HERE);
'typed';
```

### 6. Click Send
Poll up to 5 seconds for the Send button to be enabled, then click it.

```javascript
new Promise((resolve, reject) => {
  let elapsed = 0;
  const check = () => {
    const btn = document.querySelector('button.msg-form__send-button');
    if (btn && !btn.disabled) { btn.click(); return resolve('sent'); }
    elapsed += 300;
    if (elapsed >= 5000) return reject('send button not found or disabled after 5s');
    setTimeout(check, 300);
  };
  check();
}).catch(e => e);
```

### 7. Confirm
Extract the person's name from the page title or profile heading and report: "Message sent to **{name}**."

---

## Error handling

- Missing args → usage error, stop
- Message button not found → likely not a 1st connection, stop and report
- Editor not found → stop and report
- Send button not found/disabled → stop and report
- All errors: debug mode ON — stop immediately, do not try alternatives

---

## Out of scope

- No log file (one-shot, not bulk)
- No retry logic
- No support for non-connections (InMail)
