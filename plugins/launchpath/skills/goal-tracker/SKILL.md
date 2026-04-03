---
name: goal-tracker
description: Track conversions across channels — which conversations booked, which leads hit your CRM, which WhatsApp contacts converted. Smart action classification so only real outcomes count.
user-invocable: true
disable-model-invocation: false
argument-hint: [agent name or ID]
allowed-tools:
  - mcp__launchpath__*
context: fork
---

You are analyzing an agent's real business outcomes. Your job is to answer the question every business owner asks: **"How many people actually converted?"**

You do this by reading conversation transcripts and classifying tool actions as **conversion signals** (CREATE, SEND, BOOK) vs **intermediate steps** (SEARCH, LIST, GET). Only real outcomes count.

## Step 1: Identify the agent

- If the user provided an agent name or ID, use it
- Otherwise call `list_agents` and ask which agent to analyze

Then call `get_agent` to read the system prompt and understand what this agent is supposed to DO.

## Step 2: Map tools to conversion signals

Call `list_agent_tools` to get all tools attached to this agent.

> **If the agent has no tools (0 results):** This agent is a pure FAQ/support agent with no conversion actions. Skip Steps 2-5 (tool classification, conversation analysis for tool calls). Instead, go directly to a simplified analysis: pull recent conversations via `list_conversations(campaign_id, limit: 20)`, read 10 of them, and classify by outcome signals — conversations that closed naturally after 4+ messages count as "resolved," short conversations or abandoned ones count as "bounced." Present a simplified engagement funnel instead of a conversion funnel. Skip Steps 7-8 (WhatsApp/CRM cross-reference) unless specifically requested.

For each tool, classify it as a **conversion action** or an **intermediate action**:

### Composio tools

Composio tool names are action slugs like `GOOGLECALENDAR_CREATE_EVENT`. Classify by the verb:

**Conversion actions** (the goal was achieved):
| Verb pattern | What it means | Examples |
|-------------|---------------|---------|
| `*_CREATE_*` | Created a record | `GOOGLECALENDAR_CREATE_EVENT`, `HUBSPOT_CREATE_CONTACT`, `HUBSPOT_CREATE_DEAL` |
| `*_SEND_*` | Sent something | `GMAIL_SEND_EMAIL`, `SLACK_SEND_MESSAGE` |
| `*_BOOK_*`, `*_SCHEDULE_*` | Booked/scheduled | `CALENDLY_SCHEDULE_EVENT` |
| `*_ADD_*` | Added a record | `HUBSPOT_ADD_NOTE`, `PIPEDRIVE_ADD_DEAL` |
| `*_INSERT_*` | Inserted data | `GOOGLESHEETS_INSERT_ROW` |

**Intermediate actions** (steps toward the goal, NOT conversions):
| Verb pattern | What it means | Examples |
|-------------|---------------|---------|
| `*_SEARCH_*`, `*_FIND_*` | Looked something up | `HUBSPOT_SEARCH_CONTACTS`, `SALESFORCE_FIND_RECORDS` |
| `*_LIST_*` | Listed records | `GOOGLECALENDAR_LIST_EVENTS`, `HUBSPOT_LIST_DEALS` |
| `*_GET_*`, `*_FETCH_*`, `*_READ_*` | Read a record | `HUBSPOT_GET_CONTACT`, `STRIPE_GET_CHARGE` |
| `*_CHECK_*`, `*_VERIFY_*` | Checked status | `STRIPE_CHECK_SUBSCRIPTION` |

**Ambiguous — use context:**
| Verb pattern | When it's a conversion | When it's intermediate |
|-------------|----------------------|----------------------|
| `*_UPDATE_*` | Updating a deal stage to "won" | Updating a contact's last-seen timestamp |

For UPDATE actions, check the agent's system prompt for context. If the prompt says "update the deal stage when the customer agrees to proceed", treat that UPDATE as a conversion.

### Other tool types

| Tool type | Classification | Reasoning |
|-----------|---------------|-----------|
| **Webhook** | Always **conversion** | Webhooks only fire when the agent decides something happened worth notifying |
| **HTTP (POST/PUT)** | Likely **conversion** | Writing data to an external system |
| **HTTP (GET)** | **Intermediate** | Reading data |
| **Subagent** | **Neither** — look at what the subagent did | The parent delegating isn't a conversion; check the subagent's tool calls if visible |
| **Knowledge search** (`search_knowledge_base`) | Always **intermediate** | Looking up info to answer a question |

### Confirm with the user

After classification, present it:

```
I've identified these as your conversion signals:
  ✓ GOOGLECALENDAR_CREATE_EVENT — appointment booked
  ✓ HUBSPOT_CREATE_CONTACT — new lead captured
  ✓ HUBSPOT_CREATE_DEAL — deal created
  ✓ send_notification (webhook) — notification fired

These are intermediate steps (NOT counted as conversions):
  · GOOGLECALENDAR_LIST_EVENTS — checking availability
  · HUBSPOT_SEARCH_CONTACTS — looking up existing customer
  · search_knowledge_base — answering questions

Does this look right, or should I adjust?
```

Wait for the user to confirm or adjust before proceeding. This is critical — if the classification is wrong, the entire analysis is wrong.

## Step 3: Pull conversation data

Call `list_campaigns()` and filter for campaigns where `agent_id` matches the target agent (`list_campaigns` has no `agent_id` filter — you must filter client-side). Then pull conversations:
- Use `list_conversations(campaign_id)` for REAL customer conversations
- Use `list_conversations(agent_id)` only for test conversations — warn the user if that's all there is

> **Warning:** `get_conversation` only works with campaign conversation IDs (from `channel_conversations` table). Test conversation IDs from `list_conversations(agent_id)` are stored in a different table and will return 404 with `get_conversation`. If only test conversations exist, inform the user that full transcript analysis requires live campaign conversations.

**Batching strategy:** Start with `list_conversations(campaign_id, limit: 20)`. Call `get_conversation` for each to read the full transcript. If conversion patterns are clear after 20, stop. If not, fetch another batch. Never read all 50 blindly — each `get_conversation` is a separate API round-trip.

## Step 4: Classify every conversation

For each conversation, read the messages and classify:

**Converted** — A conversion action fired AND succeeded:
- A `tool-call` message exists where `toolName` matches a conversion action
- The corresponding `tool-result` has `toolSuccess: true`
- Note WHICH conversion action fired (booking vs lead capture vs email sent — they may differ)

**Failed** — A conversion action was attempted but failed:
- A `tool-call` exists for a conversion action but `tool-result` has `toolSuccess: false`
- Extract the error from the tool-result's `content` field

**Partial** — Engaged with tools but didn't reach the conversion:
- Intermediate actions fired successfully (searched calendar, found availability, looked up contact)
- But NO conversion action ever fired
- This means the visitor was interested — they went through the flow — but dropped off before the final step
- This is DIFFERENT from abandoned (never engaged with tools at all)

**Abandoned** — Left mid-conversation without engaging tools:
- Multiple messages exchanged but no tool calls at all
- OR visitor stopped responding after the agent asked for information
- Status is "active" with an old timestamp (>24h)

**Bounced** — Barely engaged:
- 1-2 messages total, then nothing
- The greeting or first response didn't hook them

**In Progress** — Still active:
- Recent timestamp (<1h), conversation ongoing

## Step 5: Diagnose failures and drop-offs

For **Failed** conversations:
- Which conversion tool failed? What error?
- Is it a connection issue (run `test_agent_tool` to check)?
- Or a data issue (wrong arguments passed)?

For **Partial** conversations (the most actionable category):
- What was the LAST intermediate action before they dropped off?
- Did the agent ask for something the visitor didn't want to provide?
- Did the agent fail to transition from information-gathering to booking/capture?
- Example: "8 visitors searched the calendar but never booked — the agent found available slots but didn't proactively suggest one"

For **Abandoned** conversations:
- At which message did the visitor stop?
- What was the agent's last message?
- Common pattern: agent asks too many questions before getting to the point

## Step 6: Build the conversion funnel

```
Conversations Analyzed: [N]

  Started:               [N]
  ├─ Engaged (3+ msgs):  [N] ([%])
  │  ├─ Used tools:       [N] ([%])
  │  │  ├─ Converted:     [N] ([%]) ← real goal completions
  │  │  ├─ Failed:        [N] ([%]) ← tool errors
  │  │  └─ Partial:       [N] ([%]) ← engaged but didn't finish
  │  └─ No tools used:    [N] ([%]) ← info-only conversations
  ├─ Abandoned:           [N] ([%])
  └─ Bounced:             [N] ([%])

  In Progress:            [N] (excluded from rates)

Conversion Rate: [N]% (Converted / total excluding In Progress)
```

If the agent has MULTIPLE conversion actions (e.g., both calendar and CRM), break them out:
```
Conversions by type:
  Appointments booked (GOOGLECALENDAR_CREATE_EVENT):  12
  Leads captured (HUBSPOT_CREATE_CONTACT):            18
  Deals created (HUBSPOT_CREATE_DEAL):                 6
```

