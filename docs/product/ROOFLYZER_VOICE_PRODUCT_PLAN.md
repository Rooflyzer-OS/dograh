# Rooflyzer Voice — Product & UX Architecture Plan

## How to offer a roofing-native voice AI agent product (Alivo competitor) built on the rebranded Dograh fork, integrated into the Rooflyzer-OS app.

---

## 1. Competitive Landscape: Alivo

Alivo is the strongest roofing-vertical AI agent platform as of May 2026. Here's what they offer and where Rooflyzer can differentiate.

### Alivo's Product

| Agent | Role | Channel |
|-------|------|---------|
| **Lilly** | Answers inbound calls 24/7, books appointments | AI Voice (phone) |
| **Evan** | Responds to web leads in <60s, follows up | SMS, Email, Web chat |
| **Alex** | Follows up on unsigned estimates, rehashes leads | SMS, Email, Voice |
| **Dylan** | Door-knock canvassing support, lead capture | Mobile app + SMS |
| **Jenna** | Post-job reviews, referral nurturing | SMS, Email |

**Pricing:** $999/mo (annual) or $1,299/mo (monthly) for Agent Team. +$499-649/mo for Team Additions. No self-serve — sales-demo only. White-glove onboarding. CRM integrations: JobNimbus, AccuLynx, ServiceTitan, Housecall Pro, Jobber.

**Alivo's weaknesses (our opportunities):**
1. **No self-serve** — you must book a demo. Rooflyzer can offer instant onboarding.
2. **No transparent pricing** — sales-quoted only. Rooflyzer can show pricing upfront.
3. **No visual workflow builder** — agents are black-box. Rooflyzer has Dograh's drag-and-drop canvas.
4. **No BYO telephony control** — Alivo owns the numbers. Rooflyzer can let contractors bring their own or buy through us.
5. **$1,299/mo locks out solo roofers** — Rooflyzer can offer a tiered model starting lower.
6. **No fine-tuned roofing model** — Alivo uses generic LLMs with prompt engineering. Rooflyzer has 41k call recordings for fine-tuning.

---

## 2. Product Architecture: The AI Agent Dashboard

### 2.1 Navigation — Mirror the Marketing Hub Pattern

The AI Agent dashboard should follow the exact same shell pattern as `MarketingHubShell`:

```
/app/ai-agents                    → Overview (pre-configured agents + context)
/app/ai-agents/agents             → Agent Library (create, edit, test)
/app/ai-agents/workflows          → Workflow Builder (journey/automation maker)
/app/ai-agents/phone-numbers      → Phone Numbers (buy, port, connect)
/app/ai-agents/recordings         → Call Recordings & Transcripts
/app/ai-agents/analytics          → Performance Analytics
/app/ai-agents/settings           → Voice, Billing, Compliance Settings
```

**Shell config (same as marketing):**
- Suppress OS sidebar, OS header, mobile bottom nav, FAB
- Sticky product header: "AI Agents" + subtitle + Back to Home
- Hub tabs: Overview · Agents · Workflows · Phone Numbers · Recordings · Analytics · Settings
- Sending-identity strip (phone number + voice) → phone-numbers / settings
- Full-bleed builders (workflow editor, agent tester) hide chrome

### 2.2 The Overview Page — Context-Aware, Zero-Config Start

This is the key differentiator from Alivo. When a contractor onboards, the overview page should already have context drawn in from their Rooflyzer CRM data:

**Auto-populated from CRM on first load:**
- **Service areas** (from their service territory settings / ZIP codes)
- **Services offered** (from their services catalog: roof replacement, roof repair, storm damage, gutter installation, solar, etc.)
- **Products offered** (from their pricing matrix: asphalt shingle, metal, tile, TPO, etc.)
- **Business hours** (from their calendar settings)
- **Pricing tiers** (from their quote engine: good/better/best or range)

**Based on this context, show 4 pre-configured agent cards:**

