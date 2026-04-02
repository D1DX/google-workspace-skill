# Google Workspace Skill

[![Author](https://img.shields.io/badge/Author-Daniel_Rudaev-000000?style=flat)](https://github.com/daniel-rudaev)
[![Studio](https://img.shields.io/badge/Studio-D1DX-000000?style=flat)](https://d1dx.com)
[![Google](https://img.shields.io/badge/Google_Workspace-Skill-4285F4?style=flat&logo=google&logoColor=white)](https://workspace.google.com)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat)](./LICENSE)

Google Workspace skill for AI agents. Covers the decision tree between `workspace-mcp` and the `gws` CLI, a complete `gws` command reference, Gmail search operators, rate limits, n8n integration, and gotchas. Built from real production use across Gmail, Calendar, Drive, Docs, Sheets, and Chat.

## What's Included

| Topic | What it covers |
|-------|---------------|
| Method Decision Tree | workspace-mcp vs gws CLI vs n8n vs Service Account — when to use each |
| workspace-mcp vs gws CLI | Side-by-side capability comparison, why gws has no MCP mode |
| gws CLI Reference | Install, auth, key commands for all services, output formats |
| Chat Thread Operations | Reply to threads, filter messages by thread, edit/delete messages |
| Gmail Search Operators | from, after, has:attachment, filename, label, in:inbox |
| Rate Limits | Quotas for Gmail, Calendar, Drive, Sheets, Docs — per user and per project |
| n8n Integration | Native Google nodes, HTTP Request fallback for uncovered operations |
| Gotchas | 12 hard-won lessons — OAuth scope planning, date handling, Docs positional API, service accounts |

## Install

### Claude Code

```bash
git clone https://github.com/D1DX/google-workspace-skill.git
cp -r google-workspace-skill ~/.claude/skills/google-workspace
```

Or as a git submodule:

```bash
git submodule add https://github.com/D1DX/google-workspace-skill.git path/to/skills/google-workspace
```

### Other AI Agents

Copy `SKILL.md` into your agent's prompt or knowledge directory. The skill is structured markdown — works with any LLM agent that reads reference files.

## Structure

```
google-workspace-skill/
└── SKILL.md    — Main skill (decision tree, gws CLI reference, rate limits, gotchas)
```

## Note: gws MCP Removal

Google added MCP support to the `gws` CLI in v0.5.0, then [removed it two days later in v0.8.0](https://github.com/googleworkspace/cli/pull/275). The 200–400 auto-generated tool definitions from Discovery Documents consumed 40K–100K context tokens. The skill documents this history and explains the CLI-first architecture that replaced it.

## Sources

- **gws CLI:** Verified against [@googleworkspace/cli](https://github.com/googleworkspace/cli) npm package and live usage (April 2026).
- **Google APIs:** Verified against [Google Workspace API documentation](https://developers.google.com/workspace) and [Gmail API reference](https://developers.google.com/gmail/api/reference/rest).
- **Rate limits:** Verified from [Google API quota documentation](https://developers.google.com/workspace/guides/implement-quota) (April 2026).

## Credits

Built by [Daniel Rudaev](https://github.com/daniel-rudaev) at [D1DX](https://d1dx.com).

## License

MIT License — Copyright (c) 2026 Daniel Rudaev @ D1DX