## Step 7: WhatsApp contact cross-reference

**Only do this step if the agent has a WhatsApp campaign.**

1. Call `list_campaigns` and find WhatsApp campaigns for this agent
2. Call `list_wa_contacts` (with the campaign_id) to get all contacts
3. Cross-reference contacts against the conversations you already analyzed

Present:
```
WhatsApp Contacts: [N]
  → Had a conversation:     [N] ([%])
  → Converted:              [N] ([%] of conversations)
  → Partial (engaged, didn't convert): [N] ([%])
  → Bounced/Abandoned:      [N] ([%])
  → No conversation yet:    [N] — imported but never messaged or never replied
```

If many contacts have "No conversation yet", suggest: "Consider sending a broadcast to re-engage these [N] contacts — use `/launchpath:whatsapp-compliance` to write a compliant template first."

## Step 8: CRM action tracking

**Only do this step if the agent has CRM tools (HubSpot, Salesforce, Pipedrive, Zoho).**

Identify CRM tools from `list_agent_tools` — look for tools with `tool_type: "composio"` and `display_name` containing CRM names (e.g., "HubSpot", "Salesforce", "Pipedrive", "Zoho"). The `config.toolkit` field may also be available but can be masked in API responses — fall back to `display_name` matching if needed.

From the conversations you already analyzed, extract every CRM tool call:

```
CRM Actions ([CRM Name]):
  Contacts created:    [N] (successful) / [N] (failed)
  Contacts searched:   [N] (found existing) / [N] (not found → new lead)
  Deals created:       [N]
  Notes logged:        [N]
```

If both WhatsApp contacts AND CRM tools exist, show the full pipeline:
```
WhatsApp → [CRM Name] Pipeline:
  WhatsApp contacts:         45
  Had a conversation:        32
  Added to [CRM]:            14 (31%) ← HUBSPOT_CREATE_CONTACT succeeded
  Already in [CRM]:           8 (18%) ← HUBSPOT_SEARCH_CONTACTS found them
  NOT in [CRM]:              23 (51%) ← these are leads you're losing

  Recommendation: 23 contacts conversed but were never added to your CRM.
  Check if the agent's prompt instructs it to create contacts — if not, update
  the prompt with CRM instructions (e.g., "After qualifying a lead, create a contact in HubSpot with their name, email, and phone number"). Use `update_agent` to modify the prompt and `add_agent_tool` to add the CRM action if missing.
```

## Step 9: Present findings

Show in this order:

1. **Conversion funnel** (Step 6)
2. **Top failure/drop-off reasons** ranked by frequency with specific fixes
3. **WhatsApp pipeline** (Step 7, if applicable)
4. **CRM sync summary** (Step 8, if applicable)
5. **Estimated impact**: "Fixing the top 3 issues would recover an estimated [N] conversions per week"

For each failure reason, give an EXACT fix — not "improve the agent" but:
- "Add FAQ: Q: Do you accept insurance? A: [answer]"
- "The Google Calendar tool failed 7 times — run `test_agent_tool` to check the connection"
- "8 visitors searched availability but never booked — update the prompt to proactively suggest a time slot after showing availability"
- "23 WhatsApp contacts aren't being added to HubSpot — add a CREATE_CONTACT action and update the prompt with CRM instructions"

## Gotchas

- **Conversion vs intermediate is the whole point.** Never count a SEARCH or LIST action as a conversion. If you're unsure about an action, ask the user.
- Conversations from `list_conversations(agent_id)` are TEST conversations. Use `list_conversations(campaign_id)` for real customer data. Warn the user if you can only find test data.
- A conversation with status "active" and a recent timestamp is still in progress — don't count it as abandoned. Use a 1-hour cutoff for "recent".
- `toolSuccess` may not exist on older conversations — treat missing toolSuccess as unknown, not failed.
- CSAT ratings (in `metadata.csat_rating`) are an additional signal — if available, mention the average and highlight any 1-2 star conversations.
- The `toolArgs` field contains PII (names, emails, phones). When showing CRM data, use it for aggregation ("14 contacts created") not for listing individual records.
- **Don't read all 50 conversations if patterns are clear after 20.** But DO read more if the conversion rate seems surprisingly high or low — small sample sizes mislead.
- Some agents have NO tools at all (pure FAQ/support). For these, fall back to conversation closure signals: status "closed", reasonable message count (4+), no frustration signals in the transcript.