| Agent | Name (editable) | Role | Voice | Pre-loaded with |
|-------|-----------------|------|-------|-----------------|
| **Inbound Call Agent** | "Sarah" (default) | Answers every call 24/7, qualifies, books appointment | Female voice option | Service areas, services, pricing, business hours, calendar |
| **Lead Response Agent** | "Marcus" (default) | Responds to web/form leads in <60s via SMS + chat | Male voice option | Services, pricing, qualification questions, service areas |
| **Estimate Follow-up Agent** | "Elena" (default) | Follows up on unsigned estimates, rehashes cold leads | Female voice option | Quote data, pricing tiers, follow-up sequences |
| **Review & Referral Agent** | "James" (default) | Post-job review requests, referral nurturing | Male voice option | Job completion data, review links, referral templates |

Each card shows:
- Agent name (editable inline)
- Voice selector (3-4 voice options, male/female)
- Status toggle (Active/Paused)
- "Test Live" button → opens the test chat panel
- "Edit Workflow" button → opens the workflow builder
- Connected phone number (or "Connect a number" CTA)

### 2.3 Agent Configuration — Locked-Down Stack

**What the contractor CAN configure:**
- Agent name (e.g., "Sarah", "Front Desk AI")
- Voice selection (3-4 pre-vetted voices, male/female)
- Personality/tone slider (Professional ←→ Friendly)
- Greeting message (custom or use template)
- Business hours (when agent is active vs. voicemail)
- Qualification questions (from templates, editable)
- Connected phone number(s)
- Transfer-to-human number (for escalations)

**What the contractor CANNOT configure (locked on backend):**
- STT provider (we pick: Deepgram or Workers AI)
- TTS provider (we pick: Cartesia or ElevenLabs)
- LLM provider (we pick: our fine-tuned roofing model)
- Telephony provider (we pick: Telnyx)
- API keys for any of the above (we manage, we bill)
- Provider switching (no BYOK mode visible in UI)

**Why lock it down:**
- Prevents contractors from deviating from roofing use cases
- Ensures consistent quality (we vetted the stack)
- Simplifies billing (we charge for overall usage, not per-provider)
- Reduces support burden (no "I switched to a cheap TTS and it sounds robotic" tickets)

### 2.4 Phone Number Connection

Three paths, presented in priority order:

1. **Buy a number through us** (recommended)
   - Search by area code or city
   - Telnyx number search API
   - Monthly fee ($1-3/mo per number) + usage billing
   - Instant activation
   - Number is owned by Rooflyzer, portable if they leave

2. **Port an existing number**
   - Upload LOA (Letter of Authorization)
   - Telnyx porting API
   - 7-14 business days
   - Status tracker in the dashboard

3. **Connect an existing Telnyx number** (advanced)
   - Enter Telnyx number + auth
   - For contractors who already have Telnyx accounts
   - We configure webhook routing to our backend

**Phone number card UI:**
```
┌─────────────────────────────────────────────┐
│ 📞 Phone Numbers                            │
│                                             │
│ Active: +1 (555) 123-4567                   │
│ Connected to: Sarah (Inbound Call Agent)    │
│                                             │
│ [Buy New Number]  [Port Number]  [Manage]   │
└─────────────────────────────────────────────┘
```

### 2.5 Live Test UI — Same as Current Test Chat

The test chat should reuse the existing `WorkflowTesterPanel` pattern from Dograh, which already has:
- **Audio mode** (voice call simulation via WebRTC)
- **Text mode** (manual text chat with the agent)
- **Simulated mode** (AI-driven conversation simulation)

The test panel should be accessible from:
1. The agent card on the Overview page ("Test Live" button)
2. The agent editor page (side panel)
3. The workflow builder (right rail — see layout fix below)

**Test panel UI (consistent with current Rooflyzer test chat):**
```
┌──────────────────────────────────┐
│ Test Agent: Sarah                │
│ [Audio] [Text]  [Manual] [Sim]   │
│                                  │
│ ┌──────────────────────────────┐ │
│ │ Agent: Hi! Thanks for        │ │
│ │ calling Rooflyzer. How can   │ │
│ │ I help you today?            │ │
│ │                              │ │
│ │ You: I have a roof leak      │ │
│ │ Agent: I'm sorry to hear     │ │
│ │ that. What's your address?   │ │
│ └──────────────────────────────┘ │
│                                  │
│ [Type message...]        [Send]  │
│                                  │
│ Status: Connected · 23s          │
└──────────────────────────────────┘
```

