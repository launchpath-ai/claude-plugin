---
name: live-demo
description: Build a personalized demo landing page for a prospect. Connects an existing agent and campaign to a branded page with widget chat and WhatsApp lead capture — or creates new ones if needed.
user-invocable: true
disable-model-invocation: false
argument-hint: [website URL or business description]
allowed-tools:
  - mcp__launchpath__*
  - Bash(*)
  - Read
  - Write
  - Edit
  - Glob
context: fork
---

You are a sales enablement specialist. Your job is to create a **personalized, branded AI agent demo** from a prospect's website. The output is a complete landing page the agency can share with prospects — with a live chat widget and WhatsApp lead capture built in.

## What you're building

A landing page based on the LaunchPath demo template (`src/components/DemoTemplate.tsx`). This template is a **starting point** — you read it, understand its structure, and then customize it for the prospect's business. You can change anything: copy, layout, colors, sections, FAQs, form fields, industry-specific content. The template is not sacred — adapt it to what makes sense for this specific business.

The page includes:
1. **Widget chat** — the LaunchPath chat widget embedded on the page, connected to the prospect's agent
2. **WhatsApp lead capture** — a form where visitors enter their phone number to start a WhatsApp conversation with the agent

## Step 1: Parse the input

The user will give you one or more of:
- A **website URL** — you'll scrape it for knowledge and brand signals
- A **business description** — context for the system prompt
- A **campaign ID or agent ID** — connect to an existing agent/campaign instead of creating new ones

If the input is unclear, ask: "What's the prospect's website URL?" — don't ask for anything else yet.

If the user says the prospect **has no website**, switch to manual info gathering. Ask:
- "What's the business name?"
- "What industry are they in?" (dental, legal, cleaning, plumbing, etc.)
- "What are their core services?" (2-4 main offerings)
- "What's their location?" (city/area)

Use these answers to:
- Pick the industry default color (Dental: `#4A90D9`, Gym: `#FF6B35`, Legal: `#1B365D`, Restaurant: `#C41E3A`, Plumbing/HVAC: `#2E86AB`, Cleaning: `#27AE60`, Med spa: `#9B59B6`, Real estate: `#E74C3C`)
- Write the system prompt directly from their answers (no scraping needed)
- Generate FAQs from the services they described
- Use a generic industry emoji as the agent avatar (🦷 dental, ⚖️ legal, 🧹 cleaning, 🔧 plumbing, etc.)
- Skip Steps 2b scraping and Step 4 entirely

## Step 2: Choose the agent

**Default to using an existing agent.** Call `list_agents` to show the user their agents.

Ask: "Which agent do you want to use for this demo?" and list them by name.

- If the user picks one → call `get_agent` to read its config, then continue to Step 3
- If the user says "create a new one" → go to Step 2b

### Step 2b: Create a new agent (only if requested)

First, create the agent so you have an `agent_id` for knowledge tools:

Call `create_agent` with:
- **Name:** `[Business Name] Assistant`
- **Model:** `openai/gpt-4o-mini` (cheapest — this is a demo)
- **System prompt:** A brief placeholder — you'll refine it after scraping

Then, if you have a website URL, scrape it for knowledge and brand signals:

Call `discover_pages(agent_id, url)` to find all pages on the domain, then `scrape_website_batch(agent_id, urls)` or `scrape_website(agent_id, url)` for the **most important pages** (cap at 7):
1. Homepage (always)
2. About / About Us
3. Services / What We Do
4. Pricing / Plans
5. FAQ / Help
6. Contact / Book / Schedule

While scraping, extract:
- **Business name, industry, services, tone, location, hours, phone**
- **Primary color** — from meta theme-color, buttons, or use industry defaults:
  - Dental: `#4A90D9`, Gym: `#FF6B35`, Legal: `#1B365D`, Restaurant: `#C41E3A`, Plumbing/HVAC: `#2E86AB`, Cleaning: `#27AE60`, Med spa: `#9B59B6`, Real estate: `#E74C3C`
- **Logo URL** — from og:image, favicon, or header image

Now refine the agent with scraped knowledge. Call `update_agent` to set the full system prompt — include business name, services, tone, greeting, key facts, and a "demo mode" instruction: when asked to book/schedule, say "In the full version, I'll be connected to your booking system to handle this automatically."

Generate 5-10 FAQ pairs from the scraped content (services, pricing, hours, location, etc.) and call `add_faq_batch(agent_id, faqs)` to add them all at once.

## Step 3: Choose the campaign and channel

**Default to using an existing campaign.** Call `list_campaigns` to see what's available.

Ask: "Which campaign do you want to connect to this demo page? For example:
- A **widget campaign** — visitors chat with the agent on the page
- A **WhatsApp campaign** — visitors see a WhatsApp link or QR code
- Or I can **create a new campaign** for this demo"

List any existing campaigns and their channel types so the user can pick.

- If the user picks an existing campaign → call `get_campaign` to read its config, then call `get_embed_code` to get the embed snippet containing the `data-channel` value. Continue to Step 4.
- If the user wants a new campaign → go to Step 3b

### Step 3b: Create a new campaign (only if requested)

1. Call `list_clients` to check if the prospect's client account already exists
2. If not, call `create_client` with the prospect's business name and website
3. Call `create_campaign` with the chosen `channel_type` (`"widget"` or `"whatsapp"`)
4. If widget: call `configure_widget` with:
   - `primary_color`: the brand color you found
   - `agent_avatar`: logo URL or industry emoji
   - `agent_name`: business name
   - `welcome_message`: personalized greeting
   - `conversation_starters`: 3-4 customer questions based on their services, max 4 (e.g., "Book a cleaning", "Do you accept my insurance?", "What are your hours?")
   - `show_branding`: `false`
   - `pre_chat_form`: `{ enabled: false, fields: [] }` (no friction for demo)
5. Call `update_campaign(campaign_id, status: "active")`
6. Call `get_embed_code(campaign_id)` — save the `data-channel` value from the embed snippet

### Step 3c: WhatsApp pipeline setup (only for WhatsApp campaigns)

If the selected campaign has `channel_type: "whatsapp"`, set up the automated lead pipeline. If the campaign is widget-only, skip this step entirely.

#### Check readiness

Call `check_wa_readiness(campaign_id)`. If Meta credentials aren't configured, tell the user:
"Your WhatsApp campaign isn't ready — Meta credentials need to be configured in the LaunchPath dashboard first."
Offer to proceed with widget-only while they set up WhatsApp. If readiness passes, continue.

