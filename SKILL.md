---
name: google-workspace
description: Google Workspace — MCP vs CLI decision tree, gws CLI reference, Gmail/Calendar/Drive/Sheets/Docs gotchas, rate limits, and n8n integration. Auto-triggers on Gmail, Google Calendar, Google Drive, Google Docs, Google Sheets, and Workspace API tasks.
disable-model-invocation: false
user-invocable: true
argument-hint: "task description"
---

# Google Workspace Skill

Two complementary tools for Google Workspace:
- **`workspace-mcp`** — MCP server for interactive Claude sessions (~30 structured tools)
- **`gws` CLI** (`@googleworkspace/cli`) — official Google CLI for full API coverage via Bash (200+ endpoints)

Each manages its own OAuth independently.

**As of April 2026.** All APIs v1.

---

## 1. Method Decision Tree

```
I need to read/send Gmail, manage Calendar, Drive, Sheets, Docs
  → workspace-mcp available? → Use MCP (preferred for Claude sessions)

I need Google Chat (threads, messages, edit/delete)
  → gws CLI (workspace-mcp has limited Chat support)

I need Admin API, full Chat API, or batch scripting
  → gws CLI (broader coverage, better for loops/iteration)

I need server-side automation (n8n, cron, background)
  → n8n native Google nodes (Gmail, Calendar, Drive, Sheets)
  → Or: Service Account + Python SDK with domain-wide delegation
```

### workspace-mcp vs gws CLI

| Capability | workspace-mcp (MCP) | gws CLI |
|---|---|---|
| MCP integration | Yes (stdio) | Removed (March 2026) |
| Chat thread replies | Yes (basic) | Yes (full API) |
| Chat thread reading | No (space-level only) | Yes (filter by thread) |
| Chat message edit/delete | No | Yes |
| Chat media upload | No | Yes |
| API coverage | ~30 tools (curated) | Full Discovery Service (200+) |
| Auth | OAuth via 1Password env vars | Built-in OAuth (self-managed) |
| Best for | Claude MCP tools | Bash scripting, automation |

