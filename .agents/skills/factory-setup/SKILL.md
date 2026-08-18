---
name: factory-setup
description: Connect a third-party coding agent (Claude Code, Codex, Cursor, or any other MCP-capable client) to Warp Factory by installing and authenticating the oz CLI and registering the Factory MCP server. Use when someone outside Warp wants to set up Factory MCP access — for example to create or operate a software factory from Claude Code, Codex, Cursor, or another external agent.
---

# Factory Setup

Bootstrap **Warp Factory** access for a coding agent that is not Warp itself: install/authenticate the `oz` CLI, register the hosted **Factory MCP** server, verify the connection, and hand off to the canonical operating skill. This is a one-time setup skill, not the operating playbook.

## Scope

This skill:
1. Detects the target harness (Claude Code, Codex, Cursor, or generic MCP client).
2. Installs/authenticates the CLI and mints an API key.
3. Registers the Factory MCP server for that harness.
4. Verifies the connection (tools visible, `list_factories` works, canonical skill readable).
5. Optionally helps pick or create a factory and save it as the default.
6. Stops and hands off to the server-served operating skill for everything after setup.

This skill does **not**:
- Duplicate the day-2 operating playbook — that lives at `skill://warp/factory-mcp/SKILL.md` on the MCP connection itself and is the authoritative source once connected.
- Automate Slack, Linear, or GitHub integration OAuth for a factory. Point the user at the Factory web app for those; do not attempt to script them here.

## Step 1 — Pick a server root

The Factory MCP is always mounted at `{server_root}/api/v1/mcp/factory`. Third-party harnesses do not know Warp's server root on their own, so make it explicit up front:

| Server root | CLI binary | MCP URL |
| --- | --- | --- |
| `https://app.warp.dev` | `oz` | `https://app.warp.dev/api/v1/mcp/factory` |

- If the machine already sets `FACTORY_MCP_URL`, use it verbatim instead of building one.
- If it sets `WARP_SERVER_ROOT` instead, build the URL as `{WARP_SERVER_ROOT}/api/v1/mcp/factory`.
- Otherwise use `https://app.warp.dev` — do not ask the user to pick a server root, and do not point them at any other host.
- If registering or calling the MCP URL fails with a not-found/disabled-feature style error, tell the user Factory MCP isn't turned on for their account yet and stop — don't guess around it or substitute a different host.

## Step 2 — Get the CLI and an API key

Warp's CLI ships as `oz` through soft launch; it is renamed to `warp` at GA. Mention this once so the user isn't confused if they see the CLI called `warp` elsewhere later.

- **Installing `oz`:** the install command is not finalized yet.
  > **PLACEHOLDER — do not invent a command here.** Ask the user how they installed/would install `oz`, or point them at `https://docs.warp.dev` for the current instructions. Do not guess a package name, curl-pipe URL, or script.

Once the CLI is available and logged in (`oz login`), mint an API key for the harness to use:

```bash
oz api-key create --expires-in 30d "factory-setup"
```

Treat the printed key as a secret: export it to an environment variable (e.g. `WARP_API_KEY`) and use that variable everywhere below. Never paste it into a config file that gets committed, and never print it back to a shared terminal or log.

API key + `Authorization: Bearer` is the supported auth path and works on every harness. Use it everywhere below. OAuth is not available for third-party clients today: Factory MCP does not support dynamic client registration, and no per-harness OAuth client is registered for external use — so a client's built-in "connect with OAuth" flow will fail. If a harness tries OAuth on its own and fails, fall back to the API key rather than debugging the OAuth flow.

## Step 3 — Detect the harness and register the MCP server

Substitute `{mcp_url}` = `{server_root}/api/v1/mcp/factory` from Step 1, and `$WARP_API_KEY` from Step 2.

### Claude Code

```bash
claude mcp add --transport http warp-factory {mcp_url} \
  --header "Authorization: Bearer $WARP_API_KEY"
```

Flag names can shift between Claude Code versions — run `claude mcp add --help` first if this fails. Verified against `@anthropic-ai/claude-code` (`mcp add --transport http ... --header "..."`) as of this writing.

Do not use `claude mcp login` / `--client-id` here — see the OAuth note in Step 2.

### Codex

```bash
codex mcp add warp-factory --url {mcp_url} --bearer-token-env-var WARP_API_KEY
```

This writes a `[mcp_servers.warp-factory]` block to `~/.codex/config.toml` with `url` and `bearer_token_env_var`; Codex reads the bearer token from that environment variable at connect time, so make sure `WARP_API_KEY` is exported in the same shell/session Codex runs in. Verified against `@openai/codex` (`mcp add <name> --url ... --bearer-token-env-var ...`).

