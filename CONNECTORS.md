# Connectors — NetSuite MCP setup

This plugin requires your **NetSuite MCP server to be connected in your Claude Cowork session** before any of its skills will work. Netgain-authored NetCash MCP tools (`getNetCashBankActivity`, `submitNetCashManualBankMatch`, `getNetCashAutomationRules`, etc.) are registered within your NetSuite MCP server alongside generic NetSuite tools — once the NetSuite MCP is connected, all NetCash tools become available automatically.

The plugin tells Cowork it needs the NetSuite MCP server. It does not embed a URL or credentials — Cowork manages those per-tenant via the Connectors flow.

## Prerequisites checklist

Before installing this plugin:

- [ ] You have a Claude Cowork team or enterprise plan
- [ ] Your organization has NetSuite with the NetCash SuiteApp installed
- [ ] You authenticate to NetSuite with a role that has MCP access and is configured with permissions for the NetCash records you need. Netgain ships two roles as starting templates:
  - **NetCash MCP View** — read-only access, suitable for dashboards, reports, and troubleshooting investigation.
  - **NetCash MCP Edit** — read + write access, required for the `matching` and `automation-rules` skills which submit matches and create rules.
  These ship as templates - your NetSuite admin should clone and customize them based on how your team will use NetCash in Cowork. If you're on OneWorld with multiple subsidiaries, your admin needs to add the **Subsidiaries** list permission to the cloned role; the templates ship without it because the Subsidiaries list permission causes errors on non-OneWorld accounts. Alternatively, your admin can build a custom role from scratch with equivalent MCP and NetCash record permissions. A role without MCP access will not surface NetCash tools in Cowork at all.
- [ ] You can authenticate into NetSuite from Cowork — the NetSuite MCP uses per-session authentication

## Connecting the NetSuite MCP server

The exact steps depend on how your organization has set up the NetSuite MCP integration in Cowork. The general flow:

1. Open Claude Cowork.
2. Navigate to **Settings → Connectors** (or the equivalent in your Cowork installation).
3. Find **NetSuite** in the connector list and click **Connect**.
4. Provide your NetSuite tenant URL when prompted. The URL is per-tenant — if you don't know it, your NetSuite admin can provide it.
5. Authenticate into NetSuite. Cowork redirects you to a NetSuite login flow; you authenticate with your NetSuite credentials.
6. Once authenticated, the NetSuite MCP server is connected for your Cowork session.

After authentication, all NetSuite + NetCash MCP tools become available. The plugin's skills will route to NetCash-specific tools (those with "NetCash" in the name) wherever possible.

## Verifying the connection

To confirm the NetSuite MCP is connected and NetCash tools are visible, ask Claude in any Cowork conversation:

> "What NetCash tools are available in this session?"

You should see a list of ~30 NetCash-specific tools (e.g., `getNetCashBankActivity`, `getNetCashReconciliations`, `submitNetCashManualBankMatch`, `getNetCashAutomationRuleFieldOptions`). If the list comes back empty or with only generic NetSuite tools, the NetCash custom records are not registered with your NetSuite MCP — contact Netgain Support.

## Per-session authentication

The NetSuite MCP requires authentication **per Cowork session**. When you start a new Cowork session, you may need to re-authenticate into NetSuite. The plugin's skills do not handle authentication — Cowork's connector flow does.

If a skill calls a NetCash tool and gets an authentication error, Cowork will surface the error and prompt you to re-authenticate. Once re-authenticated, you can re-run the failed command.

## Common issues

### "Tool not found: getNetCashBankActivity"

The NetSuite MCP is not connected, or the connection has lost its session. Reconnect via Settings → Connectors and re-authenticate.

### Subsidiary breakdown looks empty or redundant on a single-subsidiary tenant

Some NetCash views (cash position, recon status) include a subsidiary breakdown that's only meaningful on OneWorld instances. The underlying tools (including `getNetCashReportMatches`) still execute and return data on single-subsidiary tenants — it's just that grouping by subsidiary collapses to one bucket. The `recon-dashboards` and `troubleshooting` skills detect single-subsidiary tenants automatically and drop the subsidiary column from output. If you see redundant subsidiary rows or want the column gone, mention it to Claude and the skill will adjust.

### "No NetCash tools visible"

Either the NetSuite MCP is not connected, or NetCash is not installed in your NetSuite instance, or the role you authenticated with does not have MCP access or the right NetCash record permissions. Confirm with your NetSuite admin that your role has MCP access and is configured for the NetCash records you're trying to use. **NetCash MCP View** and **NetCash MCP Edit** are starting templates - your admin should have cloned and customized one for your use case, and added the Subsidiaries list permission if you're on OneWorld with multiple subsidiaries. A custom role with equivalent permissions also works. Contact Netgain Support if the role configuration is correct but tools still aren't visible.

### "Outdated tool surface"

If new NetCash features ship and the plugin's references go stale, ask Claude:

> "Run `getNetCashTables` and `getNetCashTablesAndFields` to refresh the schema you're using."

The plugin's `troubleshooting` and `ad-hoc-reporting` skills both reference these introspection tools, so the schema is discovered at runtime rather than hardcoded.

## Interactive panels (MCP Apps)

Some NetCash MCP tools return interactive panels — searchable, filterable transaction views, report apps, and similar — rendered inline in your Cowork chat. When a panel renders, you can interact with it directly (type in its search box, click filters, change dropdowns) and the panel calls back to NetSuite for fresh data on its own. You do not need to re-prompt Claude for filter changes that the panel already supports.

This is Anthropic's [MCP Apps](https://modelcontextprotocol.io/extensions/apps/overview) capability. Support is tied to the host (Claude.ai web and Cowork render apps inline; some other hosts fall back to text). If you don't see the panel render and instead see plain data, your host doesn't support MCP Apps yet — the underlying tool calls still work and you can keep going with chat.

## Optional: workbook export (Excel / PowerPoint)

Two of this plugin's skills (`recon-dashboards` and `ad-hoc-reporting`) hand off to Anthropic's pre-built `xlsx-author` and `pptx-author` skills when the user asks for a workbook or slide-deck export of a result. These are not part of this plugin — they're Anthropic-distributed plugins that need to be installed separately in your Cowork workspace if you want the file-export path.

If those skills aren't installed, in-conversation table output still works; only the explicit "send me a file" / "as a workbook" handoff fails. To enable file export, install the Anthropic skills via your Cowork connector / plugin manager.

## What if I don't have the NetSuite MCP yet?

If your organization hasn't set up the NetSuite MCP integration in Cowork, this plugin won't work — the skills will fail with "tool not found" errors when they try to call NetCash tools.

To set up the NetSuite MCP:

1. Confirm with your NetSuite administrator that the MCP integration is available for your tenant
2. Have your administrator provide the NetSuite MCP URL
3. Configure the connector in Cowork as described above

Setting up the NetSuite MCP for the first time is outside the scope of this plugin — see Netgain documentation or contact Netgain Support if you need help with that step.