**Why gws has no MCP mode:** Google added MCP to gws in v0.5.0, then [removed it 2 days later in v0.8.0](https://github.com/googleworkspace/cli/pull/275) — 200-400 auto-generated tool definitions from Discovery Documents consumed 40K-100K context tokens. A prior PR #172 had implemented meta-tool discovery (expose 2 tools, let LLM discover the rest) which would have solved this, but Google chose to remove entirely. CLI-first is Google's official architecture.

---

## 2. gws CLI Reference

### Install & Auth

```bash
npm install -g @googleworkspace/cli
gws auth login --scopes drive,gmail,calendar,chat,sheets,docs,tasks
gws auth status
```

**Environment variables:**
- `GOOGLE_WORKSPACE_CLI_CLIENT_ID` / `GOOGLE_WORKSPACE_CLI_CLIENT_SECRET` — OAuth credentials
- `GOOGLE_WORKSPACE_CLI_CREDENTIALS_FILE` — path to OAuth or service account JSON
- `GOOGLE_WORKSPACE_CLI_TOKEN` — pre-obtained access token (highest priority)
- `GOOGLE_WORKSPACE_CLI_CONFIG_DIR` — override config dir (default: `~/.config/gws`)

### Services

drive, sheets, gmail, calendar, docs, slides, tasks, people, chat, classroom, forms, keep, meet, events, admin-reports

### Key Commands

```bash
# General pattern:
gws <service> <resource> <method> --params '{}' --json '{}'

# Gmail — list messages
gws gmail users messages list --params '{"userId": "me", "q": "is:unread"}'

# Drive — list files
gws drive files list --params '{"pageSize": 10}'

# Calendar — list events
gws calendar events list --params '{"calendarId": "primary", "timeMin": "2026-03-01T00:00:00Z"}'

# Sheets — read values
gws sheets spreadsheets values get --params '{"spreadsheetId": "...", "range": "Sheet1!A1:E10"}'

# Chat — list spaces
gws chat spaces list

# Chat — send message
gws chat spaces messages create \
  --params '{"parent": "spaces/SPACE_ID"}' \
  --json '{"text": "Hello from gws"}'

# Schema introspection (see all params for any method):
gws schema drive.files.list
```

### Chat Thread Operations (Full API)

```bash
# Reply to existing thread:
gws chat spaces messages create \
  --params '{"parent": "spaces/SPACE_ID", "messageReplyOption": "REPLY_MESSAGE_FALLBACK_TO_NEW_THREAD"}' \
  --json '{"text": "Thread reply", "thread": {"name": "spaces/SPACE_ID/threads/THREAD_ID"}}'

# Reply by custom thread key:
gws chat spaces messages create \
  --params '{"parent": "spaces/SPACE_ID", "messageReplyOption": "REPLY_MESSAGE_FALLBACK_TO_NEW_THREAD"}' \
  --json '{"text": "Keyed reply", "thread": {"threadKey": "my-custom-key"}}'

# List messages in a thread:
gws chat spaces messages list \
  --params '{"parent": "spaces/SPACE_ID", "filter": "thread.name = spaces/SPACE_ID/threads/THREAD_ID"}'

# Edit / Delete a message:
gws chat spaces messages patch \
  --params '{"name": "spaces/SPACE_ID/messages/MSG_ID", "updateMask": "text"}' \
  --json '{"text": "Updated text"}'
gws chat spaces messages delete --params '{"name": "spaces/SPACE_ID/messages/MSG_ID"}'
```

**`messageReplyOption` values:**
- `REPLY_MESSAGE_FALLBACK_TO_NEW_THREAD` — reply if thread exists, else create new
- `REPLY_MESSAGE_OR_FAIL` — reply or error if thread doesn't exist

### Output Formats

```bash
--format json    # Default
--format table   # Human-readable
--format yaml / csv
--page-all       # Auto-paginate (NDJSON)
--page-limit 5   # Max pages
```

---

## 3. Gmail Search Operators

```
from:bank@example.com after:2026/01/01 has:attachment
subject:invoice is:unread
filename:pdf larger:5M
label:important in:inbox
```

---

## 4. Rate Limits

| API | Quota | Per |
|-----|-------|-----|
| Gmail read | 250 units/sec | user |
| Gmail send | 100/day (free), 2000/day (Workspace) | user |
| Calendar | 500 requests/100sec | user |
| Drive | 12,000 requests/min | project |
| Sheets read | 300 requests/min | project |
| Sheets write | 60 requests/min | user |
| Docs | 300 read, 60 write/min | user |

**Batch requests** reduce quota — one batch = one request regardless of sub-requests (up to 100).

---

## 5. n8n Integration

n8n has native Google nodes (Gmail, Calendar, Drive, Sheets). Use these instead of HTTP Request when possible.

For operations not covered by n8n nodes, use HTTP Request node with OAuth2 credential.

---

## 6. Gotchas

1. **OAuth scopes must be declared upfront** — adding new scopes requires re-authentication. Plan scope needs before first auth.
2. **Gmail `format='raw'`** — returns RFC 2822 encoded. Use `format='full'` for structured parsing.
3. **Calendar timezone** — always specify `timeZone` on events.
4. **Drive file IDs are permanent** — files keep their ID even when renamed or moved.
5. **Sheets A1 notation** — sheet name with spaces needs single quotes: `'My Sheet'!A1:E`.
6. **Sheets value render** — `FORMATTED_VALUE` returns display strings. `UNFORMATTED_VALUE` returns raw numbers/dates.
7. **Docs API is positional** — insertions shift indices. Process requests in reverse order or use named ranges.
8. **Batch request limit** — max 100 sub-requests per batch across all APIs.
9. **Service account vs OAuth** — service accounts don't have a "user" inbox/calendar. Use domain-wide delegation with `with_subject()`.
10. **Google Workspace MCP** — uses `core` tier by default. For Tasks, Contacts, Chat, use `extended` tier.
11. **Push notifications (watch)** — Calendar and Drive support push notifications via webhooks. Requires verified domain + HTTPS endpoint.
12. **File export limits** — Google Docs export to PDF has a 10MB limit.
13. **`workspace-mcp` token cache + alias trap** — `workspace-mcp` caches OAuth tokens at `~/.google_workspace_mcp/credentials/{email}.json`, keyed by the email Google's `userinfo` endpoint returns. **Account aliases produce separate files for the same Google account** — e.g. signing in once as `user@primary.com` and once as `user@alias.org` creates two credential files; whichever Google echoes back on the next auth wins, causing flaky scope/expiry behavior. **Expired tokens manifest as "permission denied" / "insufficient scope" errors that look like consent screen problems but aren't** — Google's account permissions page will show all scopes granted while the cached token is stale. **Diagnostic:** `python3 -c "import json; d=json.load(open('PATH')); print(d.get('scopes')); print(d.get('expiry'))"` reads the actual scope set + expiry from the file. **Fix recipe:** delete every file under `~/.google_workspace_mcp/credentials/`, then re-trigger auth (any workspace-mcp tool call, or hit the MCP's `localhost:8099` auth endpoint), and on Google's account chooser pick the **canonical** email form (not an alias) so the new credential file lands at the canonical filename.

