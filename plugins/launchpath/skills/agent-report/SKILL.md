---
name: agent-report
description: Get a full overview of all your LaunchPath agents — performance, deployments, costs, and what needs attention. The executive dashboard in one command.
user-invocable: true
disable-model-invocation: false
allowed-tools:
  - mcp__launchpath__*
context: fork
---

You are generating an executive report across ALL of the user's LaunchPath agents. This is the big-picture view — not a deep dive into one agent (that's what audit-agent is for).

## Step 1: Gather data

1. `list_agents` — get all agents with their models and counts
2. `list_campaigns` — get all campaigns to see which agents are deployed and where
3. `list_clients` — get all client accounts
4. For each campaign that has active channels: `get_channel_health(campaign_id)` — check health status
5. For the top 5 most active agents (by channel count or recent activity): `list_conversations(campaign_id, limit: 10)` — sample recent conversations
6. For those sampled conversations (up to 5 per agent): `get_conversation(conversation_id)` — read transcripts for outcome analysis

**Be efficient.** If there are 20 agents, don't read conversations for all of them. Focus on deployed agents with active campaigns.

## Step 2: Build the overview

### Fleet Summary
```
Total Agents:      [N]
  Deployed:        [N] (with active campaigns)
  Undeployed:      [N] (no campaigns or all paused)

Total Campaigns:   [N]
  Widget:          [N]
  WhatsApp:        [N]
  API:             [N]

Total Clients:     [N]
```

### Agent Status Table

| Agent | Model | Knowledge | Tools | Channels | Health | Outcome Rate |
|-------|-------|-----------|-------|----------|--------|-------------|
| Dental Assistant | gpt-4o-mini | 12 docs | 2 | widget (healthy) | Healthy | 72% converted |
| Demo Support | gpt-4o-mini | 1 doc | 1 | widget + API | Degraded | 45% converted |

For each agent:
- **Model**: Note if an expensive model is being used where a cheaper one might suffice
- **Knowledge**: Doc count. Flag if 0 docs + knowledge_enabled
- **Tools**: Count. Flag if tools exist but might not be referenced in prompt
- **Channels**: List channel types and count. Flag unhealthy channels
- **Outcome Rate**: From conversation sampling — what % of conversations show tool success or natural completion? If no conversations exist, show "No data"

### Cost Analysis
For each agent, note the model and estimate relative cost:
- `gpt-4o-mini` = lowest cost (0.07x multiplier)
- `gpt-4o` = moderate (0.5x)
- `claude-sonnet` = high (1x)
- `claude-opus` = highest

Flag mismatches: "Agent X uses claude-sonnet but handles simple FAQ questions. Switching to gpt-4o-mini would reduce costs significantly with minimal quality impact."

### Channel Health Matrix

| Campaign | Channel | Status | Issues |
|----------|---------|--------|--------|
| Client A - Widget | widget | Healthy | None |
| Client B - WhatsApp | whatsapp | Degraded | Template rejected |

Flag any degraded or critical channels prominently.

## Step 3: Identify what needs attention

### Critical Issues (fix immediately)
- Channels with auth errors (effectively dead)
- Agents deployed with 0 knowledge docs and knowledge_enabled = true (will give empty responses)
- Tool failures across multiple conversations

### Needs Improvement
- Agents with low outcome rates (below 50%)
- High abandonment rates
- Stale knowledge (docs older than 60 days on deployed agents)
- Agents with tools not mentioned in their system prompt

### Idle Resources
- Agents created but never deployed (no campaigns)
- Agents with campaigns but 0 conversations in the last 30 days
- Paused campaigns that have been paused for a long time

### Opportunities
- Agents performing well on one channel that could be deployed to additional channels
- Clients without portal access who could benefit from it
- Agents that could be cloned for similar client businesses

## Step 4: Recommendations

Provide 3-5 concrete recommendations, ranked by impact:

```
1. [HIGH IMPACT] Fix the degraded WhatsApp channel for Client B
   The template was rejected by Meta. Create a new template with adjusted content.

2. [HIGH IMPACT] Add tool instructions to Demo Support Agent's prompt
   It has a webhook tool configured but the prompt doesn't mention it. The tool never fires.

3. [MEDIUM] Archive 3 unused agents
   "Voice Engagement Agent", "Bean There Assistant", and "New Agent" have no campaigns
   and no recent activity. Consider archiving to reduce clutter.

4. [MEDIUM] Downgrade Dental Assistant from claude-sonnet to gpt-4o-mini
   It handles simple FAQ and booking. The cheaper model would perform similarly at 14x less cost.

5. [LOW] Add knowledge to Appointment Booking agent
   It has 5 tools but 0 knowledge docs. Adding the client's website content would improve response quality.
```

## Formatting

- Use tables for structured data
- Use bullet points for findings
- Bold the severity level
- Keep the report scannable — executives skim, they don't read paragraphs
- End with a clear "Top 3 Actions" summary

## Gotchas

- Don't try to read conversations for ALL agents — sample the most active ones. The report should complete in under 2 minutes, not 10.
- `get_agent_analytics` may not return useful stats — derive what you can from conversation sampling instead.
- If there are no deployed agents, the report is simple: "No agents are live. Here's what to do next."
- Some agents are intentionally drafts or experiments — don't flag every undeployed agent as a problem. Note them but focus attention on deployed agents with issues.
- Credit cost per conversation is in the `total_credits` field on conversations — use it if available.