---

## 3. Workflow Builder Layout Fix

### 3.1 Current Problem

In Dograh's `RenderWorkflow.tsx`, the layout is:
```
┌──────────────────────────────────────────────┐
│ Workflow Header (name, save, test, history)  │
├──────────────────────────────┬───────────────┤
│                              │               │
│   React Flow Canvas          │  Tester Panel │
│   (nodes, edges, zoom)       │  (400px right)│
│                              │               │
│   [+ Add Node] (top-right)   │               │
│   [Zoom] (bottom-left)       │               │
│                              │               │
├──────────────────────────────┴───────────────┤
│ AddNodePanel (overlay/drawer)                │
└──────────────────────────────────────────────┘
```

The user's complaint: "the drag-and-drop widgets for both the journey maker and the AI chat are on the left side, which is confusing."

### 3.2 Fixed Layout — Chat on LEFT, Layers on RIGHT

```
┌──────────────────────────────────────────────────────────┐
│ Workflow Header (name, save, publish, version history)   │
├──────────────┬───────────────────────────┬───────────────┤
│              │                           │               │
│  CHAT PANEL  │   React Flow Canvas       │  NODE/LAYER   │
│  (always     │   (workflow nodes,        │  PALETTE      │
│   visible)   │    edges, zoom)           │  (add nodes,  │
│              │                           │   configure)  │
│  320px       │   flex-1 (fills middle)   │  300px        │
│  left rail   │                           │  right rail   │
│              │                           │               │
│  [Audio]     │   [+ Add] (top-right of   │  📥 Inbound   │
│  [Text]      │    canvas, opens right    │  🗣️ STT       │
│              │    rail fully)            │  🧠 LLM       │
│  Messages    │                           │  🔊 TTS       │
│  scroll      │   [Zoom] (bottom-left)    │  📞 Telephony │
│              │                           │  🔧 Tools     │
│  [Input]     │                           │  📋 Logic     │
│  [Send]      │                           │               │
│              │                           │  (click to    │
│  Status:     │                           │   add to      │
│  Connected   │                           │   canvas)     │
│              │                           │               │
├──────────────┴───────────────────────────┴───────────────┤
│ Mobile: Chat = bottom sheet, Palette = right sheet       │
└──────────────────────────────────────────────────────────┘
```

### 3.3 Why This Layout

| Side | Content | Rationale |
|------|---------|-----------|
| **LEFT** | Chat panel (always visible) | The conversation is the source of truth. You're building a workflow to serve the conversation, not the other way around. Chat stays visible so you can test as you build. |
| **CENTER** | React Flow canvas | The workflow graph is the work surface. It gets the most space. |
| **RIGHT** | Node/layer palette | Adding nodes is a secondary action — you do it periodically, not constantly. Right side matches Figma, VS Code, and most IDE panel conventions. |

### 3.4 Implementation Changes in `RenderWorkflow.tsx`

Current structure (simplified):
```tsx
<div className="flex h-full min-w-0">
  <div className="relative min-w-0 flex-1">  {/* Canvas */}
    <ReactFlow ... />
  </div>
  {isTesterRailOpen && (
    <aside className="w-[400px] border-l">  {/* Tester - RIGHT */}
      <WorkflowTesterPanel />
    </aside>
  )}
</div>
```

Fixed structure:
```tsx
<div className="flex h-full min-w-0">
  {/* LEFT: Chat panel (always visible on desktop) */}
  <aside className="hidden w-[320px] shrink-0 border-r border-border lg:block">
    <WorkflowTesterPanel
      workflowId={workflowId}
      className="h-full"
      ...
    />
  </aside>

  {/* CENTER: Canvas */}
  <div className="relative min-w-0 flex-1">
    <ReactFlow ... />
  </div>

  {/* RIGHT: Node palette (always visible on desktop) */}
  <aside className="hidden w-[300px] shrink-0 border-l border-border lg:block">
    <AddNodePanel
      isOpen={true}
      onNodeSelect={handleNodeSelect}
      nodes={nodes}
      className="h-full"
      ...
    />
  </aside>
</div>

{/* Mobile: Chat = bottom sheet, Palette = right sheet */}
<Sheet open={isTesterSheetOpen} ...>
  <SheetContent side="bottom">
    <WorkflowTesterPanel />
  </SheetContent>
</Sheet>
<Sheet open={isAddNodeSheetOpen} ...>
  <SheetContent side="right">
    <AddNodePanel />
  </SheetContent>
</Sheet>
```

