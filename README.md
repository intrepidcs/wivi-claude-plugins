# WiVi plugins for Claude Code

A [Claude Code](https://claude.com/claude-code) plugin marketplace for WiVi
(Wireless neoVI). The `wivi` plugin connects Claude Code to one WiVi server, so
you can ask questions about your vehicle data and build Grafana dashboards from
it.

The plugin adds three MCP servers and the `wivi-grafana-dashboards` skill. It
asks for your WiVi hostname once.

## Requirements

- Claude Code installed.
- A WiVi account on your server.
- A WiVi server with the MCP services enabled. Ask your WiVi administrator if
  you are not sure.

## Install

```sh
claude plugin marketplace add intrepidcs/wivi-claude-plugins
claude plugin install wivi@wivi --config host=<wivi hostname>
```

`<wivi hostname>` is the hostname only, without `https://` or a path, for
example `myserver.wirelessneovi.com`.

## Sign in

1. Start `claude` and run `/mcp`.
2. Authenticate `wivi-analytics`. Your browser opens the WiVi login page.
3. Authenticate `wivi-api` and `wivi-grafana`. These reuse the same login and do
   not ask for a password again.

To check that it works, ask Claude: "list my fleets".

## Example

> Build a Grafana dashboard with the GPS track and vehicle speed for vehicle
> \<name\> for the last 7 days.

Claude finds the vehicle and its signals, shows you the planned panels and waits
for your approval. It then saves the dashboard as a draft in the **AI Drafts**
folder in Grafana and gives you the link.

## What the plugin adds

| Server | URL | Use |
| --- | --- | --- |
| `wivi-analytics` | `https://<wivi hostname>/analytics-ai/mcp/` | Signals, DTCs, GPS and dashboard recipes |
| `wivi-api` | `https://<wivi hostname>/wivi-api-ai/mcp/` | Organizations, fleets and vehicles |
| `wivi-grafana` | `https://<wivi hostname>/grafana-ai/mcp/` | Grafana dashboards, folders and datasources |

The `wivi-grafana-dashboards` skill tells Claude how to build dashboards from
tested recipes instead of writing panels or queries by hand.

## Security

- This repository holds no credentials. It contains only the server paths, the
  public OAuth client ID and the skill text.
- You sign in with your own WiVi login in the browser (OAuth with PKCE). Claude
  Code stores the resulting token on your computer.
- What you can read is decided by your WiVi account on the server, not by this
  plugin.
- In Grafana, the plugin can save only to the **AI Drafts** folder.

## Update or reconfigure

| Task | Command |
| --- | --- |
| Get the latest plugin version | `claude plugin marketplace update wivi`, then restart `claude` |
| Change the WiVi hostname | `claude plugin configure wivi@wivi` |

## Troubleshooting

| Problem | What to do |
| --- | --- |
| Wrong hostname | Run `claude plugin configure wivi@wivi`. |
| Plugin is out of date | Run `claude plugin marketplace update wivi` and restart `claude`. |
| A server shows "failed" in `/mcp` | Open `https://<wivi hostname>/.well-known/oauth-protected-resource/analytics-ai/mcp/` in your browser. If it does not return JSON, the MCP services are not enabled or not reachable on that server; contact your WiVi administrator. |

## Changing the plugin

The plugin is in `plugins/wivi/`. Check changes before committing
(Claude Code 2.1.289 or later):

```sh
claude plugin validate .
```
