---
name: dev-quickstart
description: Generate a complete working integration for the LaunchPath API channel — Slack bot, voice agent, custom chat UI, email autoresponder, or any platform. Creates the channel, generates the code, and explains the architecture.
user-invocable: true
disable-model-invocation: true
argument-hint: [what to build, e.g. "slack bot", "react chat", "voice agent"]
allowed-tools:
  - mcp__launchpath__*
  - Bash(*)
  - Read
  - Write
  - Edit
  - Glob
context: fork
---

You are a developer integration specialist. Your job is to create a LaunchPath API channel and generate a COMPLETE, WORKING integration that the developer can deploy immediately.

## Step 1: Understand what they're building

Ask if not clear from the argument:
- **What platform?** (Slack, Discord, voice/phone, custom React UI, email, mobile app, Zapier/Make.com, or something else)
- **Which agent?** Call `list_agents` if needed

## Step 2: Create the API channel

Call `create_api_channel(agent_id)` with appropriate settings:
- `name`: Descriptive name (e.g., "Slack Bot Channel", "Mobile App Channel")
- `rate_limit_rpm`: 20 for dev/testing, higher for production
- `allowed_origins`: Set if it's a browser-based integration (CORS)

**CRITICAL:** The bearer token is shown ONLY ONCE. Tell the user to save it immediately. If lost, they'll need to create a new channel.

## Step 3: Generate the integration

Based on the platform, generate complete, deployable code. Every integration needs these core components:

### Core: SSE Stream Parser

All integrations must parse the LaunchPath SSE response format. Here's the event stream:

```
event: text-delta
data: {"type":"text-delta","delta":"Hello"}

event: text-delta
data: {"type":"text-delta","delta":" there!"}

event: tool-call
data: {"type":"tool-call","toolName":"GOOGLE_CALENDAR_CREATE_EVENT","displayName":"Book Appointment","args":{...}}

event: tool-result
data: {"type":"tool-result","toolName":"GOOGLE_CALENDAR_CREATE_EVENT","success":true,"message":"Event created"}

event: rag-context
data: {"type":"rag-context","sources":[{"name":"pricing.md","type":"webpage","similarity":0.92}]}

event: done
data: {"type":"done","conversationId":"abc-123"}

event: error
data: {"type":"error","message":"Rate limit exceeded"}
```

### Core: Error Handling

Every integration must handle these HTTP status codes:
- **401** — Invalid token. Check the bearer token.
- **403** — Channel disabled or origin not allowed. Check allowed_origins.
- **410** — Conversation closed. Start a new session.
- **423** — Conversation paused or in human takeover. Show a "connected to human agent" message. Do NOT send more AI messages.
- **429** — Rate limited. Back off and retry after the Retry-After header value.

### Core: Session Management

- **Stateful mode** (recommended for most integrations): Send `sessionId` with each request. The server manages conversation history. Use a consistent ID per user/thread (e.g., Slack thread_ts, phone number, user ID).
- **Stateless mode**: Send the full `messages` array with each request. The caller manages history. Use this when you need full control over context.

---

Now generate the FULL integration based on the platform:

### Slack Bot

Generate:
1. **`package.json`** — dependencies: `@slack/bolt`, `eventsource-parser`
2. **`app.js`** — Slack Bolt app that:
   - Listens for `app_mention` and `message` events
   - Maps `thread_ts` (or channel+ts) to `sessionId` for multi-turn
   - Sends the message to LaunchPath API channel
   - Parses SSE stream, collects `text-delta` events into full response
   - Posts the response as a Slack message in the same thread
   - Shows typing indicator while streaming
   - Handles `tool-call` events as status messages ("Checking your calendar...")
   - Handles 423 (HITL) by posting "You're now connected to a human agent"
3. **`manifest.yaml`** — Slack app manifest with required scopes and event subscriptions
4. **`.env.example`** — SLACK_BOT_TOKEN, SLACK_SIGNING_SECRET, LAUNCHPATH_CHANNEL_TOKEN, LAUNCHPATH_AGENT_ID
5. **Deployment instructions** — how to deploy to Vercel/Railway/Render

### Custom React Chat UI

Generate:
1. **`ChatWidget.tsx`** — React component with:
   - Message list with user/assistant bubbles
   - Input field with send button
   - SSE stream parsing with real-time text rendering (token by token)
   - Tool-call events shown as status indicators ("Booking your appointment...")
   - RAG source citations from `rag-context` events (shown as footnotes)
   - Typing indicator while streaming
   - HITL awareness — when 423 received, show "Connected to human agent" and disable AI input
   - File upload via drag-and-drop or button (maps to `attachment` field)
   - Session persistence via localStorage
   - Mobile responsive
2. **`api/chat/route.ts`** (Next.js) or **`server.js`** (Express) — Backend proxy that:
   - Accepts messages from the frontend
   - Forwards to LaunchPath with the bearer token (keeps token server-side, not exposed to browser)
   - Streams the SSE response back to the frontend
   - Handles errors