Key changes:
1. `WorkflowTesterPanel` moves from right `aside` to left `aside` (320px, `border-r`)
2. `AddNodePanel` changes from overlay/drawer to a permanent right `aside` (300px, `border-l`)
3. Both panels are `hidden lg:block` (hidden on mobile, shown on desktop)
4. Mobile uses `Sheet` components: chat = bottom sheet, palette = right sheet
5. The `+ Add Node` button in the canvas top-right can toggle the right palette on mobile

### 3.5 Consistency with Marketing Journey Maker

The Rooflyzer marketing journey maker (`/app/marketing/journeys/:id`) uses React Flow with a similar canvas. To maintain consistency:
- Both builders should use the same left-chat / center-canvas / right-palette layout
- Both should use the same panel widths (320px left, 300px right)
- Both should use the same mobile Sheet pattern
- The node palette content differs (marketing nodes vs. voice agent nodes) but the shell is identical

---

## 4. Backend Architecture — Managed Stack

### 4.1 Telephony (Telnyx)

All telephony is managed on the backend:
- Rooflyzer holds the Telnyx account
- Each contractor gets a Telnyx number provisioned under our account
- Webhooks route to our Cloudflare Worker (`api.rooflyzer.pro`)
- The Worker routes calls to the appropriate Dograh instance or Workers DO

**Flow:**
```
Caller → Telnyx number → webhook → api.rooflyzer.pro/v1/voice/inbound
  → lookup client_id by phone number
  → route to Dograh instance (advanced) or Workers DO (simple)
  → agent conversation
  → transcript + recording stored in Supabase
  → usage logged for billing
```

### 4.2 AI Stack (Managed, No BYOK)

| Layer | Provider | Why | Billing |
|-------|----------|-----|---------|
| STT | Deepgram | Lowest latency, roofing vocabulary | Per-minute |
| LLM | Fine-tuned roofing model (served on Modal) | 41k call recordings, roofing-native | Per-token |
| TTS | Cartesia | Natural voice, low latency, cloning | Per-character |
| Telephony | Telnyx | Reliable, good API, number porting | Per-minute |
| Embeddings | Workers AI / Vectorize | Knowledge base retrieval | Included |

**Contractor sees:** "AI Voice Minutes" on their bill. We handle all provider costs internally and margin the usage.

### 4.3 Fine-Tuning Pipeline (Separate from This Plan)

```
41k call recordings
  → transcribe (Deepgram batch)
  → clean + label (roofing intents, dispositions)
  → format as conversation pairs
  → fine-tune base model (Llama 3.3 70B or Qwen 2.5 72B on Modal)
  → evaluate (held-out set, roofing-specific evals)
  → serve (Modal endpoint)
  → Dograh LLM provider points to this endpoint
```

### 4.4 Billing Model

| Tier | Price | Includes |
|------|-------|----------|
| **Starter** | $299/mo | 1 agent, 1 phone number, 500 AI minutes/mo, text chat only |
| **Professional** | $599/mo | 3 agents, 2 phone numbers, 2,000 AI minutes/mo, voice + text |
| **Team** | $999/mo | 5 agents, 5 phone numbers, 5,000 AI minutes/mo, voice + text + workflows |
| **Overage** | $0.15/min | Beyond included minutes |

This undercuts Alivo ($999-1,299/mo) while offering self-serve onboarding and a visual workflow builder they don't have.

---

## 5. Onboarding Flow — Zero to First Call in 5 Minutes

