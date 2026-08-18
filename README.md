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


## License

MIT. See [LICENSE](LICENSE). The license covers the contents of this repository, not the Rippit service, which is governed by its own [terms of service](https://rippit.com/terms-of-service) and [privacy policy](https://rippit.com/privacy-policy).
