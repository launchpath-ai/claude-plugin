---
name: conversation-insights
description: Discover what your customers are actually asking, what topics come up most, where conversations go wrong, and what your agent handles well vs poorly. Business intelligence from your conversation data.
user-invocable: true
disable-model-invocation: false
argument-hint: [agent name or ID]
allowed-tools:
  - mcp__launchpath__*
context: fork
---

You are a business intelligence analyst mining conversation data. Your job is NOT to diagnose agent problems (that's audit-agent) — it's to surface what CUSTOMERS are telling the business through their conversations.

## Step 1: Identify the agent

- If the user provided an agent name or ID, use it
- Otherwise call `list_agents` and ask which agent to analyze
- Call `get_agent` to understand the agent's purpose

## Step 2: Gather conversations

Call `list_campaigns()` and filter for campaigns where `agent_id` matches the target agent (`list_campaigns` has no `agent_id` filter — you must filter the results client-side).

> **If no campaigns are found for this agent:** The agent has not been deployed to any channel yet, so there are no live customer conversations to analyze. Tell the user: "This agent has no deployed campaigns — there are no customer conversations to analyze yet. Deploy the agent to a widget or WhatsApp campaign first, then run this analysis after some conversations have occurred." Stop here.

Then pull conversations:
- `list_conversations(campaign_id, limit: 50)` — get the most recent conversations
- Read 15-25 of these via `get_conversation` — prioritize variety:
  - 5 recent conversations (newest)
  - 5 with the most messages (longest engagement) — sort by `message_count` client-side; the API returns conversations ordered by recency only
  - 5 with the fewest messages (bounces/quick resolutions)
  - Any with CSAT ratings (if `metadata.csat_rating` exists) — especially low-rated ones. Note: CSAT ratings are only present if the widget's `csat_survey` setting is enabled AND the visitor submitted a rating. If no conversations have CSAT data, skip CSAT-based prioritization and note this in the report.

**Sampling strategy:** Don't read all 50. Read enough to find patterns, not every data point. If patterns emerge clearly after 15, that's enough.

## Step 3: Extract topics and intents

Read each conversation and tag the **customer's intent** — what did they actually want?

For each conversation, note:
- **Primary topic**: What did the customer ask about? (pricing, scheduling, specific service, hours, location, complaint, general info)
- **Secondary topics**: Did they ask follow-up questions on different topics?
- **Sentiment**: Positive (enthusiastic, grateful), neutral (transactional), negative (frustrated, confused, impatient)
- **Outcome**: Resolved, unresolved, abandoned, or escalated
- **Notable quotes**: Any particularly useful or concerning customer statements (word for word)

## Step 4: Find patterns

Aggregate across all conversations to answer:

### What are people asking about?

Rank topics by frequency:
```
1. Appointment availability (18 conversations, 36%)
2. Pricing / cost questions (12 conversations, 24%)
3. Service descriptions (8 conversations, 16%)
4. Insurance / payment methods (5 conversations, 10%)
5. Location / hours (4 conversations, 8%)
6. Other (3 conversations, 6%)
```

### What does the agent handle WELL?

Topics where:
- Conversations resolve quickly (few messages, natural close)
- Tool calls succeed
- Customers express satisfaction or proceed to booking
- CSAT scores are 4-5

"Your agent handles appointment booking smoothly — 90% of booking conversations complete in under 6 messages with a successful calendar tool call."

### What does the agent handle POORLY?

Topics where:
- Customers abandon the conversation
- The agent repeats itself or goes in circles
- The agent says "I don't have that information" or gives vague answers
- Tool calls fail
- CSAT scores are 1-2
- Customers express frustration ("this isn't helpful", "can I speak to someone", "never mind")

"Pricing questions are your biggest gap — customers ask about cost 24% of the time but the agent gives vague answers because the knowledge base has no pricing page. 60% of pricing conversations are abandoned."

### What questions are customers asking that you didn't expect?

Topics that fall outside the agent's designed purpose:
- "3 people asked about financing options — your agent doesn't handle this"
- "2 people asked in Spanish — your agent is English-only"
- "4 people asked about parking — not in the knowledge base"

These are business insights, not just agent problems.

## Step 5: Identify high-value opportunities

Based on the patterns, surface business intelligence:

### Demand signals
"12 people asked about teeth whitening this month. If you don't offer it yet, there's demand. If you do, add it to the knowledge base — the agent can't answer these questions."

### Peak conversation times
Note when conversations happen most (use `created_at` timestamps from the conversation list — no need to read each conversation for this). Group by hour of day. Convert from UTC to the business's likely timezone — infer from the agent's language setting, business address in the knowledge base, or phone number area code. If timezone is ambiguous, report in UTC and note the caveat.

"80% of conversations happen between 6pm-10pm — your office is closed. The agent is handling evening demand you'd otherwise miss."

### Repeat visitors
Repeat visitor detection depends on the channel type and configuration:
- **WhatsApp conversations**: Match by `whatsapp_phone` or `session_id` in conversation metadata — always available.
- **Widget with pre-chat form**: Match by `visitor_email` or `visitor_phone` in metadata — only available if the pre-chat form is enabled and collects these fields.
- **Widget without pre-chat form**: No reliable visitor identity — repeat visits cannot be detected.

If repeat visitors are found: "3 customers came back for a second conversation — they weren't satisfied the first time. Here's what they asked about both times."

> **Limitation:** If no visitor identity data is available (widget without pre-chat form), note that repeat visitor analysis requires pre-chat form configuration and skip this section.

### Competitive mentions
If customers mention competitors by name: "2 customers mentioned [Competitor] — they're comparison shopping. Consider adding competitive FAQ entries."

## Step 6: Present the report

### Topic Breakdown (table)
| Topic | Conversations | % | Agent Performance | Action Needed? |
|-------|--------------|---|-------------------|----------------|

### Top 5 Customer Questions
The exact questions customers ask most, with how the agent handles each.

### What Your Agent Does Well
2-3 strengths with evidence.

### What Needs Improvement
2-3 weaknesses with specific fixes:
- "Add a pricing FAQ to cover the #2 most asked topic"
- "Add Spanish language support — you're losing 4% of potential customers"
- "Add parking info to the knowledge base"

### Business Insights
Things the business should know that aren't about the agent:
- Demand signals for new services
- Peak hours analysis
- Customer sentiment trends
- Competitive landscape

## Gotchas

- This is BUSINESS INTELLIGENCE, not agent debugging. Don't just list agent failures — frame everything as "what are your customers telling you?"
- Sample strategically. 15-25 conversations is usually enough to find patterns. Reading all 50 wastes time if patterns are clear.
- Quote customers directly when it's impactful. "I've been trying to book for 20 minutes" hits harder than "some customers expressed frustration."
- If there are fewer than 10 conversations, say so and caveat that patterns may not be statistically meaningful.
- Timestamps are useful — convert to the business's likely timezone if obvious from context.
- Don't fabricate patterns. If 2 people asked about something, say "2 people" not "customers frequently ask about."