```
Step 1: Contractor signs up / logs into Rooflyzer
  ↓
Step 2: "Set up your AI Voice Agent" CTA on dashboard
  ↓
Step 3: Auto-detect context from CRM
  - Service areas ✓ (from territory settings)
  - Services offered ✓ (from services catalog)
  - Pricing ✓ (from quote engine)
  - Business hours ✓ (from calendar)
  ↓
Step 4: Review & confirm context
  - "We found these service areas: [ZIP codes]. Correct?"
  - "You offer: Roof Replacement, Roof Repair, Storm Damage. Correct?"
  ↓
Step 5: Pick your agent team
  - 4 pre-configured agent cards shown
  - Pick voice (male/female) for each
  - Name each agent
  - Toggle which agents to activate
  ↓
Step 6: Get a phone number
  - "Buy a number" (search by area code) OR "Port existing number"
  - Number auto-connected to Inbound Call Agent
  ↓
Step 7: Test your agent
  - "Test Live" button → chat panel opens
  - Text or audio test
  - Agent responds using their CRM context
  ↓
Step 8: Go live
  - "Activate Agent" toggle
  - Phone number is now live
  - Calls are being answered
```

---

## 6. UI/UX Design System Consistency

### 6.1 Design Tokens (Already Applied)

The rebranded Dograh fork now uses Rooflyzer's design tokens:
- Primary: `oklch(0.55 0.23 264)` (#465fff electric blue)
- CTA: brand blue (not warm amber)
- Dark mode: brand blue stays consistent
- Sidebar: dark charcoal with blue accents
- Charts: chart-1 = brand blue

### 6.2 Component Consistency

| Component | Rooflyzer main app | Dograh fork | Action |
|-----------|-------------------|-------------|--------|
| Button | shadcn/ui Button | shadcn/ui Button | Already consistent |
| Card | ProDataCard / Card | Card | Wrap Dograh's Card with Rooflyzer's card-weave class |
| Sidebar | AppSidebar (dark) | AppSidebar (dark) | Already consistent |
| Tabs | shadcn/ui Tabs | shadcn/ui Tabs | Already consistent |
| Toast | sonner | sonner | Already consistent |
| Icons | lucide-react | lucide-react | Already consistent |
| Fonts | Geist Sans | Geist Sans | Already consistent |

### 6.3 Hub Shell Pattern

The AI Agent dashboard should use a `VoiceHubShell` component that mirrors `MarketingHubShell`:
- Same sticky header pattern
- Same hub tab navigation
- Same builder-path detection (hide chrome for workflow editor)
- Same loader pattern (but with a phone/headset icon instead of megaphone)
- Same sending-identity strip (but showing phone number + active agent instead of domain + SMS)

---

## 7. Implementation Roadmap

### Phase 1: Foundation (Weeks 1-2)
- [ ] Create `VoiceHubShell` component (mirror `MarketingHubShell`)
- [ ] Create `/app/ai-agents` route structure
- [ ] Build Overview page with auto-context detection
- [ ] Build 4 pre-configured agent cards
- [ ] Wire agent cards to Supabase `ai_agents` table

### Phase 2: Agent Configuration (Weeks 2-3)
- [ ] Build agent editor page (name, voice, personality, greeting)
- [ ] Lock down provider options in `AIModelConfigurationV2Editor`
- [ ] Hide BYOK mode, show only managed mode
- [ ] Build voice selector (3-4 pre-vetted voices)

### Phase 3: Phone Numbers (Weeks 3-4)
- [ ] Build phone number management page
- [ ] Telnyx number search + purchase flow
- [ ] Number-to-agent assignment
- [ ] Porting flow (LOA upload, status tracker)

### Phase 4: Workflow Builder Fix (Week 3)
- [ ] Move `WorkflowTesterPanel` to left rail (320px)
- [ ] Move `AddNodePanel` to right rail (300px)
- [ ] Add mobile Sheet patterns
- [ ] Apply same layout to marketing journey maker

### Phase 5: Live Test Integration (Week 4)
- [ ] Connect test panel to Dograh's workflow run API
- [ ] Audio mode: WebRTC voice test
- [ ] Text mode: manual chat
- [ ] Show real-time transcript + node transitions

### Phase 6: Billing & Usage (Week 5)
- [ ] Usage tracking (AI minutes, tokens, characters)
- [ ] Stripe billing integration (metered usage)
- [ ] Usage dashboard with overage alerts
- [ ] Tier limits enforcement

### Phase 7: Fine-Tuned Model (Weeks 5-8, parallel)
- [ ] Transcribe 41k recordings (Deepgram batch)
- [ ] Clean + label training data
- [ ] Fine-tune base model on Modal
- [ ] Serve endpoint
- [ ] Point Dograh LLM provider to endpoint

### Phase 8: Polish & Launch (Week 6)
- [ ] Onboarding flow (5-minute zero-to-first-call)
- [ ] Empty states for all pages
- [ ] Mobile responsive pass
- [ ] Sentry error tracking
- [ ] Analytics (PostHog events for agent creation, testing, activation)

---

## 8. Key Files to Create/Modify

### New Files (Rooflyzer-OS main repo)
- `apps/web-app/src/components/ai-agents/hub/VoiceHubShell.tsx` — mirror of MarketingHubShell
- `apps/web-app/src/lib/ai-agents/voiceHubNav.ts` — nav config (like marketingHubNav.ts)
- `apps/web-app/src/pages/ai-agents/VoiceAgentsOverviewPage.tsx` — overview with agent cards
- `apps/web-app/src/pages/ai-agents/AgentEditorPage.tsx` — agent configuration
- `apps/web-app/src/pages/ai-agents/PhoneNumbersPage.tsx` — phone number management
- `apps/web-app/src/pages/ai-agents/VoiceAnalyticsPage.tsx` — performance analytics
- `apps/web-app/src/components/ai-agents/AgentCard.tsx` — pre-configured agent card
- `apps/web-app/src/components/ai-agents/VoiceSelector.tsx` — voice picker (3-4 options)
- `apps/web-app/src/components/ai-agents/PhoneNumberConnect.tsx` — Telnyx number flow
- `apps/web-app/src/services/telnyxService.ts` — Telnyx API client

### Modified Files (Dograh fork)
- `ui/src/app/workflow/[workflowId]/RenderWorkflow.tsx` — layout fix (chat left, palette right)
- `ui/src/components/AIModelConfigurationV2Editor.tsx` — hide BYOK mode
- `ui/src/components/VoiceSelector.tsx` — lock to 3-4 pre-vetted voices
- `ui/src/components/flow/AddNodePanel.tsx` — convert to permanent right rail

### Database (Supabase migrations)
- `ai_agent_voice_profiles` — pre-vetted voice options (name, gender, provider, voice_id)
- `ai_agent_templates` — pre-configured agent templates (inbound, lead-response, follow-up, review)
- `phone_numbers` — contractor phone numbers (number, client_id, agent_id, telnyx_id, status)
- `voice_usage_log` — usage tracking for billing (client_id, agent_id, minutes, tokens, timestamp)

---

## 9. What Makes This Better Than Alivo

| Feature | Alivo | Rooflyzer Voice |
|---------|-------|-----------------|
| Onboarding | Sales demo required | Self-serve, 5 minutes |
| Pricing | Hidden, $999-1,299/mo | Transparent, $299-999/mo |
| Workflow builder | None (black box) | Visual drag-and-drop canvas |
| Phone numbers | Alivo owns them | Buy through us or port (portable) |
| CRM integration | JobNimbus, AccuLynx, etc. | Native (it IS the CRM) |
| Fine-tuned model | Generic LLM + prompts | 41k roofing call recordings |
| Test before live | Not available | Live test in dashboard |
| Customization | Limited (they manage it) | Contractor controls name, voice, tone, hours |
| Provider lock-in | Full lock-in | Open source (BSD-2), can self-host |
| Solo roofer friendly | No ($1,299/mo minimum) | Yes ($299/mo starter) |

The killer combo: **native CRM + visual workflow builder + fine-tuned roofing model + self-serve onboarding + transparent pricing.** Alivo charges $1,299/mo for a black box. Rooflyzer charges $299/mo for a glass box with a steering wheel.
