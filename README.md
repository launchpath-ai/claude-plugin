# LaunchPath — Claude Code Plugin

Create, train, deploy, and manage AI agents for businesses entirely from Claude Code.

## Installation

### Option 1: Plugin marketplace (recommended)

```
/plugin marketplace add launchpath/claude-plugin
/plugin install launchpath
```

### Option 2: Direct from GitHub

```
/plugin install https://github.com/launchpath/claude-plugin.git
```

You'll be prompted for your LaunchPath API key. Get it from [Settings → API Keys](https://www.trylaunchpath.com/dashboard/settings) in the LaunchPath dashboard.

## What you get

### MCP Tools (50+)

Full access to the LaunchPath platform:
- **Agent management** — create, update, clone, delete agents
- **Knowledge** — scrape websites, manage FAQs, generate Q&A pairs
- **Integrations** — connect 900+ apps (Google, Slack, Stripe, etc.)
- **Deployment** — website widgets, WhatsApp, API channels
- **Clients** — create accounts, link campaigns, invite to portal
- **WhatsApp** — templates, contacts, broadcasts, drip sequences
- **Analytics** — conversations, channel health, performance metrics

### Skills (slash commands)

| Command | Description |
|---------|-------------|
| `/launchpath:deploy-agent` | End-to-end: create agent → train → deploy |
| `/launchpath:setup-client` | Create client account and link an agent |
| `/launchpath:agent-report` | Summary report of all agents and what needs attention |
| `/launchpath:whatsapp-campaign` | Set up WhatsApp templates, contacts, and campaigns |

Plus an internal guide that helps Claude understand how to use LaunchPath tools effectively.

## Quick start

```
# Deploy an agent for a business
/launchpath:deploy-agent https://example.com

# Set up a new client
/launchpath:setup-client Acme Corp

# Check how your agents are doing
/launchpath:agent-report

# Launch a WhatsApp campaign
/launchpath:whatsapp-campaign Spring promotion
```

## Requirements

- [Claude Code](https://claude.com/claude-code) CLI or VS Code extension
- A LaunchPath account with an API key ([sign up](https://www.trylaunchpath.com))
- Node.js 18+ (for the MCP server)

## Links

- [LaunchPath Dashboard](https://www.trylaunchpath.com/dashboard)
- [MCP Server on npm](https://www.npmjs.com/package/@launchpath-ai/mcp-server)
