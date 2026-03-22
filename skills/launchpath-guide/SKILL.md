---
name: launchpath-guide
description: Internal guide for using LaunchPath MCP tools to manage AI agents, clients, and deployments.
user-invocable: false
---

# LaunchPath Platform Guide

LaunchPath is an AI agent deployment platform. You have access to it via MCP tools prefixed with `mcp__launchpath__`.

## What You Can Do

- **Create AI agents** — write the system prompt yourself, choose a model and language
- **Train agents** with website content (auto-scraped) and FAQ pairs
- **Add integrations** — connect 900+ apps (Google Calendar, Gmail, Slack, Stripe, etc.)
- **Deploy agents** to website widgets, WhatsApp, or API endpoints
- **Manage clients** — create client accounts and link agents as campaigns
- **Test agents** via chat to verify quality before going live
- **Monitor** conversations, analytics, and channel health

## Common Workflows

### Build an agent for a new client
1. `create_agent` — write the system prompt with role, personality, tone, greeting, and tool instructions
2. `discover_pages` → `scrape_website` — add the client's website content
3. `generate_faqs` — auto-create Q&A pairs from scraped content
4. `browse_integrations` → `connect_app` → `add_agent_tool` — connect integrations
5. `create_client` → `create_campaign` — link the agent to the client
6. `configure_widget` or set up WhatsApp — customize the deployment
7. `get_embed_code` — get the HTML to paste on their site
8. `chat_with_agent` — test it works correctly (only when asked)

### Add integrations to an agent
1. `browse_integrations(search: "app name")` — find the app
2. `list_app_actions(toolkit)` — see available actions
3. `check_connections()` — check if already connected
4. If not connected: `connect_app(toolkit, toolkit_name)`
   - **API key apps** (Stripe, OpenAI): call without credentials first to discover required fields, then call again with credentials — connects instantly
   - **OAuth apps** (Google, Slack): returns a browser URL for the user to authorize, then `poll_connection` to verify
5. `add_agent_tool(tool_type: "composio", config: { toolkit, toolkit_name, enabled_actions })` — attach to agent
6. `update_agent` — add tool usage instructions to the system prompt

### Deploy via API channel (for developers)
1. `create_api_channel(agent_id)` — get endpoint URL + bearer token
2. Endpoint: `POST /api/channels/{agentId}/chat` with `Authorization: Bearer <token>`
3. Request body: `{ userMessage, sessionId }` (stateful) or `{ userMessage, messages }` (stateless)
4. Optional: `visitorInfo: { name, email }` for lead capture, `attachment` for file uploads
5. Response: SSE stream with `text-delta`, `tool-call`, `tool-result`, `done`, `error` events
6. `list_channels(agent_id)` — view existing channels

### Update an existing agent
1. `list_agents` — find the agent
2. `get_agent` — read current config
3. `update_agent` — change system prompt, model, language, etc.
4. `chat_with_agent` — test the changes (only when asked)

### Check on your business
1. `list_agents` — see all agents and their status
2. `list_clients` — see all clients
3. `list_conversations(campaign_id)` — check recent activity
4. `get_agent_analytics(agent_id)` — view performance metrics
5. `list_channels(agent_id)` — see deployed channels
6. `get_channel_health(campaign_id)` — check channel status

## Tips

- Always scrape the client's website before going live — it dramatically improves agent quality
- Put everything in the system prompt: personality, tone, greeting, tool instructions
- Deploy as widget first — it's the fastest path to a working agent
- Create the client and campaign before deploying so conversations are properly tracked
- Use `discover_pages` before `scrape_website` for comprehensive coverage
- Always `check_connections()` before adding integrations — tools fail at runtime without a connection
- The default model (openai/gpt-4o-mini) is the most cost-effective. Upgrade to claude-sonnet for higher quality.
- Use `list_channels` to see all channels for an agent before creating new ones
