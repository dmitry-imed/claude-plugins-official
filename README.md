# Discord Threads

Discord plugin for Claude Code with per-session thread routing. Fork of the official Discord plugin with added support for session-isolated threads and full permission context in messages.

When the bot receives a message, the MCP server forwards it to Claude and provides tools to reply, react, and edit messages.

## Prerequisites

- [Node.js](https://nodejs.org) 22+ — the MCP server runs on Node with zero npm dependencies.

## Quick Setup
> Default pairing flow for a single-user DM bot. See [ACCESS.md](./ACCESS.md) for groups and multi-user setups.

**1. Create a Discord application and bot.**

Go to the [Discord Developer Portal](https://discord.com/developers/applications) and click **New Application**. Give it a name.

Navigate to **Bot** in the sidebar. Give your bot a username.

Scroll down to **Privileged Gateway Intents** and enable **Message Content Intent** — without this the bot receives messages with empty content.

**2. Generate a bot token.**

Still on the **Bot** page, scroll up to **Token** and press **Reset Token**. Copy the token — it's only shown once. Hold onto it for step 6.

**3. Invite the bot to a server.**

Discord won't let you DM a bot unless you share a server with it.

Navigate to **OAuth2** → **URL Generator**. Select the `bot` scope. Under **Bot Permissions**, enable:

- View Channels
- Send Messages
- Send Messages in Threads
- Create Private Threads
- Read Message History
- Attach Files
- Add Reactions

Integration type: **Guild Install**. Copy the **Generated URL**, open it, and add the bot to any server you're in.

> For DM-only use you technically need zero permissions — but enabling them now saves a trip back when you want guild channels or session threads later.

**4. Install the plugin.**

These are Claude Code commands — run `claude` to start a session first.

Add the marketplace:
```
/plugin marketplace add axiumfoundry/claude-discord-threads-plugin
```

Install the plugin:
```
/plugin install discord-threads@axiumfoundry-plugins
```

**5. Restart Claude Code.**

Exit and start a new `claude` session. The skill commands (`/discord-threads:configure`, `/discord-threads:access`) won't be available until you restart.

**6. Give the server the token.**

```
/discord-threads:configure MTIz...
```

This saves the token to `~/.claude/channels/discord/.env` and auto-approves the Discord MCP tools so replies don't trigger permission prompts (`~/.claude/settings.json`).

You can also write the `.env` file by hand, or set the variable in your shell environment — shell takes precedence.

> To run multiple bots on one machine (different tokens, separate allowlists), point `DISCORD_STATE_DIR` at a different directory per instance.

**7. Relaunch with the channel flag.**

The server won't connect without this — exit your session and start a new one:

```sh
claude --dangerously-load-development-channels plugin:discord-threads@axiumfoundry-plugins
```

> The `--dangerously-load-development-channels` flag is required until the plugin is listed in the official Claude plugin directory. For convenience, add a shell alias:
> ```sh
> alias claude-discord='claude --dangerously-load-development-channels plugin:discord-threads@axiumfoundry-plugins'
> ```

**8. Pair.**

With Claude Code running from the previous step, DM your bot on Discord — it replies with a pairing code. If the bot doesn't respond, make sure you launched with the channel flag (step 7). In your Claude Code session:

```
/discord-threads:access pair <code>
```

Your next DM reaches the assistant.

**9. Lock it down.**

Pairing is for capturing IDs. Once you're in, switch to `allowlist` so strangers don't get pairing-code replies. Ask Claude to do it, or `/discord-threads:access policy allowlist` directly.

## Session threads

When running multiple Claude Code sessions simultaneously, each session can create its own Discord thread so messages are routed to the correct session instead of being broadcast to all of them.

**Setup:** Enable Developer Mode in Discord (User Settings → Advanced), right-click the text channel where threads should be created → Copy Channel ID. Then run:

```
/discord-threads:configure threads <channel_id>
```

This saves `DISCORD_THREAD_CHANNEL_ID` to `~/.claude/channels/discord/.env`. To disable: `/discord-threads:configure threads off`.

**How it works:**

- On startup, the bot creates a private thread in the configured channel, named after the project directory and git branch (e.g. `myapp/feature-auth 2026-03-27 a3f1`).
- Messages in that thread are only delivered to the session that created it.
- DMs continue to work as before — routed to whichever session receives them.
- When the session ends, the bot posts "Session ended." in the thread.

**Optional env vars** (add manually to `~/.claude/channels/discord/.env`):

| Variable | Default | Description |
| --- | --- | --- |
| `DISCORD_SESSION_NAME` | _(auto: `dir/branch date suffix`)_ | Explicit thread name override (max 100 chars). |
| `DISCORD_SESSION_ARCHIVE` | `true` | Archive the thread when the session ends. Set to `false` to keep threads open. |

**Permissions:** Private threads require the bot to have **Create Private Threads** permission. If missing, the server falls back to public threads automatically.

## Access control

See **[ACCESS.md](plugins/discord-threads/ACCESS.md)** for DM policies, guild channels, mention detection, delivery config, skill commands, and the `access.json` schema.

Quick reference: IDs are Discord **snowflakes** (numeric — enable Developer Mode, right-click → Copy ID). Default policy is `pairing`. Guild channels are opt-in per channel ID.

## Tools exposed to the assistant

| Tool | Purpose |
| --- | --- |
| `reply` | Send to a channel. Takes `chat_id` + `text`, optionally `reply_to` (message ID) for native threading and `files` (absolute paths) for attachments — max 10 files, 25MB each. Auto-chunks; files attach to the first chunk. Returns the sent message ID(s). |
| `react` | Add an emoji reaction to any message by ID. Unicode emoji work directly; custom emoji need `<:name:id>` form. |
| `edit_message` | Edit a message the bot previously sent. Useful for "working…" → result progress updates. Only works on the bot's own messages. |
| `fetch_messages` | Pull recent history from a channel (oldest-first). Capped at 100 per call. Each line includes the message ID so the model can `reply_to` it; messages with attachments are marked `+Natt`. Discord's search API isn't exposed to bots, so this is the only lookback. |
| `download_attachment` | Download all attachments from a specific message by ID to `~/.claude/channels/discord/inbox/`. Returns file paths + metadata. Use when `fetch_messages` shows a message has attachments. |

Inbound messages trigger a typing indicator automatically — Discord shows
"botname is typing…" while the assistant works on a response.

## Attachments

Attachments are **not** auto-downloaded. The `<channel>` notification lists
each attachment's name, type, and size — the assistant calls
`download_attachment(chat_id, message_id)` when it actually wants the file.
Downloads land in `~/.claude/channels/discord/inbox/`.

Same path for attachments on historical messages found via `fetch_messages`
(messages with attachments are marked `+Natt`).

## Troubleshooting

**"not on the approved channels allowlist"**

Add `--dangerously-load-development-channels` to your launch command (see step 7). This flag is required until the plugin is listed in the official Claude plugin directory.

**Bot doesn't respond to DMs**

- Make sure you launched with the channel flag: `claude --dangerously-load-development-channels plugin:discord-threads@axiumfoundry-plugins`
- Check that **Message Content Intent** is enabled in the Discord Developer Portal (Bot → Privileged Gateway Intents).
- Confirm you share a server with the bot — Discord doesn't allow DMs otherwise.

**"not found" when running `/discord-threads:configure`**

Restart Claude Code after installing the plugin. Skills aren't loaded until the next session.

**Every reply triggers a permission prompt**

Re-run `/discord-threads:configure` with your token — it auto-adds the tool permissions to `~/.claude/settings.json`.

**Guild channel messages are ignored**

Guild channels are off by default. Opt in with:

```
/discord-threads:access group add <channelId>
```

Find the channel ID by enabling Developer Mode in Discord, then right-clicking the channel → Copy Channel ID.
