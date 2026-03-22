---
name: agent-report
description: Pull a summary of all your LaunchPath agents — status, conversations, deployments, and what needs attention.
user-invocable: true
allowed-tools:
  - mcp__launchpath__*
context: fork
---

You are generating a report on the user's LaunchPath agents. Follow this workflow:

## Step 1: Gather data

1. Call `list_agents` to get all agents
2. Call `list_campaigns` to see deployment status across all agents
3. For agents that are deployed, call `list_conversations(campaign_id)` to see recent activity
4. Call `get_agent_analytics(agent_id)` for each agent with recent conversations
5. For deployed agents, call `list_channels(agent_id)` to see channel details

## Step 2: Build the report

Present a structured report with these sections:

### Overview
- Total agents (active vs draft vs archived)
- Total deployed campaigns (widget, WhatsApp, API)
- Total conversations in the last 7 days

### Agent Status Table
For each agent, show:
| Agent | Model | Campaigns | Channels | Knowledge Docs | Recent Conversations |

### Needs Attention
Flag any issues:
- Agents that are **draft with no recent edits** (may be abandoned)
- Agents that are **deployed but have zero conversations** (may need promotion)
- Agents with **no knowledge base** (will give poor responses)
- Agents with **high human takeover rates** (may need prompt improvements)
- Channels that are **unhealthy** (check via `get_channel_health` if issues suspected)

### Recommendations
Suggest concrete next steps:
- Which agents to activate or archive
- Which agents need more knowledge
- Which agents should be tested
- Growth opportunities (new channels, new clients)

Keep the report concise and actionable. Use tables and bullet points, not paragraphs.
