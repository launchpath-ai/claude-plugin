---
name: whatsapp-campaign
description: Set up a complete WhatsApp campaign — create templates, import contacts, and launch broadcasts or drip sequences.
user-invocable: true
argument-hint: [campaign name or description]
allowed-tools:
  - mcp__launchpath__*
context: fork
---

You are setting up a WhatsApp campaign using LaunchPath. Follow this workflow:

## Step 1: Find or create the campaign

- Call `list_campaigns` to check if a WhatsApp campaign already exists
- If not, ask the user which agent and client to use (call `list_agents` and `list_clients` to help)
- Call `create_campaign` with `channel_type: "whatsapp"` to create the campaign

## Step 2: Check WhatsApp credentials

WhatsApp requires Meta Business credentials that CANNOT be set via the terminal.
- Tell the user: "You need to enter your Meta credentials (Phone Number ID, Access Token, Business Account ID, App Secret) in the LaunchPath dashboard."
- Provide the direct link: `https://www.trylaunchpath.com/dashboard/campaigns/<campaign_id>`
- Wait for the user to confirm credentials are saved before continuing

## Step 3: Create message templates

All outbound WhatsApp messages require a Meta-approved template. Always create one first.

- Ask the user what messages they want to send
- Call `create_wa_template` for each message template
- Explain that Meta must approve templates (1-24 hours) before they can be sent
- Use `list_wa_templates` to check approval status

Template tips:
- Use {{1}}, {{2}} for variables (e.g., "Hi {{1}}, your appointment is on {{2}}")
- MARKETING category for promotions, UTILITY for transactional, AUTHENTICATION for login codes
- Template names must be lowercase with underscores only
- If a template is rejected, check `list_wa_templates(status: "REJECTED")` and create a new one with adjusted content

## Step 4: Import contacts

- Ask the user for their contact list
- Call `import_wa_contacts` with the contacts (phone number required, name/email/tags/custom_fields optional)
- Include `custom_fields` if templates use variables beyond name (e.g., `{ "company": "Acme Inc" }`)
- Use tags for segmentation (e.g., "vip", "new-customer")

## Step 5: Choose sending strategy

Ask the user: "Would you like to send a one-time broadcast or set up a drip sequence?"

### Option A: One-time broadcast
1. Verify templates are approved: `list_wa_templates`
2. Call `send_wa_broadcast` with:
   - `audience_filter` to target by tags/status (without a filter, sends to all active contacts)
   - `variable_mapping` to personalize messages (e.g., `{ "1": "name", "2": "custom_fields.company" }`)
   - Optionally `scheduled_for` to schedule for a future time
3. Check results with `list_send_jobs`
4. If messages failed, use `get_send_job_messages(job_id, status: "failed")` to see why

### Option B: Drip sequence
1. Call `create_wa_sequence` with timed steps:
   - Each step has a `delay_minutes` (0 for immediate, 1440 for next day, 60 for 1 hour, etc.)
   - Use `variable_mapping` per step for personalization
   - Set `stop_on_reply: true` if the sequence should stop when someone replies
2. Call `activate_wa_sequence` with `status: "active"` to start the sequence
3. Call `enroll_wa_contacts` with contact IDs or a filter to add contacts
4. Monitor with `get_sequence_stats` (overview) or `get_sequence_detail` (per-step metrics)
5. If enrollments fail, use `get_failed_enrollments(sequence_id)` to diagnose

## Step 6: Configure messaging behavior

Use `configure_wa_messaging` to set:
- Response delay (simulates typing)
- Smart follow-ups

## Step 7: Present results

Summarize what was set up:
- Campaign name and linked agent
- Templates created (and their approval status)
- Number of contacts imported
- Broadcast sent or sequence activated
- How to monitor: `list_send_jobs` for broadcasts, `get_sequence_stats` for drip sequences
- Troubleshooting: `get_channel_health` if issues arise, `sync_wa_templates` if template statuses seem stale
