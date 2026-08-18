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
5. Helps pick or create a factory, including walking the user through authorizing the code forge and choosing and connecting integrations.
6. Stops and hands off to the server-served operating skill for everything after setup.

This skill does **not**:
- Duplicate the day-2 operating playbook — that lives at `skill://warp/factory-mcp/SKILL.md` on the MCP connection itself and is the authoritative source once connected.
- Complete browser consent flows on the user's behalf. Several steps below open a browser to install a GitHub/Slack/Linear app; drive the user through those and verify the result, but never claim a connection succeeded without checking.

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

1. List tools (`tools/list` or the client's equivalent) and confirm the Factory tools are present: `list_factories`, `create_factory`, `list_tasks`, `search_task`, `get_task`, `message_foreman`, `get_conversation`, `send_task`, `complete_task`, `list_notification_routes`, `get_factory_file_schema`, `validate_factory_files`. Treat the live list as authoritative — it grows, so extra tools are fine and only missing ones are a problem.
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

## Step 5 — Pick or create a factory

Connection setup is complete once Step 4 passes. If the user only wanted MCP access, stop there. If they want to create a factory, the rest of this step is required work, not optional polish — a new user cannot create a working factory without authorizing the code forge first.

If the user **already has a factory**, use `list_factories` to find its `factory_uid` and, if they want, save it as their default per the operating skill's `~/.warp/factory/config.json` contract (`skill://warp/factory-mcp/references/factory-config.md`). Nothing below is needed.

### 5a — Authorize the code forge (required, and the usual failure)

`create_factory` validates repository access up front. Without the Warp GitHub app installed and granted on the target repositories, every attempt fails:

```
Failed to create factory: GitHub repositories not found or inaccessible: owner/repo
```

This is the single most likely blocker for a new user, and it is **not** the `integrations` field — the code forge is configured through `code_forge` plus repository access. Warp prompts to install or update its GitHub app, in a browser flow, when the user creates an environment or an integration. So authorization is a side effect of 5b/5c rather than a standalone command.

If the message above appears, do not retry blindly and do not conclude the repo name is wrong. Check the obvious causes in order: the repository really doesn't exist or is misspelled; the Warp GitHub app isn't installed on that account or org; or it is installed but that specific repository wasn't granted (common with "only select repositories"). Then send the user to complete or update the installation, wait, and retry.

### 5b — Environment

Factory agents run in a cloud environment. Omit `default_environment` from `create_factory` and a managed one is auto-created from the factory name and repositories — the simplest path when the user wants no integrations.

Watch the ordering, though: `oz integration create` in 5c requires an existing `--environment` UID, and the auto-created one doesn't exist until the factory does. **If the user wants any integrations, create the environment explicitly first**, use its UID for both the integrations and `default_environment`. Otherwise you'd have to create the factory, then wire integrations against an environment that only appeared afterward.

To create one explicitly:

```bash
oz environment create --name <name> --docker-image <image> --repo <owner/repo> --setup-command "<command>"
```

`oz environment list` shows existing environments and their UIDs. Creating an environment is also what triggers the GitHub app prompt from 5a for a user who has never authorized it.

### 5c — Choose and connect integrations

Integrations are how work reaches the factory from outside — a Linear issue assigned to an agent, or tagging the agent in a Slack thread. Walk the user through this rather than sending them away.

Only **`slack`** and **`linear`** are valid entries in the `integrations` array. `github` is rejected (`integrations contains unsupported provider "github"`) because the code forge is not an integration in this sense.

First show the user where they stand:

```bash
oz integration list
```

Each provider reports one of three states, and they mean different things:

| Status | Meaning | What's needed |
| --- | --- | --- |
| `This integration is not connected.` | no workspace grant | connect, then configure |
| `Connection is active, but the agent integration has not been configured yet.` | workspace granted, no agent wired | configure only |
| configured (shows an environment and timestamps) | ready | nothing |

Ask which integrations the user wants, presenting both options and letting them decline — an empty `integrations` array **is** accepted, and creation will succeed without any. Be honest about the tradeoff instead of implying it's mandatory: with none, the factory has no external intake channel, so work can only be sent through the MCP tools and the web app. That's a legitimate choice for a test factory.

For each provider they pick:

```bash
oz integration create slack --environment <ENV_UID>
oz integration create linear --environment <ENV_UID>
```

This links the integration to the team and environment, opens a browser flow to install the app into their Slack or Linear workspace, and generates an integration ID. The browser step is the user's to complete — say so, wait for them, then re-run `oz integration list` and confirm the status actually changed before moving on. A provider can be connected at the workspace level yet still unusable for notifications; `list_notification_routes` reports `connected: true` alongside an `unavailable_reason` such as `identity_not_bound`, and an empty `routes` array. Report that honestly rather than calling it done.

### 5d — Create the factory

```
create_factory(
  team_uid,        # from `oz whoami`
  name,
  code_forge,      # GITHUB or GITLAB
  repositories,    # ["owner/repo", ...]
  integrations,    # ["slack"], ["linear"], both, or []
)
```

All five are required by the schema; `repositories` and `integrations` are nullable, so `[]` is a valid value for integrations. Optional: `description`, `alias` (unique per workspace, case-insensitive), `default_environment`, `default_model`. Avatars are REST-only, not settable over MCP.

Afterward, offer to save the new `factory_uid` as the user's default per the config contract referenced above.

## Step 6 — Hand off and stop

Once verified, tell the user setup is done. Day-2 operation — finding tasks, delegating work, pulling a task down locally, coordinating with the foreman, handing work back — is governed entirely by the canonical skill the server exposes at `skill://warp/factory-mcp/SKILL.md`. Read and follow that skill for anything after this point; do not reimplement its operating rules here.

## Troubleshooting

- **404 / feature-disabled-looking error at the MCP URL**: Factory MCP is not enabled for the user's account yet. Report that and stop; it is not a skill bug, and no other host is a valid substitute.
- **401 Unauthorized**: the API key is missing, wrong, or expired. Recreate it (Step 2) and re-register.
- **OAuth client not found / OAuth flow rejected / DCR failure**: expected — Factory MCP has no OAuth client registered for third-party harnesses. Use the API key path from Step 2 instead of debugging the flow.
- **`resources/read` for the skill URI fails or is unsupported**: some clients don't implement MCP resources. The tool surface still works, but fetch the canonical skill yourself with the standalone `curl` call in Step 4 rather than leaving the user without it.