Do not use `--oauth-client-id` / `codex mcp login` here — see the OAuth note in Step 2.

### Cursor

Cursor has no MCP CLI; edit `~/.cursor/mcp.json` (global) or `.cursor/mcp.json` (project-scoped):

```json
{
  "mcpServers": {
    "warp-factory": {
      "url": "{mcp_url}",
      "headers": {
        "Authorization": "Bearer ${env:WARP_API_KEY}"
      }
    }
  }
}
```

Cursor resolves `${env:VAR}` references in `headers` from its own process environment at connect time — never inline the raw key in this file, since a project-scoped `.cursor/mcp.json` is easy to commit or expose. Make sure `WARP_API_KEY` is actually set in Cursor's launch environment: a shell-only export in `.bashrc`/`.zshrc` is not always inherited by a desktop app launched outside that shell, so prefer launching Cursor from a terminal where `echo $WARP_API_KEY` already prints the value, or set it somewhere Cursor's launcher reliably reads (e.g. `/etc/environment` on Linux). Restart Cursor after editing, then check Settings → Tools & MCP for a green connection.

Use the `headers` block above, not an `auth` block: Factory MCP has no dynamic client registration, so Cursor's discovery/DCR fallback will fail — see the OAuth note in Step 2.

### Generic / any other MCP-capable client

Any client that supports streamable HTTP with custom headers can connect directly: point it at `{mcp_url}` with header `Authorization: Bearer $WARP_API_KEY`. There is no special Warp-specific transport.

## Step 4 — Verify

From the harness with the MCP server attached:

1. List tools (`tools/list` or the client's equivalent) and confirm the Factory tools are present: `list_factories`, `create_factory`, `list_tasks`, `search_task`, `get_task`, `message_foreman`, `get_conversation`, `send_task`, `complete_task`, `list_notification_routes`.
2. Call `list_factories`. Success is any response without an auth error — an empty list just means the principal has no factories yet, which is fine.
3. Read the resource `skill://warp/factory-mcp/SKILL.md`. A successful read proves the handoff to the operating skill works. Day-2 Factory use depends on this operating skill, so if the harness can't do `resources/read` itself (some tool-only clients don't), don't just tell the user to "fetch it manually" — fetch it yourself with one standalone JSON-RPC call against the same authenticated endpoint (verified: this works without a prior `initialize` handshake) and put the returned text somewhere the agent will read it before starting real Factory work:

   ```bash
   curl -s {mcp_url} \
     -H "Authorization: Bearer $WARP_API_KEY" \
     -H "Content-Type: application/json" \
     -H "Accept: application/json, text/event-stream" \
     -d '{"jsonrpc":"2.0","id":1,"method":"resources/read","params":{"uri":"skill://warp/factory-mcp/SKILL.md"}}'
   ```

   The skill text is in `result.contents[0].text` of the returned JSON.

If any step fails, stop and report the exact error instead of guessing a fix — see Troubleshooting below.

## Step 5 — Optional: pick or create a factory

Setup is complete once Step 4 passes; this step is optional follow-up, not the success bar.

- If the user already has a factory, use `list_factories` to find its `factory_uid` and, if they want, save it as their default per the operating skill's `~/.warp/factory/config.json` contract (`skill://warp/factory-mcp/references/factory-config.md`).
- If they want a new factory, `create_factory` needs `team_uid`, `name`, `code_forge`, and at least one `repositories` entry (`owner/repo`); `integrations` is optional. Avatars are REST-only, not settable over MCP.
- Connecting Slack, Linear, GitHub, or Jira integrations for a factory is out of scope here — send the user to the Factory web app for that; do not try to script integration OAuth from this skill.

## Step 6 — Hand off and stop

Once verified, tell the user setup is done. Day-2 operation — finding tasks, delegating work, pulling a task down locally, coordinating with the foreman, handing work back — is governed entirely by the canonical skill the server exposes at `skill://warp/factory-mcp/SKILL.md`. Read and follow that skill for anything after this point; do not reimplement its operating rules here.

## Troubleshooting

- **404 / feature-disabled-looking error at the MCP URL**: Factory MCP is not enabled for the user's account yet. Report that and stop; it is not a skill bug, and no other host is a valid substitute.
- **401 Unauthorized**: the API key is missing, wrong, or expired. Recreate it (Step 2) and re-register.
- **OAuth client not found / OAuth flow rejected / DCR failure**: expected — Factory MCP has no OAuth client registered for third-party harnesses. Use the API key path from Step 2 instead of debugging the flow.
- **`resources/read` for the skill URI fails or is unsupported**: some clients don't implement MCP resources. The tool surface still works, but fetch the canonical skill yourself with the standalone `curl` call in Step 4 rather than leaving the user without it.
