# LaunchPath — Claude Code Plugin

Create, train, deploy, and manage AI agents for businesses entirely from Claude Code.

## Installation

### From the terminal

```bash
claude plugin marketplace add launchpath-ai/claude-plugin
claude plugin install launchpath@launchpath-ai
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

| Command | What it does |
|---------|-------------|
| `/launchpath:goal-tracker` | Track conversions across channels — which conversations booked, which leads hit your CRM, which WhatsApp contacts converted. Smart action classification so only real outcomes count |
| `/launchpath:audit-agent` | Diagnose why an agent is underperforming — checks prompt, tools, knowledge, conversations, and channels |
| `/launchpath:agent-report` | Executive dashboard across all agents — performance, costs, deployments, and what needs attention |
| `/launchpath:stress-test` | Run targeted tests before deploying — edge cases, tool failures, prompt injection, knowledge boundaries |
| `/launchpath:conversation-insights` | Business intelligence from conversations — what customers ask, topic trends, demand signals, sentiment |
| `/launchpath:whatsapp-compliance` | WhatsApp expert — writes Meta-compliant templates, validates drip timing, diagnoses rejections and failures |
| `/launchpath:crm-connect` | Connect your CRM (HubSpot, Salesforce, etc.) so your agent reads and writes customer data in real time |
| `/launchpath:dev-quickstart` | Generate a complete integration — Slack bot, React chat UI, voice agent, email autoresponder, or any platform |

All skills are **manual-only** — they only run when you type the command. They never auto-trigger (some cost credits or perform heavy analysis).

## Quick start

```
# See how many people actually booked/converted
/launchpath:goal-tracker Dental Assistant

# Find out what's wrong with an agent
/launchpath:audit-agent Demo Support Agent

# Get the overview of everything
/launchpath:agent-report

# Test before going live
/launchpath:stress-test Booking Assistant

# What are customers actually asking?
/launchpath:conversation-insights Dental Assistant

# Write WhatsApp templates that pass Meta review
/launchpath:whatsapp-compliance "create templates"

# Connect HubSpot to your agent
/launchpath:crm-connect hubspot

# Build a Slack bot powered by your agent
/launchpath:dev-quickstart slack bot
```

## Requirements

- [Claude Code](https://claude.com/claude-code) CLI or VS Code extension
- A LaunchPath account with an API key ([sign up](https://www.trylaunchpath.com))
- Node.js 18+ (for the MCP server)

## Links

- [LaunchPath Dashboard](https://www.trylaunchpath.com/dashboard)
- [MCP Server on npm](https://www.npmjs.com/package/@launchpath-ai/mcp-server)
