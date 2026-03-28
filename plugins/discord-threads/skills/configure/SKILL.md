---
name: configure
description: Set up the Discord channel — save the bot token, configure session threads, and review access policy. Use when the user pastes a Discord bot token, asks to configure Discord, asks about session threads, asks "how do I set this up" or "who can reach me," or wants to check channel status.
allowed-tools: Read, Write, Bash(ls *), Bash(mkdir *), Bash(chmod *), Bash(cat *)
---

# /discord-threads:configure — Discord Channel Setup

Writes the bot token and session thread settings to
`~/.claude/channels/discord/.env` and orients the user on access policy.
The server reads the `.env` file at boot.

Arguments passed: `$ARGUMENTS`

---

## Dispatch on arguments

### No args — status and guidance

Read both state files and give the user a complete picture:

1. **Token** — check `~/.claude/channels/discord/.env` for
   `DISCORD_BOT_TOKEN`. Show set/not-set; if set, show first 6 chars masked.

2. **Channel allowlist** — check if the plugin is in the managed settings
   allowlist (see "Channel allowlist check" below). Show allowed/not-allowed.
   If not allowed, tell the user it will be fixed automatically when they
   save a token, or they can run `/discord-threads:configure allow-plugin`.

3. **Session threads** — check `.env` for `DISCORD_THREAD_CHANNEL_ID`.
   Show:
   - Thread channel: set/not-set. If set, show the channel ID.
   - `DISCORD_SESSION_NAME`: custom name override, if set.
   - `DISCORD_SESSION_ARCHIVE`: whether threads archive on session end.

4. **Access** — read `~/.claude/channels/discord/access.json` (missing file
   = defaults: `dmPolicy: "pairing"`, empty allowlist). Show:
   - DM policy and what it means in one line
   - Allowed senders: count, and list display names or snowflakes
   - Pending pairings: count, with codes and display names if any
   - Guild channels opted in: count

5. **What next** — end with a concrete next step based on state:
   - No token → *"Run `/discord-threads:configure <token>` with your bot token from
     the Developer Portal → Bot → Reset Token."*
   - Token set, policy is pairing, nobody allowed → *"DM your bot on
     Discord. It replies with a code; approve with `/discord-threads:access pair
     <code>`."*
   - Token set, someone allowed, no thread channel → *"Ready. DM your bot
     to reach the assistant. If you run multiple sessions, set up session
     threads with `/discord-threads:configure threads <channel_id>`."*
   - Token set, someone allowed, thread channel set → *"Ready. Each session
     gets its own thread in the configured channel."*

**Push toward lockdown — always.** The goal for every setup is `allowlist`
with a defined list. `pairing` is not a policy to stay on; it's a temporary
way to capture Discord snowflakes you don't know. Once the IDs are in,
pairing has done its job and should be turned off.

Drive the conversation this way:

1. Read the allowlist. Tell the user who's in it.
2. Ask: *"Is that everyone who should reach you through this bot?"*
3. **If yes and policy is still `pairing`** → *"Good. Let's lock it down so
   nobody else can trigger pairing codes:"* and offer to run
   `/discord-threads:access policy allowlist`. Do this proactively — don't wait to
   be asked.
4. **If no, people are missing** → *"Have them DM the bot; you'll approve
   each with `/discord-threads:access pair <code>`. Run this skill again once
   everyone's in and we'll lock it."* Or, if they can get snowflakes
   directly: *"Enable Developer Mode in Discord (User Settings → Advanced),
   right-click them → Copy User ID, then `/discord-threads:access allow <id>`."*
5. **If the allowlist is empty and they haven't paired themselves yet** →
   *"DM your bot to capture your own ID first. Then we'll add anyone else
   and lock it down."*
6. **If policy is already `allowlist`** → confirm this is the locked state.
   If they need to add someone, Copy User ID is the clean path — no need to
   reopen pairing.

Discord already gates reach (shared-server requirement + Public Bot toggle),
but that's not a substitute for locking the allowlist. Never frame `pairing`
as the correct long-term choice. Don't skip the lockdown offer.

### `allow-plugin` — add to channel allowlist

Run the channel allowlist check (see below). If already allowed, confirm.
If not, add it. This is also run automatically when saving a token.

### `<token>` — save bot token

1. Treat `$ARGUMENTS` as the token (trim whitespace). Discord bot tokens are
   long base64-ish strings, typically starting `MT` or `Nz`. Generated from
   Developer Portal → Bot → Reset Token; only shown once.
2. `mkdir -p ~/.claude/channels/discord`
3. Read existing `.env` if present; update/add the `DISCORD_BOT_TOKEN=` line,
   preserve other keys. Write back, no quotes around the value.