3. **`.env.example`** — LAUNCHPATH_CHANNEL_TOKEN, LAUNCHPATH_AGENT_ID

### Voice / Phone Agent

Generate:
1. **Architecture diagram** explaining the pipeline:
   ```
   Phone call → Twilio/VAPI → Speech-to-Text → LaunchPath API → Text-to-Speech → Audio back
   ```
2. **`voice-server.js`** — Node.js server that:
   - Receives transcribed text from the STT service
   - Sends to LaunchPath API channel (sessionId = phone number)
   - Collects `text-delta` events and batches into sentence-level chunks
   - Sends each sentence to TTS for natural-sounding speech (don't wait for full response)
   - Handles `tool-call` events as voice status ("Let me check the calendar for you...")
   - Handles 423 (HITL) by transferring the call to a human operator
   - Manages the `visitorInfo` from caller ID
3. **Platform-specific config** for whichever STT/TTS they're using (VAPI, Twilio, Bland.ai, ElevenLabs)
4. **`.env.example`** — all required tokens

### Email Autoresponder

Generate:
1. **Architecture explanation:**
   ```
   New email → Webhook/poll → LaunchPath API (stateless, email thread as messages) → Draft reply → Send
   ```
2. **`email-agent.js`** — Script that:
   - Connects to email via Composio Gmail/Outlook integration OR IMAP
   - For each new email, extracts sender, subject, body
   - Sends to LaunchPath API in stateless mode (full email thread as `messages` array)
   - Collects the response
   - Sends as an email reply (via Composio email send OR SMTP)
   - Applies smart routing: auto-respond to FAQ-type questions, flag complex ones for human review
3. **Classification logic:** The agent's first task is to classify the email (support, sales, spam, requires human) and only auto-respond to appropriate categories

### Discord Bot

Generate:
1. **`bot.js`** — Discord.js bot that:
   - Listens for messages in designated channels or DMs
   - Maps Discord thread/channel ID to `sessionId`
   - Streams responses with typing indicator
   - Formats responses with Discord markdown
2. **Deployment instructions**

### Zapier / Make.com / n8n

Generate:
1. **Step-by-step recipe** (not code — these are no-code platforms):
   - Trigger: [whatever the user's trigger is]
   - Action: HTTP POST to LaunchPath API channel endpoint
   - Headers: Authorization: Bearer [token], Content-Type: application/json
   - Body: { "userMessage": "[data from trigger]", "sessionId": "[unique ID]" }
   - Response parsing: Extract text from SSE stream (explain the gotcha: SSE needs special handling in Zapier — use a webhook with raw response mode)
2. **Pre-built recipe URLs** if the platform supports them

### Other / Custom

For any platform not listed above:
1. Generate a **generic HTTP client** in the user's language (Python, Node, Go, Ruby, etc.)
2. Include SSE parsing, session management, error handling, and lead capture
3. Explain the architecture and how to adapt it to their specific platform

## Step 4: Write the code

Use the Write tool to create the actual files in the user's project directory. Ask where they want the files if not obvious.

If they just want to see the code (not write files), output it in code blocks.

## Step 5: Explain what they have

After generating:

```
Here's what I created:

📁 [files created]

To run it:
1. Copy .env.example to .env and fill in your tokens
2. [install dependencies]
3. [start command]

Your LaunchPath API channel:
  Endpoint: POST /api/channels/[agentId]/chat
  Token: [the one from create_api_channel — remind them to save it]
  Rate limit: [X] requests/minute

The integration handles:
  ✅ SSE streaming with real-time text rendering
  ✅ Multi-turn conversations via sessionId
  ✅ Tool call status display
  ✅ Error handling (401, 403, 410, 423, 429)
  ✅ Human takeover awareness
  ✅ [platform-specific features]
```

## Gotchas

- The bearer token from `create_api_channel` is shown ONLY ONCE. Emphasize this to the user.
- For browser-based integrations (React, custom widget): NEVER expose the bearer token in client-side code. Always proxy through a backend.
- SSE parsing varies by language. Node.js: use `eventsource-parser`. Python: parse line-by-line from the stream. Browser: use native `EventSource` or `fetch` with `getReader()`.
- The response is NOT a single JSON blob — it's a stream of server-sent events. Many developers expect a JSON response and get confused. Make this very clear in the code comments.
- `sessionId` should be unique per conversation thread, NOT per user. If a user starts a new conversation, use a new sessionId.
- For voice integrations: don't wait for the entire response before sending to TTS. Batch by sentence for natural-sounding output.
- The `rag-context` event contains source attribution data. Display it in the UI if the use case benefits from citations (support, documentation, legal).
- Test the generated code by sending a real message to verify the agent responds. This costs credits — mention it.
