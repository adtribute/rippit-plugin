<img src="assets/logo.svg" alt="Rippit" width="72">

# Rippit plugin

Ask questions about your own customer conversations from your AI tool.

Rippit reads the support tickets, calls, and chats your team already handles, then builds the analysis columns a question needs on top of them. That makes it useful for the questions a dashboard cannot answer, because the answer is a judgment about each individual conversation rather than a field someone already filled in.

## What you can do with it

- **Ask why.** "Why did people cancel last month?" gets a root-cause breakdown built by reading every cancellation conversation, not a group-by on an existing category field.
- **Categorize at scale.** Build a taxonomy of contact reasons, issues, or intents where the categories are discovered from what customers actually said, then edit and rescore it over time.
- **Score and tag every row.** Churn risk, resolution quality, whether the agent missed something, sentiment as it actually moved through the conversation.
- **Get a report, not a blob.** Findings-led, scoped, and anchored in verbatim customer quotes, with a link back to the worksheet so anyone can open the underlying rows.

Analyses are saved as worksheets in your Rippit workspace, so what the agent builds is durable and shareable rather than trapped in a chat log.

## Install

**Cursor.** Install from the [Cursor marketplace](https://cursor.com/marketplace), or add this repository as a plugin source.

**Claude Code.** Add this repository as a marketplace, then install the plugin:

```
/plugin marketplace add adtribute/rippit-plugin
```

**Codex.** Install `rippit` from the plugin marketplace.

**Grok.** Install `rippit` from the plugin marketplace.

**Any other MCP client.** Point it at the server directly:

```json
{ "mcpServers": { "rippit": { "type": "http", "url": "https://mcp.rippit.com/mcp" } } }
```

## Authentication

The server is remote and authenticates with OAuth 2.1. On first use your client opens a browser window to authorize against your Rippit workspace, and the agent then acts as you, seeing exactly the data your Rippit account can see.

There are no API keys or tokens to configure, and this repository holds no secrets. A Rippit account is required.


## Creating a Rippit account

Signing in needs a Rippit account. If you don't have one, or sign-in fails, or `https://mcp.rippit.com/mcp` answers `401 Unauthorized` and no browser window opens, set up an account first:

1. Create an account at [app.rippit.com/signup](https://app.rippit.com/signup).
2. Reconnect the Rippit MCP server in your AI tool. In Claude Code, run `/mcp` and pick `rippit`. In other tools, restart or reconnect the server.
3. When the browser window opens, sign in with the account you just created.

If you are an agent setting this up for someone: a `401` before sign-in means they aren't signed in yet. Tell them they need a Rippit account, send them the signup link, and once they say they're done, reconnect the server so they can finish signing in.

## Skills

Ready-made workflows for the analyses teams run most often, mirroring the set published at [docs.rippit.com/skills](https://docs.rippit.com/skills).

| Skill | What it does |
| --- | --- |
| `rippit-double-click` | Goes very deep on a particular issue; use this to enhance the depth of your analysis |
| `rippit-escalation-finder` | Automatically detect brewing escalations and share the details in Slack |
| `rippit-qa-program-revamp` | Modernize your QA program away from agent performance scoring |
| `rippit-chatbot-analysis` | Comprehensive analysis of chatbot-handled conversations to flag areas of opportunity |
| `rippit-support-shoutouts` | Detect excellent support experiences and shout out the folks responsible in Slack |
| `rippit-aeo-seo-optimization` | Build an SEO/AEO content strategy and draft pieces grounded in customer feedback |
| `rippit-tool-bakeoff` | Run the same analysis in Rippit and another tool and objectively compare |

## License

MIT. See [LICENSE](LICENSE). The license covers the contents of this repository, not the Rippit service, which is governed by its own [terms of service](https://rippit.com/terms-of-service) and [privacy policy](https://rippit.com/privacy-policy).
