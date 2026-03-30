# Live Chat & SMS Widget Implementation Plan

## Context

Breezit needs an embeddable live chat widget that venue owners can place on their websites — similar to VenueAI's "Chat with us!" widget on granparaisogardens.com. The widget allows visitors to ask questions, get AI-powered answers using venue-specific data, and capture leads. When a visitor provides their phone number with SMS consent, the system hands off to the existing SMS agent pipeline (Agent 1 → 2 → 3 → 4).

**Competitor reference:** Gran Paraiso Gardens (VenueAI) — branded chat bubble, conversational AI, lead capture form within chat, SMS consent, "Powered by VenueAI" branding.

---

## 1. Chat Widget Architecture

### Recommendation: Standalone JavaScript Widget

**NOT** an Angular Web Component (the existing `embeddable-components` app bundles the full Angular runtime at 200KB+) and **NOT** an iframe (can't do floating overlays cleanly).

A framework-agnostic vanilla TypeScript bundle compiled to a single `<script>` tag. This is the industry standard (Intercom, Drift, VenueAI all work this way).

| Attribute | Value |
|-----------|-------|
| Framework | Preact (3KB) or lit-html (2KB) for minimal reactive rendering |
| Bundler | Vite in library mode → single IIFE bundle |
| Communication | Socket.io-client (slim build) for real-time |
| Styling | Shadow DOM (CSS isolation from host page) |
| Target size | ~30-40KB gzipped |

### Embed Code (What venues paste on their website)

```html
<script>
  window.BreezitChat = { serviceEventId: 'VENUE_SERVICE_EVENT_ID' };
</script>
<script src="https://cdn.breezit.com/chat/v1/breezit-chat.min.js" async></script>
```

Optional configuration:
```javascript
window.BreezitChat = {
  serviceEventId: '66fdbc0ed1616e9b7d70ed85',
  primaryColor: '#2D5016',
  greeting: 'Welcome to Gran Paraiso Gardens!',
  position: 'bottom-right',  // or 'bottom-left'
  locale: 'en'
};
```

### Widget UI Components

1. **Floating trigger button** — "Chat with us!" pill, bottom-right, venue-branded color
2. **Chat panel** — 400px wide, 600px tall on desktop; full-screen on mobile
3. **Message thread** — scrollable with typing indicators
4. **Lead capture form** — inline form within chat (name, email, phone)
5. **SMS consent disclaimer** — TCPA-compliant opt-in text
6. **Branding footer** — "Powered by Breezit" with optional whitelabel

### Project Location

```
chat-widget/
  src/
    index.ts           # Entry, initializes widget
    widget.ts          # Core widget class
    components/
      ChatBubble.ts    # Floating trigger button
      ChatPanel.ts     # Main chat panel
      MessageList.ts   # Message rendering
      LeadForm.ts      # Name/email/phone collection
      ConsentBanner.ts # TCPA/SMS consent
    services/
      api.ts           # REST API calls to backend
      socket.ts        # Socket.io client for real-time
      storage.ts       # localStorage for session persistence
    styles/
      widget.css       # Shadow DOM styles
    types.ts
  vite.config.ts
  package.json
  tsconfig.json
```

---

## 2. Backend: New Models

### LiveChatSession

File: `auth/src/modules/chat/liveChat/liveChatSessionModel.ts`

```typescript
{
  sessionId: String,          // UUID, stored in visitor's localStorage
  serviceEvent: ObjectId,     // ref: ServiceEvent
  vendor: ObjectId,           // ref: User
  visitorFingerprint: String, // Browser fingerprint for abuse detection
  visitorIp: String,          // For rate limiting
  leadInfo: {
    name: String,
    email: String,
    phone: String,
    smsConsent: Boolean,
    smsConsentTimestamp: Date
  },
  booking: ObjectId,          // ref: Booking (linked after lead capture)
  phoneMessageBase: ObjectId, // ref: PhoneMessageBase (linked after SMS handoff)
  status: String,             // 'active' | 'lead_captured' | 'sms_handed_off' | 'closed'
  messageCount: Number,       // For abuse tracking
  tokenUsage: Number,         // Cumulative ORQ tokens used
  lastMessageAt: Date,
  metadata: {
    referrer: String,         // Page URL where widget is embedded
    userAgent: String,
    source: String            // 'live_chat_widget'
  },
  createdAt: Date,
  updatedAt: Date
}
```

### LiveChatMessage

File: `auth/src/modules/chat/liveChat/liveChatMessageModel.ts`

```typescript
{
  session: ObjectId,          // ref: LiveChatSession
  role: String,               // 'visitor' | 'ai' | 'system'
  content: String,
  type: String,               // 'text' | 'lead_form' | 'availability_check' | 'pricing'
  metadata: Object,           // Tool call results, availability data, etc.
  tokenCount: Number,         // Token count for billing
  createdAt: Date
}
```

---

## 3. Backend: API Routes

Add to `auth/src/routes/routes.ts`: `router.use("/livechat", liveChatRouter);`

### New Endpoints

```
POST   /api/livechat/session       # Create/resume session
POST   /api/livechat/message       # Send visitor message, get AI response
POST   /api/livechat/lead          # Submit lead capture form
GET    /api/livechat/config/:id    # Get widget config (venue name, colors, etc.)
POST   /api/livechat/availability  # Check date availability (public)
```

### New Files

| File | Path |
|------|------|
| Router | `auth/src/modules/chat/liveChat/liveChatRoutes.ts` |
| Controller | `auth/src/modules/chat/liveChat/liveChatController.ts` |
| Service | `auth/src/modules/chat/liveChat/liveChatService.ts` |
| Rate limiter | `auth/src/modules/chat/liveChat/liveChatRateLimit.ts` |

---

## 4. Socket.io Changes

The existing Socket.io setup (`auth/src/index.ts` lines 27-42) restricts CORS to `*.breezit.com`. The live chat widget runs on arbitrary venue websites, so we need:

1. **Dedicated namespace** `/livechat` with relaxed CORS
2. **Authentication** via `sessionId` (UUID) instead of cookies
3. Existing `/api/socket.io` namespace stays unchanged

```typescript
// auth/src/index.ts addition
const liveChatIo = io.of("/livechat");
liveChatIo.use(async (socket, next) => {
  const sessionId = socket.handshake.query?.sessionId;
  const serviceEventId = socket.handshake.query?.serviceEventId;
  // Validate sessionId exists in LiveChatSession collection
  // Rate limit by IP
  next();
});
```

### CORS Configuration

Update `auth/src/app.ts` to add open CORS only for livechat endpoints:

```typescript
app.use('/api/livechat', cors({ origin: true, credentials: false }));
```

No cookies for auth — uses sessionId tokens instead, so `credentials: false` is appropriate.

---

## 5. AI Agent Orchestration for Live Chat

### Key Decision: Single Agent, Not Multi-Agent

The existing SMS pipeline makes 3-4 sequential ORQ.ai calls per message (Agent 1 → 2 → 3/4 in `phoneMessageService.ts`). Each call takes 2-5s. Total latency of 6-20s is acceptable for SMS but **terrible for live chat**.

**Recommendation:** Single ORQ.ai deployment `live_chat_reply` with tool calls. One LLM call per message (~2-3s).

**Trade-off:** Less specialized than multi-agent. If the live chat needs full extraction capabilities of Agent 2 later, add it as a background job (extract data asynchronously after the chat message is sent).

### Chat Agent Flow

```
Visitor Message
    |
    v
[Rate limit + abuse check]
    |
    v
[Build context: venue data + conversation history + availability]
    |
    v
ORQ.ai deployment: "live_chat_reply"
    (prompt: venue description, pricing, packages,
     + tool definitions for availability/scheduling)
    |
    v
[Handle tool calls: check_availability, capture_lead_info, schedule_tour]
    |
    v
AI Response → Socket.io → Visitor
```

### ORQ.ai Tool Definitions for Chat Agent

- `check_availability(date)` — reuses `isServiceAvailable` from calendarController
- `get_pricing(guests, package)` — reads ServiceEvent pricing data
- `schedule_tour(date, time)` — calls Cal.com via `bookEventFunction`
- `request_lead_info()` — triggers inline lead capture form in widget
- `get_alternative_dates()` — finds next available dates

### Prompt Design

Store in the AI Prompts system (`auth/src/modules/ai/prompts/`). Include:
- Venue identity and personality
- All venue data (packages, pricing, capacity) — reuse `generateDescription()` from `auth/src/utils/chatGPT.ts`
- Lead qualification strategy
- Tool definitions
- Instructions for when to trigger lead capture form
- SMS consent language

---

## 6. Lead Capture Flow

### Sequence

```
1. Visitor opens chat → AI greets with venue personality
2. Visitor asks questions → AI answers from venue data
3. After 2-3 exchanges (or pricing/availability question):
   → AI: "I'd love to help! Could I get your contact info?"
4. Widget shows inline form: [Name] [Email] [Phone]
   + SMS consent checkbox
5. Form submitted → POST /api/livechat/lead
6. Backend creates Booking (state: "inquiry", source: "live_chat_widget")
7. If phone + consent → initiate SMS handoff
8. AI continues conversation with full context
```

### CRM Integration

Trigger existing integrations on lead capture:
- **Booking creation** via `createBookingRequestObj`
- **HubSpot** via `createOrUpdateContact` (`auth/src/utils/hubspot.ts`)
- **Klaviyo** via `createEvent` (`auth/src/services/email/klaviyoEmailSend.ts`)
- **Slack notification** via `sendSlackMessage`
- **Vendor notification** via `notification-center.service.ts`

### Deduplication

Before creating a new Booking, check for existing bookings by phone number variants and email against active bookings for the same serviceEvent (same pattern as `phoneMessageService.ts` lines 180-228).

---

## 7. SMS Handoff

### Flow

```
[Live Chat: Phone collected + SMS consent given]
    |
    v
1. Create PhoneMessageBase (link to serviceEvent, phone, booking)
2. Send initial SMS via Twilio:
   "Hi [Name]! This is [Venue Name]. Thanks for chatting!
   I'll continue our conversation via text. [Context]"
3. Mark LiveChatSession status: 'sms_handed_off'
4. Future inbound SMS → existing smsWebhook pipeline
   (Agent 1 → Agent 2 → Agent 3/4)
```

### TCPA Compliance

1. **Explicit opt-in** — checkbox in lead form (NOT pre-checked)
2. **Record consent** — `smsConsent: true` + `smsConsentTimestamp` in LiveChatSession.leadInfo
3. **Consent language** — "By providing your phone number and checking this box, you agree to receive automated text messages from [Venue Name] regarding your inquiry. Message and data rates may apply. Reply STOP to opt out."
4. **Store consent proof** — LiveChatSession document serves as the consent record
5. **STOP handling** — existing Twilio built-in opt-out management

---

## 8. Agent Routing Diagram

```
                    ┌──────────────────────────────────────────┐
                    │    LIVE CHAT WIDGET (on venue site)       │
                    │    Visitor types message                  │
                    └──────────────────┬───────────────────────┘
                                       │
                                       v
                    ┌──────────────────────────────────────────┐
                    │  POST /api/livechat/message               │
                    │  1. Rate limit check                      │
                    │  2. Token budget check                    │
                    │  3. Credit check                          │
                    │  4. Build context (venue + history)        │
                    └──────────────────┬───────────────────────┘
                                       │
                                       v
                    ┌──────────────────────────────────────────┐
                    │  ORQ.ai: "live_chat_reply" deployment     │
                    │  Single agent with tool calls:            │
                    │  - check_availability(date)               │
                    │  - get_pricing(guests, package)           │
                    │  - schedule_tour(date, time)              │
                    │  - request_lead_info()                    │
                    │  - get_alternative_dates()                │
                    └──────────────────┬───────────────────────┘
                                       │
                         ┌─────────────┴──────────────┐
                         │                            │
                         v                            v
                ┌─────────────────┐          ┌──────────────────┐
                │ Text response   │          │ Tool call         │
                │ → Socket.io     │          │ → execute tool    │
                │ → store in DB   │          │ → re-invoke ORQ   │
                └─────────────────┘          └──────────────────┘
                         │
                         │  [Lead form submitted with phone + consent]
                         v
                ┌──────────────────────────────────────────────┐
                │  SMS HANDOFF                                  │
                │  1. Create PhoneMessageBase                   │
                │  2. Create Booking (state: inquiry)           │
                │  3. Send initial SMS via Twilio               │
                │  4. Mark session as sms_handed_off            │
                └──────────────────┬───────────────────────────┘
                                   │
                                   │  [Customer replies via SMS]
                                   v
                ┌──────────────────────────────────────────────┐
                │  EXISTING SMS PIPELINE                        │
                │  POST /api/chat/smsWebhook/:id               │
                │                                               │
                │  Agent 1 (sms_action): Route decision         │
                │    → create_booking / edit_booking / reply     │
                │                                               │
                │  Agent 2 (sms_extractor): Extract data        │
                │    → dates, guest count, budget, etc.         │
                │                                               │
                │  Agent 3/4 (create_sms / edit_or_reply_sms):  │
                │    → Generate + send response SMS             │
                └──────────────────────────────────────────────┘
```

---

## 9. Cost Measurement and Billing

### Existing Credit System

The current system (`saveAiAnalytics` at `aiController.ts:2733`) deducts from `PrivateInfo(type: "userInfo").data.credits`.

### Live Chat Billing Model: Per-Message Token-Based

```typescript
const LIVE_CHAT_COST_PER_1K_TOKENS = 0.003; // $0.003 per 1K tokens

// After each ORQ response:
const tokenCount = orqResponse.usage?.total_tokens || estimateTokens(response);
const messageCost = (tokenCount / 1000) * LIVE_CHAT_COST_PER_1K_TOKENS;

await saveAiAnalytics(
  vendorUserId,
  serviceEvent._id,
  'live_chat_message',  // New analytics type
  booking?._id,
  { sessionId, messageRole: 'ai', tokenCount },
  messageCost
);
```

### Cost Estimation

| Scenario | Messages | Est. Tokens | Est. ORQ Cost | SMS Cost |
|----------|----------|-------------|---------------|----------|
| Average chat session | 8 messages | ~4,000 | ~$0.012 | $0 |
| Lead captured + SMS | 10 + 1 SMS | ~5,000 | ~$0.015 + $0.0075 | ~$0.02 |
| Heavy session (max) | 30 messages | ~10,000 | ~$0.03 | $0 |
| Abuse session (capped) | 30 (capped) | 10,000 | ~$0.03 | $0 |

### Credit Check

Before each AI response, check credits (reuse `PhoneMessageService.checkAiCredits` pattern):

```typescript
if (!(await PhoneMessageService.checkAiCredits(aiConfig))) {
  // Return graceful "Please contact us directly" message
  // Don't burn tokens when vendor is out of credits
}
```

---

## 10. Abuse Prevention

### Multi-Layer Defense

| Layer | Mechanism | Limit |
|-------|-----------|-------|
| **1. IP Rate Limiting** | New middleware `liveChatRateLimit.ts` | 20 msgs / 5 min per IP; 5 sessions / hour per IP |
| **2. Token Budget** | Per-session `tokenUsage` field | 10,000 tokens / session (~20-30 exchanges) |
| **3. Message Length** | Reject > 1000 chars | HTTP 400 |
| **4. Cooldown** | Min 2s between messages from same session | Prevents rapid-fire automation |
| **5. Browser Fingerprint** | Canvas + WebGL + fonts hash | Flag same fingerprint creating many sessions |
| **6. Monitoring** | Slack alerts on threshold breach | Alert when session exceeds budget |

### Why No CAPTCHA (Initially)

CAPTCHAs reduce conversion rates significantly. The max cost of a single abusive session is ~$0.03 in tokens. With rate limiting at 5 sessions/hour/IP, an attacker costs at most ~$0.15/hour. Add CAPTCHA only if real abuse is detected.

---

## 11. Open Questions

1. **Agent 2 → Agent X hand-off:** For live chat, we use a single agent (`live_chat_reply`) instead of the multi-agent pipeline. The multi-agent pipeline activates only after SMS handoff, when the visitor replies via SMS and enters the existing `smsWebhook` flow.

2. **How to measure/calculate cost / charge customers for live chat?** Per-message token-based billing (see section 9). Deducted from existing vendor credit balance via `saveAiAnalytics`.

3. **How to limit exploitation risk?** Multi-layer defense (see section 10). Max cost per abusive session: $0.03.

4. **SMS communication flow:** Visitor provides name, phone, email, question → system creates PhoneMessageBase + sends initial SMS → inbound SMS replies enter existing pipeline: Agent 1 (route), Agent 2 (extract), Agent 3/4 (respond).

---

## 12. Widget Configuration

Store in `PrivateInfo` with `type: "aiConfig"`, extending `data`:

```typescript
liveChat: {
  enabled: boolean,
  primaryColor: string,
  greeting: string,
  logo: string,
  position: 'bottom-right' | 'bottom-left',
  tokenBudgetPerSession: number,  // Default 10000
  rateLimit: number               // Default 20 per 5 min
}
```

Config endpoint: `GET /api/livechat/config/:serviceEventId` returns venue name, colors, logo, feature flags.

---

## 13. Phased Implementation

### Phase 1: Backend Foundation (Week 1-2)
1. Create LiveChatSession and LiveChatMessage models
2. Create liveChatRoutes, liveChatController, liveChatService
3. Add Socket.io `/livechat` namespace with session-based auth
4. Update CORS for `/api/livechat` endpoints
5. Implement rate limiting middleware
6. Add widget config endpoint
7. Wire up `saveAiAnalytics` for live_chat type

### Phase 2: ORQ.ai Chat Agent (Week 2-3)
1. Create `live_chat_reply` deployment in ORQ.ai with venue prompt
2. Define tool calls: check_availability, get_pricing, schedule_tour, request_lead_info
3. Implement tool call handlers (reusing existing functions)
4. Add prompt to the AI Prompts system
5. Test end-to-end with curl/Postman

### Phase 3: Chat Widget (Week 3-4)
1. Set up `chat-widget/` project with Vite + Preact
2. Build UI components (bubble, panel, messages, lead form)
3. Implement Socket.io client for real-time messaging
4. Add Shadow DOM styling with venue theming
5. Implement localStorage session persistence
6. Build CDN deployment pipeline

### Phase 4: Lead Capture + SMS Handoff (Week 4-5)
1. Lead form → Booking creation
2. SMS consent UI and TCPA-compliant language
3. PhoneMessageBase creation + initial SMS
4. Full flow test: chat → lead → SMS → existing pipeline
5. HubSpot/Klaviyo integration

### Phase 5: Admin UI + Configuration (Week 5-6)
1. Live chat config in vendor dashboard
2. Widget customization UI (colors, greeting, logo)
3. Live chat conversation viewer in messaging inbox
4. Enable/disable toggle per venue
5. Token budget configuration

### Phase 6: Monitoring + Polish (Week 6-7)
1. Error monitoring (ErrorLoggingService pattern)
2. Slack alerts for abuse and errors
3. Analytics dashboard
4. Mobile responsive testing
5. Cross-browser testing
6. Load testing

---

## 14. Critical Files to Reuse

| File | What to Reuse |
|------|---------------|
| `auth/src/modules/chat/phoneMessage/phoneMessageService.ts` | AI pipeline pattern, context building, ORQ.ai invocation, tool call handling, SMS sending, credit checking |
| `auth/src/modules/ai/orq/orqController.ts` | `generateOrqAiReplyFn`, `OrqMetadata` interface |
| `auth/src/index.ts` | Socket.io setup (lines 27-42), add `/livechat` namespace |
| `auth/src/modules/chat/phoneMessage/phoneMessageController.ts` | `sendSms` function, `handleSmsWebhook`, error monitoring pattern |
| `auth/src/modules/ai/aiController.ts` | `saveAiAnalytics` for credit deduction |
| `auth/src/modules/marketplace/calendar/calendarController.ts` | `isServiceAvailable` for availability checks |
| `auth/src/services/calcomService.ts` | `getAvailableTimesFunction`, `bookEventFunction` |
| `auth/src/utils/chatGPT.ts` | `generateDescription` for venue prompt context |
| `auth/src/services/notification-center.service.ts` | Vendor notifications on lead capture |
| `auth/src/utils/hubspot.ts` | `createOrUpdateContact` for CRM |