---

## 7. Large Payloads & Workarounds

### The hard ceiling — `gws --json` ~1 MB (ARG_MAX)

- `gws … --json <JSON>` passes the body as a shell argument. macOS/Linux kernel `ARG_MAX` (~1 MB on macOS, ~128 KB–2 MB on Linux) makes anything above fail with `zsh: argument list too long`.
- **`--json -` (stdin) is NOT supported** — gws returns `"Invalid --json body: EOF while parsing a value"`.
- **`--json @file` is NOT supported** — gws treats the `@file` as literal JSON and fails to parse.
- **`--upload <path>` wraps the file in `multipart/related; boundary=gws_…`** — Gmail's `drafts.create` rejects it with `"Media type 'multipart/related' is not supported"` because the endpoint wants bare `message/rfc822`. Upload flag works for Drive/Sheets but **not** for the Gmail endpoints that expect raw RFC-822.

### Can't reuse gws credentials from Python

- `gws auth export` decrypts `credentials.enc` and returns `{client_id, client_secret, refresh_token, type}`.
- **The exported `client_secret` is frozen at the time gws first authenticated.** If `~/.config/gws/client_secret.json` was rotated later, the exported secret no longer matches Google → token-refresh fails with `invalid_client: "The provided client secret is invalid."` **gws itself keeps working** because it uses the cached access token in `token_cache.json` (AEAD-encrypted, binary header `0x87 0xD3 …`, unreadable externally) instead of calling the refresh endpoint.
- Google's token endpoint **requires** `client_secret` for `grant_type=refresh_token` (returns `400: client_secret is missing.` without it) — no PKCE fallback for installed-app clients.
- **gws has no `GWS_ACCESS_TOKEN` / `GOOGLE_ACCESS_TOKEN` env var override.** The Bearer token it sends cannot be captured without an HTTPS MITM proxy with a trusted CA (mitmproxy, etc.).

### The escape hatch — Apps Script as a proxy for large Gmail drafts with attachments

When the draft body + N PDFs exceeds ~1 MB, use this pattern:

1. **Upload each attachment to Drive** via `mcp__google-workspace-daniel__create_drive_file` with `fileUrl: "https://…"` (fetches remote), or `fileUrl: "file:///tmp/foo.pdf"` (local). Keep the IDs.
2. **Create an Apps Script project** via `mcp__google-workspace-daniel__create_script_project`.
3. **Push code** via `mcp__google-workspace-daniel__update_script_content` with both an `appsscript` (JSON manifest, required) and `Code` (SERVER_JS) file. The manifest MUST declare `oauthScopes` — e.g. `["https://www.googleapis.com/auth/gmail.compose", "https://www.googleapis.com/auth/drive.readonly"]`.
4. **Function body** — `GmailApp.createDraft(to, subject, body, {attachments: fileIds.map(id => DriveApp.getFileById(id).getBlob()), from: 'alias@…', replyTo: '…', name: '…'})`.
5. **Run the function.**

### Apps Script pre-requisites & first-run authorization trap

- **Apps Script API must be enabled per-user** at `https://script.google.com/home/usersettings`. Without it, `create_script_project` fails with `403: "User has not enabled the Apps Script API."`
- **`run_script_function` returns `404 "Requested entity was not found"` on the first call** — even with `dev_mode: true`. The script needs a one-time browser-side authorization before the execution API can invoke it.
- **Fix:** open `https://script.google.com/d/{SCRIPT_ID}/edit`, select the function in the dropdown, click ▶ Run, accept the OAuth consent dialog (Gmail + Drive scopes). After that, `run_script_function` works programmatically for subsequent calls.

### JS string escaping for Apps Script push

- `update_script_content` takes `files: [{name, type, source}]`. The `source` is a single-line JSON string — literal newlines in the body will be parsed as line breaks in the JS source and crash at runtime with `SyntaxError: Invalid or unexpected token`.
- Build the source in Python with `json.dumps(body_text)` to get a properly-escaped JS string literal (handles `\n`, embedded `"`, unicode via `\uXXXX`). Interpolate the result into the JS `var body = …;` template.
- Hebrew/RTL content survives transport as `\uXXXX` escapes — no UTF-8 handling needed inside Apps Script.

### Name cadence

- Drive-upload → Apps-Script-proxy is the canonical pattern for **any** Gmail send/draft where the MIME exceeds `gws --json`'s limit. Don't fight gws — route around it.

