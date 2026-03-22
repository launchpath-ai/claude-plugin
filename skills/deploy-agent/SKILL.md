---
name: deploy-agent
description: Create, train, and deploy an AI agent for a client's business. Scrapes their website, builds a knowledge base, tests it, and deploys to widget, WhatsApp, or API.
user-invocable: true
argument-hint: [business description or website URL]
allowed-tools:
  - mcp__launchpath__*
context: fork
---

You are deploying an AI agent using LaunchPath. Follow this workflow precisely:

## Step 1: Parse the user's input

- If they gave a **website URL**, you'll scrape it for knowledge
- If they **described a business**, use that as context for the system prompt
- If both, use both

## Step 2: Create the agent

Use `create_agent` and write the system prompt yourself. The system prompt should include:
- The business type, industry, and what the agent helps with (support, sales, scheduling, etc.)
- Tone and personality (e.g., "Be friendly and professional")
- A greeting message (e.g., "Start every conversation with: Hi! How can I help you today?")
- Any specific instructions from the user

Choose an appropriate model (default: `openai/gpt-4o-mini`) and language.

## Step 3: Add knowledge

- If a website URL was provided, call `discover_pages` to find all pages on the domain
- Then call `scrape_website` for each relevant page (prioritize: about, services, pricing, FAQ, contact)
- Call `generate_faqs` to auto-create Q&A pairs from the scraped content
- Add any additional FAQs with `add_faq`

## Step 4: Add integrations (if applicable)

If the business needs integrations (calendar booking, email, CRM, etc.):
1. `browse_integrations` to find the right app
2. `list_app_actions` to see available actions
3. `check_connections` to verify the app is connected
4. If not connected: `connect_app` to connect it
   - API key apps: call without credentials first to discover required fields, then call again with credentials
   - OAuth apps: returns a browser URL for the user to authorize, then `poll_connection` to verify
5. `add_agent_tool` to attach the integration to the agent
6. Update the system prompt via `update_agent` with instructions for how the agent should use the tools

## Step 5: Test

Ask the user if they'd like to test the agent before deploying.
- Call `chat_with_agent` with 2-3 test questions relevant to the business
- Verify the agent responds accurately using the scraped knowledge
- If responses are poor, add more knowledge or FAQs and re-test

## Step 6: Ask about deployment

Ask the user which channel they want to deploy to:

### Option A: Website Widget (default)
1. Create a client if one doesn't exist: `create_client`
2. `create_campaign` with `channel_type: "widget"`
3. `configure_widget` — customize the widget appearance and behavior:
   - **Branding**: `primary_color`, `theme`, `border_radius`, `widget_size`, `show_branding`
   - **Icons**: `agent_avatar` and `launcher_icon` — set to the client's logo URL or an emoji
   - **Content**: `agent_name`, `welcome_message`, `greeting_message`, `greeting_delay`, `conversation_starters`
   - **Behavior**: `pre_chat_form`, `csat_survey`, `end_chat`, `auto_close`, `file_upload`
4. `update_campaign(status: "active")` to activate
5. `get_embed_code` to get the ready-to-paste HTML snippet

### Option B: WhatsApp
Tell the user: "For WhatsApp, we need to set up a campaign. Let me walk you through it."
1. Create a client if one doesn't exist: `create_client`
2. `create_campaign` with `channel_type: "whatsapp"`
3. Tell the user they must enter Meta credentials in the LaunchPath dashboard before proceeding
4. Once credentials are saved, guide them through templates, contacts, and broadcasts or drip sequences

### Option C: API Channel
For developers building custom chat UIs or integrations:
1. `create_api_channel(agent_id)` — creates an API endpoint with a bearer token
2. The response includes:
   - The endpoint URL: `POST /api/channels/{agentId}/chat`
   - A bearer token (shown only once — tell the user to save it)
   - A ready-to-use curl example
3. Explain the key features:
   - **Stateful mode**: Pass `sessionId` — server manages conversation history
   - **Stateless mode**: Pass `messages` array — caller manages history
   - **Lead capture**: Pass `visitorInfo: { name, email }` to capture visitor details
   - **File uploads**: Pass `attachment: { url, fileName, fileType, fileSize }`
   - **SSE streaming**: Response streams events (`text-delta`, `tool-call`, `done`, etc.)
4. Use `list_channels` to verify the channel was created

### Option D: Skip deployment
If the user doesn't want to deploy yet, skip this step. Remind them they can deploy later.

## Step 7: Present results

Summarize what was created:
- Agent name and what it does
- Knowledge base contents (pages scraped, FAQs added)
- Integrations added (if any)
- Test conversation results (if tested)
- Deployment details (embed code, WhatsApp status, or API endpoint + token)
- Link to LaunchPath dashboard for further configuration
