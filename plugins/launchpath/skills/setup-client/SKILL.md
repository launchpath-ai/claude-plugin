---
name: setup-client
description: Create a client account on LaunchPath, link an existing agent as a campaign, and generate deployment details.
user-invocable: true
argument-hint: [client name]
allowed-tools:
  - mcp__launchpath__*
context: fork
---

You are setting up a new client on LaunchPath. Follow this workflow:

## Step 1: Gather information

Ask the user for:
- **Client name** (required)
- **Client email** (optional)
- **Client website** (optional)
- **Which agent to assign** — show the list from `list_agents` and let them pick

## Step 2: Create the client

Call `create_client` with the provided details.

## Step 3: Create the campaign

Call `create_campaign` linking the chosen agent to the new client. Use a descriptive name like "[Client Name] - [Agent Name]".

Ask which channel type: widget (default), whatsapp, or both.

## Step 4: Deploy

### For widget (default):
1. `configure_widget` — customize colors, theme, avatar, welcome message, conversation starters, and behavior (pre-chat form, CSAT, auto-close, file upload)
2. `update_campaign(status: "active")` to activate
3. `get_embed_code` to get the HTML snippet

### For WhatsApp:
1. Tell the user they must enter Meta credentials in the LaunchPath dashboard
2. `update_campaign(status: "active")` once credentials are saved
3. Guide through template creation with `create_wa_template`

## Step 5: Optional portal access

Ask if they want to invite the client to the portal:
- If yes, call `invite_client_member` with the client's email and role ("admin" or "member")
- The client will receive an email invitation to access their portal

## Step 6: Present results

Summarize:
- Client created with ID
- Campaign linked (agent → client)
- Deployment details (embed code or WhatsApp status)
- Portal invite status (if applicable)
- Next steps (customize widget, add more knowledge, monitor conversations)