#### Check for existing drip sequences

Call `list_wa_sequences(campaign_id)`.

If active sequences exist:
  Show them: "[Name] — [N] steps, status: [active/paused]"
  Ask: "Want to use this existing sequence for demo leads, or create a new one?"
  If using existing → note the sequence_id, skip to "Save campaign details" below

If no sequences exist, or user wants a new one:
  Ask: "Do you want leads from this demo page to automatically receive WhatsApp messages?
  I can set up a welcome drip:
  1. Welcome message (sent immediately when they submit the form)
  2. Follow-up (24 hours later)
  3. Booking nudge (72 hours later)

  Or you can customize the timing and content. What do you prefer?"

#### Check/create templates

Call `list_wa_templates(campaign_id, status: "APPROVED")`.

If approved templates exist, show them and ask which to use for each drip step.

If no approved templates:
  "You need at least one approved WhatsApp template. I'll create a welcome template now — Meta takes 1-24 hours to approve."

  Create a welcome template:
    `create_wa_template(campaign_id, name: "demo_welcome", category: "UTILITY", language: "en_US", body: "Hi {{1}}, thanks for your interest in [Business Name]! Our AI assistant is ready to help you with [service]. Reply here to start a conversation, or we'll follow up with more details soon.")`

  Tell the user: "Template submitted for approval. The drip will start working once Meta approves it (usually 1-24 hours). I'll set everything else up now. Note: if approval takes longer than 3 days, enrolled contacts will be marked as failed and you'll need to re-enroll them. Check template status with `list_wa_templates(campaign_id)` — once approved, new leads will flow through automatically."

#### Create the drip sequence

Call `create_wa_sequence` with:
- `name`: "[Business Name] Demo Leads"
- `steps`: based on available templates
  - Step 1: template (delay: 0 min) — welcome
  - Step 2: template (delay: 1440 min / 24h) — follow-up (if second template available)
  - Step 3: template (delay: 4320 min / 72h) — booking nudge (if third template available)
- `auto_enroll_on_import: true`
- `auto_enroll_on_ingest_tags: ["demo-lead"]` — only contacts tagged "demo-lead" enter this drip
- `stop_on_reply: true`

If only 1 template is available, create a single-step sequence. The user can add more steps later.

#### Activate the sequence

Call `activate_wa_sequence(campaign_id, sequence_id, status: "active")`.

#### Save campaign details for form wiring

Note the following for Step 5 (page generation):
- `campaign_id` — for the ingest URL: `https://www.trylaunchpath.com/api/campaigns/{campaign_id}/contacts/ingest`
- Channel token — retrieved from campaign data via `get_campaign(campaign_id)`, extract from channels[0].token. This is the Bearer token for the ingest API.
- These go into `.env.local` as environment variables (LP_INGEST_URL and LP_INGEST_TOKEN).

## Step 4: Scrape for brand signals (if not already done)

If you have a website URL and didn't scrape in Step 2b (because the user chose an existing agent), scrape the **homepage only** using `scrape_website(agent_id, url)` to extract brand signals for the landing page (note: this adds the page to the existing agent's knowledge base):
- Business name, industry, services, tone
- Primary color, logo URL
- Key selling points for the landing page copy

This is NOT for training the agent — it's for customizing the landing page copy and colors.

## Step 5: Generate the landing page (the core of this skill)

### Ensure the template exists

Check if a `DemoTemplate.tsx` file exists in the user's project (e.g., `src/components/demo/DemoTemplate.tsx` or similar).
- If it **doesn't exist** → write it using the Reference Template at the bottom of this skill file
- If it **already exists** from a previous demo → skip, reuse it

### Create the demo page

Create `src/app/demo/[prospect-name]/page.tsx` — a short file (~50-80 lines) that imports DemoTemplate and passes a customized config object. Do NOT copy the full template — import it.

```tsx
"use client";
import DemoTemplate from "@/components/demo/DemoTemplate";

export default function ProspectDemoPage() {
  return (
    <DemoTemplate
      config={{
        // Brand
        businessName: "[Business Name]",
        primaryColor: "[brand color or industry default]",
        logoUrl: "[logo URL if found]",

        // Copy — rewrite ALL of these for the prospect's industry
        headline: "[Industry-specific headline, e.g. 'Never Miss a Patient Call Again']",
        subheadline: "[Industry-specific subheadline]",
        announcementBar: "[Urgency message for their industry]",
        painPoint: "[PAS agitation copy for their industry — the pain they feel daily]",

        // Value props — 3 cards tailored to their services
        valueProps: [
          { title: "...", boldLine: "...", description: "...", stat: "..." },
          { title: "...", boldLine: "...", description: "...", stat: "..." },
          { title: "...", boldLine: "...", description: "...", stat: "..." },
        ],

        // Form — service options for THEIR industry
        serviceOptions: [
          { value: "service1", label: "[Their Service 1]" },
          { value: "service2", label: "[Their Service 2]" },
          { value: "service3", label: "[Their Service 3]" },
        ],

        // FAQ — from scraped content or user's description
        faq: [
          { question: "[Q1]", answer: "[A1]" },
          { question: "[Q2]", answer: "[A2]" },
          { question: "[Q3]", answer: "[A3]" },
        ],

        // Widget — from get_embed_code
        widgetChannelId: "[CHANNEL_ID]",

        // Form submission — see WhatsApp section below
        // If WhatsApp connected: set onFormSubmit + success text
        // If widget-only: omit onFormSubmit (uses default simulation)
      }}
    />
  );
}
```

### Content guidelines

Every config field should be customized. Do NOT rely on the template's built-in defaults:
- **headline**: Industry-specific benefit, not generic "Turn Missed Calls Into Booked Jobs"
- **painPoint**: Rewrite for their business — a dental practice losing patients is different from a plumber losing jobs
- **valueProps**: 3 cards matching their actual services
- **serviceOptions**: Their real service categories
- **faq**: From scraped content or user description, addressing prospect objections
- **primaryColor**: From scraping or industry defaults (see Step 1 no-URL fallback)

### WhatsApp lead capture (when Step 3c completed)

When a WhatsApp campaign is connected, also generate a server-side API route at `src/app/demo/[prospect-name]/api/ingest/route.ts`:

```typescript
import { NextRequest, NextResponse } from "next/server";

// Note: The ingest API normalizes phone numbers automatically (e.g., (555) 000-0000 → +15550000000)
// but E.164 format is preferred. If submissions fail with phone errors, add client-side normalization.
export async function POST(req: NextRequest) {
  const INGEST_URL = process.env.LP_INGEST_URL;
  const INGEST_TOKEN = process.env.LP_INGEST_TOKEN;

  if (!INGEST_URL || !INGEST_TOKEN) {
    return NextResponse.json({ error: "WhatsApp ingest not configured" }, { status: 500 });
  }

  const { name, businessName, industry, phone } = await req.json();
  if (!phone) {
    return NextResponse.json({ error: "Phone number is required" }, { status: 400 });
  }

  const res = await fetch(INGEST_URL, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${INGEST_TOKEN}`,
    },
    body: JSON.stringify({
      contacts: [{
        phone,
        name: name || undefined,
        tags: ["demo-lead"],
        source: "demo-page",
        custom_fields: {
          business_name: businessName || undefined,
          industry: industry || undefined,
        },
      }],
    }),
  });

  if (!res.ok) {
    return NextResponse.json({ error: "Failed to submit" }, { status: 502 });
  }
  return NextResponse.json({ success: true });
}
```

Then in the page.tsx config, add:

```tsx
onFormSubmit: async (data) => {
  const res = await fetch("/demo/[prospect-name]/api/ingest", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(data),
  });
  if (!res.ok) throw new Error("Submission failed");
},
successHeadline: "Check your WhatsApp!",
successDescription: "You'll receive a message from [Business Name] shortly. Reply to start a conversation with our AI assistant.",
```

**Token security:** `LP_INGEST_TOKEN` is NEVER in client-side code — only the server-side route reads it from env vars.

**When WhatsApp is NOT connected** (widget-only): Omit `onFormSubmit` from the config. The template's default simulates a 1.5s delay then shows the success screen.

## Step 6: Present the result

```
DEMO PAGE READY: [Business Name]

Agent: [agent name] (trained on [N] pages, [N] FAQs)
Campaign: [campaign name] (widget active)
Page: [file path]

What's on the page:
  - Branded landing page with [Business Name]'s colors and messaging
  - Live chat widget (bottom corner) — prospects can talk to the AI instantly
  - Lead capture form — collects name, phone, industry
  - Industry-specific copy, stats, and FAQs

To preview: run `npm run dev` and visit [route]

CUSTOMIZATION:
The page is a React component you fully control. Edit anything:
  - Copy, layout, colors, sections
  - Add/remove form fields
  - Change the value props, stats, testimonials
  - Swap the FAQ content

WHATSAPP PIPELINE: [Active / Not configured]
  Ingest URL: [URL from Step 3c]
  Auto-enrollment: Tag-based ("demo-lead")
  Drip sequence: "[Name]" — [N] steps, stop on reply
  Templates: [N] approved, [N] pending approval

  Add to .env.local:
    LP_INGEST_URL=https://www.trylaunchpath.com/api/campaigns/{campaignId}/contacts/ingest
    LP_INGEST_TOKEN=lp_ch_... (from campaign channel)

  IMPORTANT: Never commit the token to version control.

  If WhatsApp was not set up (widget-only), show instead:
  WHATSAPP: Not configured. To add WhatsApp later:
  1. Create a WhatsApp campaign with Meta credentials
  2. Run /launchpath:whatsapp-compliance to set up templates
  3. Re-run /launchpath:live-demo and select the WhatsApp campaign

NEXT STEPS (when prospect converts):
1. This demo agent becomes their production agent — no rebuilding
2. Upgrade the model if needed (claude-sonnet for complex conversations)
3. Add integrations (Google Calendar, CRM, etc.)
4. Scrape more pages for deeper knowledge
5. Run /launchpath:stress-test before going live
```

## Important Rules

- **Use existing agents and campaigns by default.** Only create new ones if the user explicitly asks. Always call `list_agents` and `list_campaigns` first.
- **Ask which agent and which campaign.** Don't assume — let the user pick from their existing resources.
- **Speed matters.** Don't ask unnecessary questions beyond agent/campaign selection. Get the URL, scrape for brand signals, generate the page.
- **Cap scraping at 7 pages.** Agency can add more knowledge later.
- **Use gpt-4o-mini for new demo agents.** Cheap and fast. Upgrade for production.
- **The demo agent IS the production agent.** If the prospect signs on, the agency just upgrades — no rebuilding.
- **Don't test automatically.** `chat_with_agent` costs credits. The widget on the page IS the test.
- **Import, don't copy.** Write the DemoTemplate once (from the Reference Template below), then each demo page is a short import+config file. Never copy 500 lines of template code into each page.
- **WhatsApp pipeline:** When a WhatsApp campaign is connected, the generated API route proxies form submissions server-side to the LaunchPath ingest API. Token stays in env vars, never in client code. When WhatsApp is NOT connected, the form simulates submission — the agency decides where to store leads later.
- **Conversation starters should be CUSTOMER questions.** "Book a cleaning" not "We offer cleanings."
- **The system prompt should include a "demo mode" note** (for new agents only). When asked about booking/scheduling, the agent says it can do this in the full version. Natural upsell moment.

---

## Reference Template: DemoTemplate.tsx

The following is the DemoTemplate component that powers all demo landing pages. When generating a demo page, first check if `DemoTemplate.tsx` exists in the user's project (e.g., at `src/components/demo/DemoTemplate.tsx`). If it doesn't exist, create it with the code below. If it already exists from a previous demo, skip this step and reuse it.

**Dependencies required:** `react`, `next/link`, `lucide-react`. The user's Next.js project must have these installed. If `@/components/Logo` doesn't exist, replace the Logo import with a simple text span showing "LaunchPath".

```tsx
"use client";

import React, { useEffect, useRef, useState } from "react";
import Link from "next/link";
import { ChevronDown, Check, ArrowRight, PhoneOff, Clock, UserMinus, ShieldCheck, PlayCircle, Zap, Star } from "lucide-react";
import { Logo } from "@/components/Logo";

