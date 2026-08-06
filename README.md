# picc
this is the repository for the picc app

# Slack Channel Issue Capture — Setup & Usage

This project uses a lightweight Node.js proxy (`server.js`) to pull messages from Slack channels — both public and private — and surface issues, bugs, and feedback into consolidated dashboards like `pi-command-center.html`.

---

## How It Works

The server exposes a `/api/slack/issues` endpoint that:
1. Accepts a comma-separated list of channel names
2. Uses your Slack token (passed as a request header) to call the Slack Web API
3. Fetches the last 80 messages per channel
4. Returns raw messages for the frontend (or Claude) to filter and categorize

The key point: **private channels are included** as long as your Slack token belongs to a user/bot that is a member of that channel.

---

## Prerequisites

- Node.js installed (`node -v` to confirm)
- A Slack token with the following OAuth scopes:
  - `channels:history` — read public channel messages
  - `groups:history` — read private channel messages
  - `channels:read` — list public channels (needed for name → ID resolution)
  - `groups:read` — list private channels

### Getting a Token

**Option A — User token (simplest for personal use):**
1. Go to [api.slack.com/apps](https://api.slack.com/apps) → your app → **OAuth & Permissions**
2. Copy the **User OAuth Token** (`xoxp-...`)
3. Ensure the scopes above are added under **User Token Scopes**

**Option B — Bot token:**
1. Same page → copy **Bot User OAuth Token** (`xoxb-...`)
2. Add scopes under **Bot Token Scopes**
3. Invite the bot to each private channel you want to read: `/invite @your-bot-name`

---

## Starting the Server

```bash
node server.js
```

Server runs on **http://localhost:3000** by default.

---

## Channels Currently Tracked

| Channel | ID | Type | Purpose |
|---|---|---|---|
| `#cloudsuccess-orgcs-hq-questions` | `C05B7R799UJ` | Public | OS/Guide questions, bug reports, process gaps |
| `#cloudsuccess-orgcs-hq-intake` | `C05ATARUVPZ` | Public | Feature requests, enhancement intakes |

To add a channel, see [Adding Channels](#adding-channels) below.

---

## Calling the Endpoint

### From a browser/frontend (with a Slack token in headers):

```js
fetch('http://localhost:3000/api/slack/issues?channels=cloudsuccess-orgcs-hq-questions,cloudsuccess-orgcs-hq-intake', {
  headers: { 'X-Slack-Token': 'xoxp-your-token-here' }
})
.then(r => r.json())
.then(data => console.log(data.messages));
```

### From curl:

```bash
curl -H "X-Slack-Token: xoxp-your-token-here" \
  "http://localhost:3000/api/slack/issues?channels=cloudsuccess-orgcs-hq-questions"
```

The response is `{ ok: true, messages: [...] }` where each message includes `text`, `user`, `ts` (Unix timestamp), and `channel`.

---

## What Gets Captured

When reviewing messages, filter for content related to:

| Topic | Keywords to watch for |
|---|---|
| **Bugs / Errors** | error, insufficient privileges, not working, broken, bug, failed, unexpected |
| **Scheduling issues** | appointment scheduler, booking, available time slots, portal access, scheduler link |
| **Calendar sync** | EAC, OOO, out of office, buffer block, private event, calendar not syncing |
| **Recording / Gemini** | recording, transcript, Gemini Notes, ECI, deleted, 30 days |
| **Portal access** | insufficient privileges, H&T portal, designated contact, provisioning, Basecamp ticket |
| **Google Meet** | join link, separate room, host link, customer link, Teams, Zoom |
| **Feature requests** | request, enhancement, can we, would it be possible, idea, suggestion |
| **Customer feedback** | customer said, customer quote, customer refused, customer can't, friction |

---

## Dashboard Refresh Workflow

The `scheduler-consolidated.html` dashboard is refreshed manually by reading the last 100 messages from each tracked channel, comparing against existing findings, and adding new or updated items.

Steps (done via Claude Code in this project):

1. Read last 100 messages from each channel using the Slack MCP tool
2. Read the Team Feedback Canvas (`F0BEA7ZK2RJ`) for structured team input
3. Compare against all existing dashboard findings (title, severity, description, reporters)
4. Add new findings; update reporters/notes on existing ones
5. Regenerate `scheduler-consolidated.html` with an updated change-summary banner
6. Banner shows: date of refresh, how many items are new, how many updated

Canvas items are tagged with a `Canvas` source badge in the HTML. Slack items are tagged with the channel name (`#hq-questions`, `#hq-intake`).

---

## Adding Channels

To track a new channel:

1. **Get the channel ID** — in Slack, open the channel → click the channel name at the top → scroll to the bottom of the About panel, the ID is shown (e.g. `C05B7R799UJ`). Or use:
   ```bash
   curl -H "Authorization: Bearer xoxp-your-token" \
     "https://slack.com/api/conversations.list?types=public_channel,private_channel&limit=200" \
     | jq '.channels[] | select(.name=="your-channel-name") | .id'
   ```

2. **For private channels** — ensure the token's user or bot is a member of that channel.

3. **Add to the channel table** in this README and to the dashboard's source tags in `scheduler-consolidated.html`.

4. **Pass the channel name** (without `#`) in the `channels` query param when calling `/api/slack/issues`.

---

## Severity Classification

When triaging captured messages into the dashboard, use this scale:

| Severity | When to use |
|---|---|
| **Critical** | Compliance risk (GDPR, SMA), complete workflow blocker with no workaround |
| **High** | Reproducible bug affecting multiple users, or customer-facing friction causing refusals/escalations |
| **Medium** | Recurring issue with a known workaround, or process gap with moderate impact |
| **Low** | Edge case, cosmetic issue, or enhancement with minor impact |
| **Resolved** | Confirmed fix shipped or workaround formally documented in release notes |

---

## Token Security

- Never commit your Slack token to git
- The token is passed as a request header (`X-Slack-Token`) at call time — it is never stored server-side
- For local use, store it in a `.env` file (add `.env` to `.gitignore`) and inject it into your frontend via an environment variable or a local config file
