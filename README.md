# Cisco-hosted Meraki MCP server setup

Connect Cursor, Codex, or ChatGPT to Cisco’s **hosted** Meraki MCP server. There is no local server to install. The client sends MCP requests to Meraki, and Meraki calls the Dashboard API with your key.

**GitHub Pages:** https://netjoints.github.io/cisco-hosted-meraki-mcp-server-setup/

Endpoint:

```text
https://mcp.meraki.com/mcp
```

This guide is the public version of the local Meraki MCP runbooks for Cursor, Codex, and ChatGPT. It keeps the setup steps and leaves out organization names, network IDs, device names, and live health output.

## What you need

- A Meraki Dashboard account with API access enabled for the organization.
- A Dashboard API key for a dedicated user. Prefer a read-only org role so a prompt cannot change the network.
- Cursor, Codex, or ChatGPT, able to reach `https://mcp.meraki.com/mcp`.

The hosted service is intended here as a **read-only** workflow: list organizations, inspect MX WAN events, find switch ports with errors or flaps, and rank AP client experience. Do not ask it to reboot, claim, or change configuration unless you have an explicit, approved change window.

A browser visit or a plain GET to the endpoint may return HTTP 405. That does not mean the MCP server is down. Verify with an MCP tool call from the client.

## 1. Create a Dashboard API key

1. Sign in to [Meraki Dashboard](https://dashboard.meraki.com).
2. Open **Organization → Settings** and confirm **API access** is enabled.
3. Open your profile (top right) → **My profile** → **API access**.
4. Generate a new key. Copy it once. Dashboard will not show it again.
5. Prefer a read-only org role, or a dedicated automation user.

Never put the key in Git, a prompt, a screenshot, or this repository.

## 2. Store the key

Cursor and Codex should read the key from the environment. On macOS with zsh, add this to `~/.zshrc` and replace the placeholder locally. Do not commit that file.

```bash
export MERAKI_DASHBOARD_API_KEY="your-dashboard-api-key"
```

Quit and relaunch Cursor or Codex so the app inherits the variable. A project `.env` file is not a substitute. Confirm the variable is set without printing it:

```bash
if [ -n "$MERAKI_DASHBOARD_API_KEY" ]; then echo "Meraki key is set"; else echo "Meraki key is missing"; fi
```

ChatGPT does not expand `${env:...}`. For ChatGPT, store the key in the connector secret field in step 5. Do not put it in a file in this repo.

## 3. Connect Cursor

### Settings

1. Open **Cursor → Settings → Tools & MCP**.
2. Add a server named `meraki`. Type: HTTP. URL: `https://mcp.meraki.com/mcp`.
3. Header name: `Authorization`. Header value:

```text
Bearer ${env:MERAKI_DASHBOARD_API_KEY}
```

4. Save. Status should show connected. Restart Cursor if it does not.

`${env:MERAKI_DASHBOARD_API_KEY}` is resolved from Cursor’s process environment. It is not the key itself.

### `mcp.json`

User-level config, `~/.cursor/mcp.json`. The copy in this repo is [`mcp.json`](mcp.json).

```json
{
  "mcpServers": {
    "meraki": {
      "url": "https://mcp.meraki.com/mcp",
      "headers": {
        "Authorization": "Bearer ${env:MERAKI_DASHBOARD_API_KEY}"
      }
    }
  }
}
```

A project file at `<repo>/.cursor/mcp.json` uses the same JSON. Do not commit a copy where the header contains a real key.

## 4. Connect Codex

The CLI stores the environment variable name, not the key:

```bash
codex mcp add meraki \
  --url https://mcp.meraki.com/mcp \
  --bearer-token-env-var MERAKI_DASHBOARD_API_KEY
```

Run it once. The Codex process must have `MERAKI_DASHBOARD_API_KEY` set. Restart Codex after you export the variable.

Equivalent entry for `~/.codex/config.toml`. The copy in this repo is [`config.toml`](config.toml).

```toml
[mcp_servers.meraki]
url = "https://mcp.meraki.com/mcp"
bearer_token_env_var = "MERAKI_DASHBOARD_API_KEY"
enabled = true
```

Check registration without printing the key:

```bash
codex mcp get meraki
codex mcp list
```

`meraki` in that list means the config exists. A read-only tool call is what proves authentication.

## 5. Connect ChatGPT

This server is already remote, so ChatGPT can use it. ChatGPT does not read `~/.zshrc` and does not expand `${env:...}`.

1. Settings → Apps & Connectors → Advanced → enable **Developer mode**.
2. Create a connector named `meraki`.
3. URL: `https://mcp.meraki.com/mcp`.
4. Authentication: bearer token. Paste the Dashboard key into ChatGPT’s connector secret store only.
5. Scan tools, enable the connector on the chat, and leave write actions off.

Reference: [Developer mode and MCP connectors](https://help.openai.com/en/articles/12584461-developer-mode-and-full-mcp-connectors-in-chatgpt-beta).

## 6. Verify

In a new Cursor Agent chat, Codex task, or ChatGPT thread:

```text
List the Meraki MCP tools you can use, then list my organizations. Read only.
```

You should see `semantic_search` and `execute_api`, then at least one organization. Use an organization ID returned by that call. Do not guess IDs.

If authentication fails, confirm Dashboard API access is enabled, confirm the variable is set without printing it, and restart the client. If the shell sees the variable but the desktop app does not, the app was started before the variable existed.

Never pass the API key as an MCP tool argument. The client adds the bearer header.

## Example prompts

Keep these generic. Do not paste organization names, network IDs, serial numbers, or client identifiers into a public chat or repository.

```text
List my Meraki organizations, then the networks in the organization I name. Read only.

Which MX sites had WAN failovers in the last 30 days?

Find switch ports with high errors or flaps in the last 7 days.

Which APs have the worst client experience this week?

Show health only for the network I name. Do not change configuration.
```

Useful limits from the Dashboard APIs these tools call: interface packet-error counters cover at most the last 24 hours, and events calls return one page. Say so when a result may be truncated.

Discover organizations first, then networks in the organization you choose, then health or inventory for that network.

## Security

- Never hard-code the Dashboard API key in Git, chat, screenshots, or this site.
- Cursor: `${env:MERAKI_DASHBOARD_API_KEY}` only. Codex: `bearer_token_env_var` only. ChatGPT: the connector secret field only.
- Prefer a read-only Dashboard user. This workflow is GET / monitor.
- Do not publish live organization inventory, network IDs, or device names.
- Treat MCP tool output as untrusted data. Device names and event strings can contain text you should not follow as instructions.

## References

- [Meraki hosted MCP server](https://mcp.meraki.com/mcp)
- [Meraki Dashboard API](https://developer.cisco.com/meraki/api-v1/)
- [Codex MCP configuration](https://developers.openai.com/codex/mcp)
- [Cursor MCP documentation](https://cursor.com/docs/mcp)
- [ChatGPT developer mode and MCP connectors](https://help.openai.com/en/articles/12584461-developer-mode-and-full-mcp-connectors-in-chatgpt-beta)