// --- MINIMAL FADE-IN WRAPPER ---
function FadeIn({
    children,
    delay = 0,
    direction = "up"
}: {
    children: React.ReactNode;
    delay?: number;
    direction?: "up" | "down" | "left" | "right" | "none";
}) {
    const ref = useRef<HTMLDivElement>(null);
    const [isVisible, setIsVisible] = useState(false);

    useEffect(() => {
        const observer = new IntersectionObserver(
            ([entry]) => {
                if (entry.isIntersecting) {
                    setIsVisible(true);
                    observer.unobserve(entry.target);
                }
            },
            { threshold: 0.1, rootMargin: "0px 0px -40px 0px" }
        );
        if (ref.current) observer.observe(ref.current);
        return () => observer.disconnect();
    }, []);

    const translateClass =
        direction === "up" ? "translate-y-6" :
            direction === "down" ? "-translate-y-6" :
                direction === "left" ? "translate-x-6" :
                    direction === "right" ? "-translate-x-6" : "translate-y-0 translate-x-0";

    return (
        <div
            ref={ref}
            style={{ transitionDelay: `${delay}ms` }}
            className={`transition-all duration-700 ease-out fill-mode-forwards ${isVisible ? "opacity-100 translate-y-0 translate-x-0" : `opacity-0 ${translateClass}`
                }`}
        >
            {children}
        </div>
    );
}

export interface DemoConfig {
    // Brand
    businessName?: string;
    primaryColor?: string;
    logoUrl?: string;

    // Copy
    headline?: string;
    subheadline?: string;
    announcementBar?: string;
    painPoint?: string;

    // Value props (the 3 cards)
    valueProps?: {
        title: string;
        boldLine: string;
        description: string;
        stat: string;
    }[];

    // Form
    serviceOptions?: { value: string; label: string }[];
    consentText?: string;
    ctaText?: string;
    successHeadline?: string;
    successDescription?: string;

    // FAQ
    faq?: { question: string; answer: string }[];

    // Widget
    widgetChannelId?: string;

    // Callbacks
    onFormSubmit?: (data: { name: string; businessName: string; industry: string; phone: string }) => void | Promise<void>;
}