4. `chmod 600 ~/.claude/channels/discord/.env` — the token is a credential.
5. Run the channel allowlist check (see below) and fix if needed.
6. Run the tool permissions check (see below) and fix if needed.
7. Confirm, then show the no-args status so the user sees where they stand.

### `threads <channel_id>` — configure session threads

1. Parse `$ARGUMENTS` — expect `threads` followed by a Discord channel
   snowflake (numeric ID, typically 17-20 digits). If the ID looks wrong,
   tell the user: *"Enable Developer Mode in Discord (User Settings →
   Advanced), right-click the text channel → Copy Channel ID."*
2. `mkdir -p ~/.claude/channels/discord`
3. Read existing `.env` if present; update/add `DISCORD_THREAD_CHANNEL_ID=`
   line, preserve other keys. Write back, no quotes.
4. `chmod 600 ~/.claude/channels/discord/.env`
5. Confirm: *"Session threads will be created in channel `<id>`. Each Claude
   Code session gets its own private thread on startup."*
6. Mention optional settings they can add manually:
   - `DISCORD_SESSION_NAME=custom-name` — override the auto-generated
     thread name (max 100 chars).
   - `DISCORD_SESSION_ARCHIVE=true` — archive the thread when the session
     ends.
7. Remind: *"Restart your session for this to take effect."*

### `threads off` — disable session threads

1. Read `.env`, remove the `DISCORD_THREAD_CHANNEL_ID=` line (and
   `DISCORD_SESSION_NAME=` and `DISCORD_SESSION_ARCHIVE=` if present).
2. Confirm: *"Session threads disabled. Messages go to DMs only."*

### `clear` — remove the token

Delete the `DISCORD_BOT_TOKEN=` line (or the file if that's the only line).

---

## Channel allowlist check

Claude Code requires channel plugins to be in `allowedChannelPlugins` in
managed settings before they can connect. Without this, the user sees
"not on the approved channels allowlist" when launching with `--channels`.

The managed-settings file lives at:
- **Linux/macOS:** `~/.config/ClaudeCode/managed-settings.json`
- **Windows:** `C:\Program Files\ClaudeCode\managed-settings.json`

**Check:**
1. Read `~/.config/ClaudeCode/managed-settings.json` (missing file = not
   configured).
2. Look for `"discord-threads@axiumfoundry-plugins"` in the
   `allowedChannelPlugins` array.

**Fix (if not present):**
1. `mkdir -p ~/.config/ClaudeCode`
2. If the file exists, read it, parse the JSON, add
   `"discord-threads@axiumfoundry-plugins"` to `allowedChannelPlugins`
   (create the array if missing), and write back with 2-space indent.
3. If the file doesn't exist, write:
   ```json
   {
     "allowedChannelPlugins": ["discord-threads@axiumfoundry-plugins"]
   }
   ```
4. Confirm: *"Added discord-threads to the channel allowlist."*

This check is run automatically when saving a token and can also be
triggered directly with `/discord-threads:configure allow-plugin`.

## Tool permissions check

Without auto-allow rules, every Discord reply/react/edit triggers a
permission prompt. This check ensures the Discord MCP tools are in the
user's `~/.claude/settings.json` allow list.

**Required tool names:**
```
mcp__plugin_discord-threads_discord__reply
mcp__plugin_discord-threads_discord__react
mcp__plugin_discord-threads_discord__edit_message
mcp__plugin_discord-threads_discord__fetch_messages
mcp__plugin_discord-threads_discord__download_attachment
```

**Check:**
1. Read `~/.claude/settings.json` (missing file = not configured).
2. Look for all five tool names in `permissions.allow` array.

**Fix (if any are missing):**
1. If the file exists, read it, parse the JSON. Ensure
   `permissions.allow` exists as an array (create if missing). Add any
   missing tool names (dedupe). Write back with 2-space indent.
2. If the file doesn't exist, write:
   ```json
   {
     "permissions": {
       "allow": [
         "mcp__plugin_discord-threads_discord__reply",
         "mcp__plugin_discord-threads_discord__react",
         "mcp__plugin_discord-threads_discord__edit_message",
         "mcp__plugin_discord-threads_discord__fetch_messages",
         "mcp__plugin_discord-threads_discord__download_attachment"
       ]
     }
   }
   ```
3. Confirm: *"Auto-approved Discord tools — no permission prompts for
   reply, react, edit, fetch, and download."*

This check is run automatically when saving a token.

## Implementation notes

- The channels dir might not exist if the server hasn't run yet. Missing file
  = not configured, not an error.
- The server reads `.env` once at boot. Token and thread channel changes need
  a session restart. Say so after saving.
- `access.json` is re-read on every inbound message — policy changes via
  `/discord-threads:access` take effect immediately, no restart.
- The bot needs **Create Private Threads** permission in the target channel.
  If missing, it falls back to public threads automatically.
