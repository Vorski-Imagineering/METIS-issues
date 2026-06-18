# METIS-pub

A public collection of automation tools and workflows built around the METIS ecosystem.

These tools are designed for people using [Claude Code](https://claude.ai/code) who want to automate repetitive work — no programming background required.

---

## Modules

### [LinkedIn Automation](.claude/linkedin-automation/README.md)

Automate LinkedIn connection management directly from Claude Code:
- List pending invitations
- Accept invitations and auto-send a welcome message
- Message existing connections (with duplicate tracking)
- Export people search results to Google Sheets

**→ [Setup and usage guide](.claude/linkedin-automation/README.md)**

---

### [METIS API](.claude/metis-api/README.md)

Query the live METIS instance from Claude Code using natural language:
- Search people and holons
- Browse worklists and responsible items
- Record relationship notes and advance journey steps

**→ [Setup and usage guide](.claude/metis-api/README.md)**  
**→ [Full API reference](.claude/metis-api/API.md)**

---

## Using these tools

All automations run as Claude Code slash commands. Open this folder in Claude Code, follow the setup guide for the module you want, and type `/command-name` to run it.

LinkedIn tools use your live Chrome browser (via the [Claude in Chrome](https://chromewebstore.google.com) extension). METIS API tools use `curl` via the terminal — no browser needed.

---

## Issue tracking

This repo also tracks public issues for the METIS project. If you find a bug or want to request a feature, [open an issue](../../issues).