export default function DemoTemplate({ config }: { config?: DemoConfig }) {
    const [isConsentChecked, setIsConsentChecked] = useState(false);
    const [isSubmitting, setIsSubmitting] = useState(false);
    const [isSuccess, setIsSuccess] = useState(false);

    const primaryColor = config?.primaryColor || "#4F46E5";
    const businessName = config?.businessName;

    // Form state
    const [formData, setFormData] = useState({
        name: "",
        businessName: "",
        industry: "",
        phone: ""
    });

    const serviceOptions = config?.serviceOptions || [];

    const faqs = config?.faq || [
        {
            question: "Will the AI sound robotic to my customers?",
            answer: "No. Hear it for yourself above. Our conversational models use natural language, understand context, and adapt to your specific business tone."
        },
        {
            question: "What happens to the information I submit here?",
            answer: "Your details exist purely to run this live, personalized demo. They are securely processed and never sold to third parties."
        },
        {
            question: "How much does a system like this cost?",
            answer: "Experiencing this demo is 100% free. If you like the results we deliver on your phone right now, we can discuss a customized pricing plan tailored to your lead volume."
        }
    ];

    // Inject widget script when channelId is provided
    useEffect(() => {
        if (!config?.widgetChannelId) return;
        const script = document.createElement("script");
        script.src = `${window.location.origin}/widget.js`;
        script.dataset.channel = config.widgetChannelId;
        script.async = true;
        document.body.appendChild(script);
        return () => {
            try { document.body.removeChild(script); } catch { /* already removed */ }
        };
    }, [config?.widgetChannelId]);

    const handleDemoSubmit = async (e: React.FormEvent) => {
        e.preventDefault();
        if (!isConsentChecked) return;

        setIsSubmitting(true);

        try {
            if (config?.onFormSubmit) {
                await config.onFormSubmit(formData);
            } else {
                // Default: simulate a short delay
                await new Promise((r) => setTimeout(r, 1500));
            }
            setIsSuccess(true);
        } catch {
            // onFormSubmit failed �� don't show success screen
        } finally {
            setIsSubmitting(false);
        }
    };

    return (
        <div className="min-h-screen bg-[#FDFDFD] text-[#0F172A] font-sans flex flex-col justify-start w-full relative overflow-x-hidden" style={{ ["--selection-color" as string]: primaryColor }}>

            {/* --- INJECTED STYLE --- */}
            <style dangerouslySetInnerHTML={{
                __html: `
        :root {
          --primary: ${primaryColor};
          --primary-hover: ${primaryColor}CC;
        }

        ::selection {
          background-color: ${primaryColor};
          color: white;
        }

        /* Subtle noise texture for premium tactile feel */
        .bg-noise {
          background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 200 200' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='noiseFilter'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.85' numOctaves='3' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23noiseFilter)'/%3E%3C/svg%3E");
          opacity: 0.02;
          pointer-events: none;
        }
      `}} />

            {/* --- ATTENTION ANNOUNCEMENT BAR --- */}
            <div className="w-full bg-[var(--primary)] text-white text-center py-2.5 px-4 text-xs sm:text-sm font-medium z-50 relative shadow-md flex items-center justify-center">
                <span className="flex items-center gap-2">
                    <span className="flex h-2 w-2 relative shrink-0">
                        <span className="animate-ping absolute inline-flex h-full w-full rounded-full bg-white opacity-75"></span>
                        <span className="relative inline-flex rounded-full h-2 w-2 bg-white"></span>
                    </span>
                    <span>{config?.announcementBar || <><strong className="font-extrabold hidden sm:inline">Limited Availability:</strong> Accepting 5 new local business partners this week.</>}</span>
                </span>
            </div>

            {/* --- BACKGROUND FX --- */}
            <div className="absolute inset-0 bg-noise z-0 mix-blend-overlay mt-10" />
            <div className="absolute top-10 right-0 w-[50vw] h-[500px] bg-[var(--primary)]/5 blur-[120px] rounded-bl-full pointer-events-none -z-10" />

            {/* NO NAVIGATION BAR - One page, One action (Lesson 7: Focus) */}

            {/* --- HERO SPLIT (Mobile Visual Hierarchy Optimized) --- */}
            {/* Used CSS Grid to ensure the Form (Col 2) drops neatly below the Headline (Col 1, Row 1) but ABOVE the Agitation/Social Proof (Col 1, Row 2) on mobile. */}
            <main className="w-full max-w-6xl mx-auto px-5 pt-8 pb-16 md:pt-16 md:pb-24 grid grid-cols-1 lg:grid-cols-2 lg:gap-x-16 gap-y-10 relative z-10">

                {/* ROW 1 LEFT: HEADLINE (Always visible first) */}
                <div className="flex flex-col justify-start lg:pr-4">
                    {/* 5. Borrowed Social Proof / Industry Stat (For new agencies) */}
                    <FadeIn>
                        <div className="flex items-center gap-3 mb-6">
                            <div className="flex -space-x-2 opacity-80">
                                <div className="w-8 h-8 rounded-full border-2 border-white bg-blue-100 flex items-center justify-center text-blue-600 shadow-sm"><Zap className="w-4 h-4 fill-current" /></div>
                                <div className="w-8 h-8 rounded-full border-2 border-white bg-emerald-100 flex items-center justify-center text-emerald-600 shadow-sm"><Clock className="w-4 h-4" /></div>
                                <div className="w-8 h-8 rounded-full border-2 border-white bg-indigo-100 flex items-center justify-center text-indigo-600 shadow-sm"><ShieldCheck className="w-4 h-4" /></div>
                            </div>
                            <div className="flex items-center gap-1.5 text-sm font-semibold text-slate-700">
                                <div className="flex gap-0.5">
                                    {[1, 2, 3, 4, 5].map(i => <Star key={i} className="w-3.5 h-3.5 text-amber-500 fill-current" />)}
                                </div>
                                <span className="text-[13px] md:text-sm">Instant response increases bookings by <strong className="text-slate-900">up to 391%*</strong></span>
                            </div>
                        </div>
                    </FadeIn>

                    <FadeIn delay={100}>
                        {/* 1. Benefit-Driven Headline: Unified outome and mechanism */}
                        <h1 className="text-[2.8rem] sm:text-[3.2rem] md:text-[3.8rem] font-extrabold tracking-tighter text-[#0F172A] leading-[1.05] mb-5">
                            {config?.headline || <>Turn Missed Calls Into <span className="text-[var(--primary)] inline-block relative">Booked Jobs<svg className="absolute w-full h-3 -bottom-1 left-0 text-[var(--primary)]/30" viewBox="0 0 100 10" preserveAspectRatio="none"><path d="M0 5 Q 50 10 100 5" stroke="currentColor" strokeWidth="3" fill="transparent" /></svg></span> With AI.</>}
                        </h1>
                    </FadeIn>

                    <FadeIn delay={150}>
                        {/* 1. Sub-headline: USP hook emphasized */}
                        <p className="text-lg md:text-xl text-slate-600 mb-2 max-w-xl leading-relaxed">
                            {config?.subheadline || <><strong className="text-slate-900 font-bold block mb-1.5 text-[1.1rem] md:text-[1.25rem]">Experience our AI answering your phones live right now.</strong>
                            It takes exactly <span className="font-bold text-[var(--primary)] bg-[var(--primary)]/10 px-1.5 py-0.5 rounded">60 seconds</span> to try the interactive demo below.</>}
                        </p>
                    </FadeIn>
                </div>

                {/* RIGHT (or direct center on Mobile): THE CONVERSION FORM (Lesson 6: Visual Hierarchy) */}
                {/* Spans 2 rows on large screens so it sits adjacent to both the headline AND the social proof below it. */}
                <div className="lg:row-span-2 relative w-full lg:max-w-md xl:max-w-lg mx-auto lg:mx-0 lg:ml-auto" id="demo-start">
                    <FadeIn delay={150} direction="left">

                        <div className="bg-white rounded-[24px] shadow-[0_20px_60px_rgba(0,0,0,0.08)] border border-slate-200 relative overflow-hidden flex flex-col">

                            {/* Form Header */}
                            <div className="bg-slate-50 border-b border-slate-100 p-6 md:px-8 py-6 flex items-center justify-between">
                                <div>
                                    <h3 className="text-xl font-bold text-[#0F172A] tracking-tight">Experience Your 60-Second Demo</h3>
                                    <p className="text-[13px] md:text-sm font-medium text-slate-500 mt-0.5">Tell us about your business so the AI can tailor the conversation.</p>
                                </div>
                                <div className="w-10 h-10 bg-[var(--primary)]/10 text-[var(--primary)] rounded-full flex items-center justify-center shrink-0">
                                    <PlayCircle className="w-5 h-5" fill="currentColor" stroke="white" />
                                </div>
                            </div>

                            {/* Dynamic State: Form vs Success */}
                            {!isSuccess ? (
                                <form className="p-6 md:p-8 space-y-4 md:space-y-5 flex-grow" onSubmit={handleDemoSubmit}>

                                    {/* Minimized Fields - Max 4 (Lesson 7: Simplicity) */}
                                    <div className="space-y-1.5">
                                        <label className="text-[13px] font-bold text-slate-700">Business Name</label>
                                        <input
                                            type="text"
                                            required
                                            value={formData.businessName}
                                            onChange={(e) => setFormData({ ...formData, businessName: e.target.value })}
                                            placeholder="e.g. Acme Plumbing"
                                            className="w-full bg-white border border-slate-300 text-slate-900 rounded-xl px-4 py-3 md:py-3.5 min-h-[44px] md:min-h-[48px] text-[15px] md:text-[16px] font-medium placeholder:text-slate-400 focus:outline-none focus:border-[var(--primary)] focus:ring-[3px] focus:ring-[var(--primary)]/20 transition-all shadow-sm"
                                        />
                                    </div>

                                    <div className="space-y-1.5">
                                        <label className="text-[13px] font-bold text-slate-700">What service do you provide?</label>
                                        {serviceOptions.length > 0 ? (
                                            <div className="relative">
                                                <select
                                                    required
                                                    value={formData.industry}
                                                    onChange={(e) => setFormData({ ...formData, industry: e.target.value })}
                                                    className="w-full bg-white border border-slate-300 text-slate-900 rounded-xl pl-4 pr-10 py-3 md:py-3.5 min-h-[44px] md:min-h-[48px] text-[15px] md:text-[16px] font-medium appearance-none focus:outline-none focus:border-[var(--primary)] focus:ring-[3px] focus:ring-[var(--primary)]/20 transition-all shadow-sm"
                                                >
                                                    <option value="" disabled hidden>Select your service...</option>
                                                    {serviceOptions.map((opt) => (
                                                        <option key={opt.value} value={opt.value}>{opt.label}</option>
                                                    ))}
                                                </select>
                                                <ChevronDown className="absolute right-4 top-1/2 -translate-y-1/2 w-5 h-5 text-slate-400 pointer-events-none" />
                                            </div>
                                        ) : (
                                            <input
                                                type="text"
                                                required
                                                value={formData.industry}
                                                onChange={(e) => setFormData({ ...formData, industry: e.target.value })}
                                                placeholder="e.g. Dental, Legal, Cleaning..."
                                                className="w-full bg-white border border-slate-300 text-slate-900 rounded-xl px-4 py-3 md:py-3.5 min-h-[44px] md:min-h-[48px] text-[15px] md:text-[16px] font-medium placeholder:text-slate-400 focus:outline-none focus:border-[var(--primary)] focus:ring-[3px] focus:ring-[var(--primary)]/20 transition-all shadow-sm"
                                            />
                                        )}
                                    </div>

                                    {/* Forced stack on mobile using grid-cols-1 */}
                                    <div className="grid grid-cols-1 sm:grid-cols-2 gap-4 md:gap-5">
                                        <div className="space-y-1.5">
                                            <label className="text-[13px] font-bold text-slate-700">Your First Name</label>
                                            <input
                                                type="text"
                                                required
                                                value={formData.name}
                                                onChange={(e) => setFormData({ ...formData, name: e.target.value })}
                                                placeholder="John"
                                                className="w-full bg-white border border-slate-300 text-slate-900 rounded-xl px-4 py-3 md:py-3.5 min-h-[44px] md:min-h-[48px] text-[15px] md:text-[16px] font-medium placeholder:text-slate-400 focus:outline-none focus:border-[var(--primary)] focus:ring-[3px] focus:ring-[var(--primary)]/20 transition-all shadow-sm"
                                            />
                                        </div>
                                        <div className="space-y-1.5">
                                            <label className="text-[13px] font-bold text-slate-700">Mobile Phone Number</label>
                                            <input
                                                type="tel"
                                                required
                                                value={formData.phone}
                                                onChange={(e) => setFormData({ ...formData, phone: e.target.value })}
                                                placeholder="(555) 000-0000"
                                                className="w-full bg-white border border-slate-300 text-slate-900 rounded-xl px-4 py-3 md:py-3.5 min-h-[44px] md:min-h-[48px] text-[15px] md:text-[16px] font-medium placeholder:text-slate-400 focus:outline-none focus:border-[var(--primary)] focus:ring-[3px] focus:ring-[var(--primary)]/20 transition-all shadow-sm"
                                            />
                                        </div>
                                    </div>

                                    {/* High Trust Consent */}
                                    <div className="flex items-start gap-3 md:mt-2 pt-2 pb-2">
                                        <button
                                            type="button"
                                            className={`mt-0.5 min-w-[20px] min-h-[20px] w-5 h-5 rounded-md flex shrink-0 items-center justify-center transition-all border shadow-[0_1px_2px_rgba(0,0,0,0.05)] ${isConsentChecked ? "bg-[var(--primary)] border-[var(--primary)]" : "bg-white border-slate-300 hover:border-slate-400"
                                                }`}
                                            onClick={() => setIsConsentChecked(!isConsentChecked)}
                                        >
                                            {isConsentChecked && <Check className="w-3.5 h-3.5 text-white" strokeWidth={3.5} />}
                                        </button>
                                        <p className="text-[11px] md:text-[12px] text-slate-500 font-medium leading-[1.5]">
                                            {config?.consentText || "I consent to receive a one-time live demo text to the number above. Reply STOP to end."}
                                        </p>
                                    </div>

                                    {/* 1. Large Contrasting CTA Button (Lesson 6: Visual Hierarchy) */}
                                    <button
                                        disabled={isSubmitting || !isConsentChecked}
                                        className="w-full py-4.5 md:py-4.5 rounded-xl font-bold flex items-center justify-center transition-all duration-300 shadow-[0_10px_20px_rgba(79,70,229,0.25)] hover:shadow-[0_10px_25px_rgba(79,70,229,0.35)] active:scale-[0.98] mt-2 min-h-[56px] md:min-h-[60px] disabled:opacity-70 disabled:cursor-not-allowed group relative overflow-hidden"
                                        style={{ backgroundColor: "var(--primary)" }}
                                    >
                                        <div className="absolute inset-0 bg-white/20 translate-y-full group-hover:translate-y-0 transition-transform duration-300 ease-in-out" />
                                        <span className="text-white text-[16px] md:text-[19px] relative z-10">
                                            {isSubmitting ? "Connecting to AI..." : (config?.ctaText || "Start My Free Demo Now")}
                                        </span>
                                    </button>

                                    {/* FUD Reducer explicitly under CTA (Lesson 1) */}
                                    <div className="text-center w-full mt-3 space-y-1.5 px-2">
                                        <p className="text-[12px] md:text-[13px] font-bold text-slate-700 flex items-center justify-center gap-1.5"><ShieldCheck className="w-4 h-4 text-emerald-500" /> 100% Free Demo. No commitment.</p>
                                        <p className="text-[11px] md:text-[12px] text-slate-500 font-medium leading-tight text-balance">We will never spam you. Your details are securely used <strong className="text-slate-600">only once</strong> to run this personalized live demonstration.</p>
                                    </div>

                                </form>
                            ) : (
                                // --- POST FORM EXPERIENCE (The Magic Moment) ---
                                <div className="p-8 md:p-10 flex flex-col items-center justify-center text-center flex-grow bg-slate-50/50">
                                    <div className="w-16 h-16 md:w-20 md:h-20 rounded-full bg-emerald-100 flex items-center justify-center mb-6">
                                        <Check className="w-8 h-8 md:w-10 md:h-10 text-emerald-600" strokeWidth={4} />
                                    </div>
                                    <h3 className="text-2xl md:text-3xl font-bold text-slate-900 mb-2 md:mb-3 tracking-tight">{config?.successHeadline || "The AI is texting you now."}</h3>
                                    <p className="text-slate-600 font-medium text-base md:text-lg mb-8">
                                        {config?.successDescription || <><strong className="text-slate-900">{formData.name || "There"}</strong>, please check your mobile device at <strong className="text-slate-900">{formData.phone}</strong>.</>}
                                    </p>

                                    <div className="bg-white p-5 md:p-6 rounded-xl border border-slate-200 text-sm font-medium text-slate-600 text-left w-full shadow-sm">
                                        <p className="text-slate-800 font-bold text-[15px] md:text-base mb-3 flex items-center gap-2"><Zap className="w-4 h-4 md:w-5 md:h-5 text-amber-500 fill-current" /> Next Steps to experience the power:</p>
                                        <ul className="space-y-3 list-none pl-1 text-[13px] md:text-[14px]">
                                            <li className="flex items-start gap-2"><span className="text-[var(--primary)] font-bold">1.</span> Reply to the text message.</li>
                                            <li className="flex items-start gap-2"><span className="text-[var(--primary)] font-bold">2.</span> Ask it any hard questions about {formData.industry === "other" || !formData.industry ? "your business" : formData.industry}.</li>
                                            <li className="flex items-start gap-2"><span className="text-[var(--primary)] font-bold">3.</span> Instruct it to schedule an appointment with you.</li>
                                        </ul>
                                    </div>
                                </div>
                            )}
                        </div>
                    </FadeIn>
                </div>

                {/* ROW 2 LEFT: AGITATION & SOCIAL PROOF (Sits below the form on mobile) */}
                <div className="flex flex-col justify-start lg:pr-4">
                    {/* 2. Emotional Agitation / PAS Framework (Lesson 2) */}
                    <FadeIn delay={250}>
                        <div className="bg-slate-50 border border-slate-200 rounded-2xl p-5 md:p-6 max-w-lg relative shadow-sm">
                            <div className="absolute -left-2 top-6 w-1 h-12 bg-rose-500 rounded-r-md"></div>
                            <p className="text-[14px] md:text-[15px] text-slate-700 leading-relaxed font-medium">
                                {config?.painPoint || <><strong className="text-slate-900 block mb-1">We know the feeling:</strong>
                                You&apos;re working a job site, you miss a call, and lose the lead to voicemail. By the time you call them back 3 hours later, <strong className="text-slate-900">they&apos;ve already paid your competitor</strong> who answered immediately.</>}
                            </p>
                        </div>
                    </FadeIn>
                </div>

            </main>

            {/* --- SCANNABLE VALUE PROP / HOW IT WORKS (Lesson 4: People Scan, They Don't Read & Lesson 3) --- */}
            <section className="w-full bg-[#f8fafc] py-16 md:py-28 border-y border-slate-200 shadow-sm relative z-0 mt-8 md:mt-0">
                <div className="max-w-6xl mx-auto px-5">
                    <div className="text-center md:text-left mb-12 md:mb-16 max-w-2xl">
                        <FadeIn>
                            {/* 4. Bold Benefit-Driven Headline (Instead of "How it works") */}
                            <h2 className="text-[1.75rem] md:text-[2.5rem] font-extrabold tracking-tight text-[#0F172A] mb-4 leading-tight">Three things it handles while you're on the job.</h2>
                            <p className="text-slate-600 md:text-lg font-medium">Your business doesn't stop, so neither should your reception. Here is exactly what the system achieves for you in the background.</p>
                        </FadeIn>
                    </div>

                    <div className="grid md:grid-cols-3 gap-8 md:gap-12 text-left">
                        <FadeIn delay={100} direction="up">
                            <div className="flex flex-col group h-full bg-white p-6 md:p-8 rounded-2xl border border-slate-200 shadow-sm hover:shadow-md transition-shadow">
                                <div className="w-12 h-12 bg-blue-50 text-blue-600 rounded-xl flex items-center justify-center mb-6 border border-blue-100">
                                    <PlayCircle className="w-6 h-6" />
                                </div>
                                {/* Formatting for scanning: Bold core value */}
                                <h3 className="text-lg md:text-xl font-bold text-slate-900 mb-3"><span className="text-[var(--primary)] text-bold">Sub-second</span> Engagement</h3>

                                <p className="font-bold text-slate-900 text-[15px] mb-2 leading-snug">Hook prospects before they call your competitors.</p>
                                <p className="text-slate-600 font-medium text-[14px] leading-relaxed mb-6">When a prospect texts you, the AI replies instantly. It starts a polite conversation the moment their text hits your number.</p>

                                <div className="mt-auto bg-slate-50 border border-slate-100 rounded-lg p-3 text-[13px] font-bold text-slate-700 flex items-center gap-2">
                                    <Zap className="w-4 h-4 text-blue-500 fill-blue-500" /> Average response time: 0.8 seconds
                                </div>
                            </div>
                        </FadeIn>

                        <FadeIn delay={200} direction="up">
                            <div className="flex flex-col group h-full bg-white p-6 md:p-8 rounded-2xl border border-slate-200 shadow-sm hover:shadow-md transition-shadow">
                                <div className="w-12 h-12 bg-amber-50 text-amber-600 rounded-xl flex items-center justify-center mb-6 border border-amber-100">
                                    <ShieldCheck className="w-6 h-6" />
                                </div>
                                <h3 className="text-lg md:text-xl font-bold text-slate-900 mb-3"><span className="text-[var(--primary)] text-bold">Ruthless</span> Qualification</h3>

                                <p className="font-bold text-slate-900 text-[15px] mb-2 leading-snug">Stop wasting time on dead-end tire kickers.</p>
                                <p className="text-slate-600 font-medium text-[14px] leading-relaxed mb-6">It asks the budget, timeline, and scope questions you require. If they aren't a fit, it politely turns them away so you only talk to perfect leads.</p>

                                <div className="mt-auto bg-slate-50 border border-slate-100 rounded-lg p-3 text-[13px] font-bold text-slate-700 flex items-center gap-2">
                                    <ShieldCheck className="w-4 h-4 text-amber-500 fill-amber-500" /> Only 1 in 4 leads makes it to your calendar
                                </div>
                            </div>
                        </FadeIn>

                        <FadeIn delay={300} direction="up">
                            <div className="flex flex-col group h-full bg-white p-6 md:p-8 rounded-2xl border border-slate-200 shadow-sm hover:shadow-md transition-shadow">
                                <div className="w-12 h-12 bg-emerald-50 text-emerald-600 rounded-xl flex items-center justify-center mb-6 border border-emerald-100">
                                    <Clock className="w-6 h-6" />
                                </div>
                                <h3 className="text-lg md:text-xl font-bold text-slate-900 mb-3">Direct Calendar <span className="text-[var(--primary)] text-bold">Booking</span></h3>

                                <p className="font-bold text-slate-900 text-[15px] mb-2 leading-snug">Wake up to qualified jobs ready on your schedule.</p>
                                <p className="text-slate-600 font-medium text-[14px] leading-relaxed mb-6">Once perfectly qualified, the AI presents available slots directly from your Google calendar and locks in the meeting seamlessly.</p>

                                <div className="mt-auto bg-slate-50 border border-slate-100 rounded-lg p-3 text-[13px] font-bold text-slate-700 flex items-center gap-2">
                                    <Clock className="w-4 h-4 text-emerald-500 fill-emerald-500" /> 100% automated. Zero back-and-forth.
                                </div>
                            </div>
                        </FadeIn>
                    </div>
                </div>
            </section>

            {/* --- INDUSTRY STAT PROOF SECTION (Lesson 5 Alternative for New Agencies) --- */}
            <section className="w-full py-16 md:py-20 bg-white">
                <div className="max-w-6xl mx-auto px-5">
                    <FadeIn>
                        <div className="bg-slate-900 rounded-3xl p-6 md:p-12 shadow-2xl overflow-hidden relative border border-slate-800 flex flex-col md:flex-row items-center gap-8 md:gap-10">
                            {/* Background gradient */}
                            <div className="absolute top-0 right-0 w-full md:w-1/2 h-full bg-gradient-to-l from-[var(--primary)]/20 to-transparent pointer-events-none -z-10 blur-xl" />

                            <div className="w-full md:w-1/3 flex flex-col items-start">
                                <div className="flex items-center gap-1 mb-3 md:mb-4">
                                    {[1, 2, 3, 4, 5].map(i => <Star key={i} className="w-4 md:w-5 h-4 md:h-5 text-amber-500 fill-current" />)}
                                </div>
                                <h2 className="text-[1.35rem] md:text-3xl font-bold text-white tracking-tight mb-2">The data doesn't lie.</h2>
                                <p className="text-slate-400 font-medium text-[14px] md:text-base">Why Fortune 500 companies rely on <strong className="text-white">sub-second AI response</strong> architecture.</p>
                            </div>

                            <div className="w-full md:w-2/3 border-t md:border-t-0 md:border-l border-slate-700 pt-8 md:pt-0 md:pl-10">
                                <p className="text-base md:text-xl text-slate-200 font-medium italic leading-relaxed mb-6">
                                    "Industry data shows that 78% of customers buy from the company that responds to their inquiry first. By deploying a conversational AI that responds in under 5 minutes, businesses see a <strong className="text-[var(--primary)] font-bold">391% increase</strong> in conversion rates from lead to booked appointment."
                                </p>
                                <div className="flex items-center gap-4">
                                    <div className="w-10 h-10 md:w-12 md:h-12 rounded-full border-2 border-slate-700 bg-slate-800 flex flex-col items-center justify-center shrink-0">
                                        <span className="text-white font-black text-xs md:text-sm tracking-tighter">HBR</span>
                                    </div>
                                    <div>
                                        <p className="text-white font-bold text-[15px] md:text-base mb-0.5">Lead Response Management Study</p>
                                        <p className="text-slate-400 text-[12px] md:text-sm font-medium">As featured in Harvard Business Review</p>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </FadeIn>
                </div>
            </section>

            {/* --- FAQ SECTION --- */}
            <section className="w-full py-12 md:py-16 px-5 max-w-4xl mx-auto mb-10 md:mb-16">
                <div className="mb-10 md:mb-12 text-center md:text-left">
                    <FadeIn>
                        <h2 className="text-[1.75rem] md:text-4xl font-extrabold text-[#0F172A] tracking-tight mb-2 md:mb-3">What local business owners ask before trying it.</h2>
                        <p className="text-slate-600 font-medium md:text-lg">We know you might be skeptical. Read these first.</p>
                    </FadeIn>
                </div>

                <div className="space-y-4">
                    {faqs.map((faq, idx) => {
                        const isPricing = idx === faqs.length - 1;
                        return (
                            <FadeIn key={idx} delay={50 * idx} direction="up">
                                <div className={`rounded-2xl border p-6 md:p-8 shadow-sm transition-all ${isPricing
                                    ? "bg-[var(--primary)]/5 border-[var(--primary)]/30"
                                    : "bg-white border-slate-200 hover:shadow-md"
                                    }`}>
                                    <h3 className={`font-bold text-[17px] md:text-[19px] mb-3 ${isPricing ? 'text-[var(--primary)]' : 'text-slate-900'}`}>
                                        {faq.question}
                                    </h3>
                                    <div className="text-slate-600 font-medium text-[15px] leading-relaxed">
                                        <p>{faq.answer}</p>
                                    </div>
                                </div>
                            </FadeIn>
                        );
                    })}
                </div>
            </section>

            {/* --- FOOTER (Powered By LaunchPath Branding) --- */}
            <footer className="w-full bg-[#0F172A] py-10 md:py-12 px-5 text-center mt-auto border-t border-slate-800">
                <div className="max-w-6xl mx-auto flex flex-col items-center justify-center gap-5">
                    <FadeIn delay={100}>
                        <div className="flex flex-col items-center justify-center gap-2">
                            <div className="flex items-center gap-2 text-slate-400 font-medium opacity-80 hover:opacity-100 transition-opacity cursor-pointer">
                                <span>Powered by</span>
                                <div className="flex items-center gap-2 text-white">
                                    <div className="bg-[var(--primary)] w-6 h-6 rounded flex items-center justify-center shadow-inner">
                                        <Zap className="w-3.5 h-3.5 fill-white text-white" />
                                    </div>
                                    <Logo className="text-lg md:text-xl" />
                                </div>
                            </div>
                        </div>
                    </FadeIn>
                    <FadeIn delay={200}>
                        <div className="flex flex-col items-center gap-3">
                            <div className="text-slate-500 text-[13px] md:text-sm font-medium">
                                &copy; {new Date().getFullYear()} LaunchPath Inc. All rights reserved. <span className="mx-1">|</span> Demo purposes only
                            </div>
                            <div className="flex items-center gap-4 text-xs md:text-[13px] text-slate-500 font-medium">
                                <Link href="/privacy-policy" className="hover:text-white transition-colors">Privacy Policy</Link>
                                <span className="text-slate-700">&bull;</span>
                                <Link href="/terms-of-service" className="hover:text-white transition-colors">Terms of Service</Link>
                            </div>
                        </div>
                    </FadeIn>
                </div>
            </footer>

        </div>
    );
}
```
