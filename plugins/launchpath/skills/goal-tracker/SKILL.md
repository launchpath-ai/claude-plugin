---
name: goal-tracker
description: Track how many conversations actually converted — bookings made, leads captured, questions answered. Shows a conversion funnel with exactly where and why people dropped off.
user-invocable: true
disable-model-invocation: true
argument-hint: [agent name or ID]
allowed-tools:
  - mcp__launchpath__*
context: fork
---

You are analyzing an agent's real business outcomes. Your job is to answer the question every business owner asks: **"How many people actually converted?"**

## Step 1: Identify the agent

- If the user provided an agent name or ID, use it
- Otherwise call `list_agents` and ask which agent to analyze

Then call `get_agent` to read the system prompt and understand what this agent is supposed to DO.

## Step 2: Determine the business goal

Read the system prompt and tools to infer the agent's primary goal. Map it to a measurable signal:

| Agent type | Goal signal | How to detect |
|------------|-------------|---------------|
| Booking/scheduling agent | Appointment created | `tool-result` with calendar/booking tool + `toolSuccess: true` |
| Lead capture agent | Contact info collected + pushed to CRM/webhook | `tool-result` with CRM/webhook/email tool + `toolSuccess: true` |
| Support agent | Question resolved | Conversation reached natural close (status: "closed", decent message count) |
| Sales agent | Qualification completed | Agent asked qualifying questions + captured info + tool fired |
| FAQ/info agent | Question answered | Conversation closed, no re-engagement, reasonable length |

If the goal isn't clear from the prompt, ask the user: "What counts as a successful conversation for this agent?"

Also call `list_agent_tools` to know which tools exist — these are your conversion signals.

## Step 3: Pull conversation data

Call `list_conversations` for the agent (use campaign_id if deployed, agent_id if test only). Pull the last 50 conversations (or fewer if less exist).

For EACH conversation, call `get_conversation` to read the full transcript.

**Be smart about batching** — if there are 50+ conversations, start with 20 and expand if the user wants more detail.

## Step 4: Categorize every conversation

For each conversation, read the messages and classify it:

**Converted** — The goal was achieved:
- A tool-call message exists for the goal-relevant tool AND the corresponding tool-result has `toolSuccess: true`
- OR for support/FAQ agents: conversation closed naturally with a resolution

**Failed** — The goal was attempted but didn't succeed:
- A tool-call exists but tool-result has `toolSuccess: false` (tool error)
- OR the agent tried to complete the goal but got stuck (repeated attempts, error messages)

**Abandoned** — The visitor left before reaching the goal:
- Conversation has few messages (1-3) and status is still "active" with old timestamp
- OR the visitor stopped responding mid-flow (agent asked a question, no reply)

**Bounced** — Visitor barely engaged:
- 1 message from user, 1 from agent, nothing else
- The greeting or first response didn't hook them

**In Progress** — Still active:
- Recent timestamp, conversation ongoing

## Step 5: Diagnose failures and abandonments

For each **Failed** conversation, identify the specific failure:
- Which tool failed? What was the error?
- Did the agent ask for the wrong information?
- Did the agent hallucinate a capability it doesn't have?
- Was there a knowledge gap (agent said "I don't have that information")?

For each **Abandoned** conversation, identify the drop-off point:
- At which message did the visitor stop?
- What was the agent's last message? (Was it confusing? Did it ask too much?)
- Did the agent go off-topic?
- Was the visitor's question not understood?

## Step 6: Build the conversion funnel

Present results as a funnel:

```
Conversations Started:  [N]
  -> Engaged (3+ msgs):  [N] ([%])
  -> Reached Goal Step:   [N] ([%])
  -> Converted:           [N] ([%])
  -> Failed:              [N] ([%])

Bounced (1-2 msgs):     [N] ([%])
Abandoned mid-flow:     [N] ([%])
In Progress:            [N]
```

## Step 7: Present findings

Show:
1. **The funnel** (above)
2. **Conversion rate**: Converted / (Converted + Failed + Abandoned) as a percentage
3. **Top failure reasons** ranked by frequency:
   - "Calendar tool returned error (7 times) — check the integration connection"
   - "Visitors abandon when asked for phone number (5 times) — consider making it optional"
   - "Agent couldn't answer insurance questions (4 times) — add FAQ about insurance"
4. **Specific fixes** for each failure reason — not just "improve the agent" but exact actions:
   - "Add FAQ: Q: Do you accept insurance? A: [answer]"
   - "The Google Calendar tool failed 7 times — run `test_agent_tool` to check the connection"
   - "Remove phone number from the required fields in the pre-chat form"
5. **Estimated impact**: "Fixing the top 3 issues would recover an estimated [N] conversations per week"

## Gotchas

- Conversations from `list_conversations(agent_id)` are TEST conversations (dashboard). Use `list_conversations(campaign_id)` for REAL customer conversations.
- A conversation with status "active" and a recent timestamp is still in progress — don't count it as abandoned.
- `toolSuccess` may not exist on older conversations — treat missing toolSuccess as unknown, not failed.
- CSAT ratings (in metadata.csat_rating) are an additional signal — if available, mention the average and highlight any 1-2 star conversations.
- Keep the output focused and actionable. Business owners want numbers and fixes, not lengthy analysis.
