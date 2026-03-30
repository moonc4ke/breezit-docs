# MCP / CLI / API Implementation Plan for the Event Industry

## Context

Breezit wants to position itself as the MCP / CLI / API for the event industry. By exposing venue data, pricing, availability, and booking capabilities through the Model Context Protocol (MCP), AI assistants (Claude, ChatGPT, Gemini, etc.) can programmatically access venue information and help customers plan events. This creates a new distribution channel where Breezit becomes the data source that AI recommends.

The existing codebase already has all the underlying data and business logic — this plan layers an MCP interface on top.

---

## 1. Architecture: Multi-Tenant MCP Server

### Recommendation: Single Multi-Tenant MCP Server

A single server routing by venue slug/ID, NOT per-venue MCP servers.

**Rationale:**
- All venue data is already in a shared MongoDB database, queried by `serviceEventId`
- The existing marketplace module already provides public access patterns (`getServiceBySlug`, `checkServiceAvailability`)
- Per-venue servers would mean thousands of processes — impractical
- Routing by venue slug in a single server is trivial

### Module Layout

```
foxyBackend/
├── auth/           (port 3000 - existing)
├── admin/          (port 3001 - existing)
└── mcp/            (port 3002 - NEW)
    ├── src/
    │   ├── index.ts                    # MCP server entry point
    │   ├── server.ts                   # MCP server setup (stdio + HTTP/SSE transport)
    │   ├── tools/                      # MCP tool definitions
    │   │   ├── getVenueInfo.ts
    │   │   ├── getPackagesAndPricing.ts
    │   │   ├── checkDateAvailability.ts
    │   │   ├── checkAppointmentSlots.ts
    │   │   ├── scheduleAppointment.ts
    │   │   └── createInquiry.ts
    │   ├── services/                   # Data access (imports/reuses from auth)
    │   │   ├── venueDataService.ts
    │   │   ├── availabilityService.ts
    │   │   ├── calcomService.ts
    │   │   └── inquiryService.ts
    │   ├── middleware/
    │   │   ├── apiKeyAuth.ts
    │   │   └── rateLimiter.ts
    │   ├── transport/
    │   │   ├── stdio.ts                # stdio transport for CLI/local
    │   │   └── httpSse.ts              # HTTP+SSE transport for WebMCP
    │   └── config/
    │       └── index.ts
    ├── package.json
    └── tsconfig.json
```

The MCP server connects to the same MongoDB as the auth module and reuses the same Mongoose models (ServiceEvent, ServiceBase, Calendar, Booking, Inquiry, PrivateInfo).

---

## 2. MCP Tools Specification

### Tool 1: `get_venue_info`

**Purpose:** Return public venue information.

**Reuses:** `getServiceBySlug` logic from `auth/src/modules/marketplace/service/serviceController.ts`

```typescript
{
  name: "get_venue_info",
  description: "Get venue information including name, description, photos, location, capacity, and amenities for a Breezit venue",
  inputSchema: {
    type: "object",
    properties: {
      venue_slug: { type: "string", description: "Venue slug or ID" }
    },
    required: ["venue_slug"]
  }
}
```

**Returns:**
- `name`, `slug`, `brandName` (from ServiceBase)
- `description` (from ServiceBase.description)
- `address` (from ServiceBase.address)
- `location` coordinates
- `photos` (from event type photos + profileLinkPicture)
- `capacity` (maxBanquetPeopleCount, maxBuffetPeopleCount, maxAccomodationPeopleCount)
- `facilities` (from ServiceBase.facilities)
- `enabledTypes` (wedding, businessEvent, privateEvent)
- `breezit_url`: canonical URL for backlinks

**Data sanitization:** Strip internal MongoDB `_id` fields, vendor user IDs, Cal.com API keys, sort internals, admin flags.

---

### Tool 2: `get_packages_and_pricing`

**Purpose:** Return package/pricing information.

**Data source:** `ServiceEvent.wedding.venuePricing`, `.catering`, `.cateringBeverages`, `.additionalServices`, etc.

```typescript
{
  name: "get_packages_and_pricing",
  description: "Get available packages and pricing tiers for a Breezit venue",
  inputSchema: {
    type: "object",
    properties: {
      venue_slug: { type: "string", description: "Venue slug or ID" },
      event_type: {
        type: "string",
        enum: ["wedding", "businessEvent", "privateEvent"],
        description: "Type of event (defaults to wedding)"
      }
    },
    required: ["venue_slug"]
  }
}
```

**Returns:**
- `basePrice`, `basePrice50`, `basePrice100`
- `venuePricing` details
- `catering` and `cateringBeverages` details
- `additionalServices`
- `accomodationPricing`
- `overtime` rates
- `charges`
- `seasonalPricingEnabled`, `weeklyPricingEnabled` flags
- `customPricing` flag (if true, prices are indicative)

**Note:** The serviceEventModel stores pricing data as flexible `Object` types. The MCP tool needs to handle the flexible schema and present it clearly.

---

### Tool 3: `check_date_availability`

**Purpose:** Check if a venue is available on a specific date.

**Reuses:** `isServiceAvailable` and `checkServiceAvailability` from `auth/src/modules/marketplace/calendar/calendarController.ts`

```typescript
{
  name: "check_date_availability",
  description: "Check if a Breezit venue is available for an event on a specific date or date range",
  inputSchema: {
    type: "object",
    properties: {
      venue_slug: { type: "string", description: "Venue slug or ID" },
      date: { type: "string", description: "Event date in YYYY-MM-DD format" },
      date_to: { type: "string", description: "Optional end date for multi-day events (YYYY-MM-DD)" }
    },
    required: ["venue_slug", "date"]
  }
}
```

**Returns:** `{ available: boolean, reason: string, timezone: string, date: string }`

**Implementation notes:**
1. Resolve `venue_slug` to `serviceEventId` (via `ServiceEvent.findOne({ slug })`)
2. Call existing `isServiceAvailable` function directly
3. The 7AM rule, cross-day slot logic, keyword filtering, and all other availability logic comes for free

---

### Tool 4: `check_appointment_slots`

**Purpose:** Get available appointment/tour time slots via Cal.com.

**Reuses:** `getAvailableTimesFunction` from `auth/src/services/calcomService.ts`

```typescript
{
  name: "check_appointment_slots",
  description: "Get available time slots for booking a tour or meeting at a Breezit venue",
  inputSchema: {
    type: "object",
    properties: {
      venue_slug: { type: "string", description: "Venue slug or ID" },
      date_from: { type: "string", description: "Start date (YYYY-MM-DD)" },
      date_to: { type: "string", description: "End date (YYYY-MM-DD, defaults to 2 weeks from date_from)" }
    },
    required: ["venue_slug", "date_from"]
  }
}
```

**Returns:** Array of available time slots with ISO 8601 timestamps in the venue's timezone.

**Notes:** Requires looking up the venue's Cal.com credentials from `PrivateInfo` where `type: "calUserInfo"` and `data.serviceEvent` matches. Not all venues will have Cal.com configured — the tool should return a clear message if unavailable.

---

### Tool 5: `schedule_appointment`

**Purpose:** Book a venue tour or meeting. **Write operation.**

**Reuses:** `bookEventFunction` from `auth/src/services/calcomService.ts`

```typescript
{
  name: "schedule_appointment",
  description: "Schedule a tour or meeting at a Breezit venue via Cal.com",
  inputSchema: {
    type: "object",
    properties: {
      venue_slug: { type: "string", description: "Venue slug or ID" },
      start_time: { type: "string", description: "Appointment start time in ISO 8601 format" },
      attendee_name: { type: "string", description: "Full name of the person booking" },
      attendee_email: { type: "string", description: "Email address" },
      attendee_phone: { type: "string", description: "Phone number (optional)" },
      notes: { type: "string", description: "Additional notes (optional)" }
    },
    required: ["venue_slug", "start_time", "attendee_name", "attendee_email"]
  }
}
```

**Implementation notes:**
1. Resolve slug to serviceEventId
2. Look up Cal.com credentials from PrivateInfo (type: "calUserInfo")
3. Look up timezone from aiConfig (PrivateInfo type: "aiConfig", at `data.timezone`)
4. Call `bookEventFunction`
5. Requires stronger authentication/rate limiting than read tools

---

### Tool 6: `create_inquiry`

**Purpose:** Submit a new event inquiry.

**Reuses:** `creteNewInquiry` from `auth/src/modules/marketplace/inquiries/inquiryController.ts`

```typescript
{
  name: "create_inquiry",
  description: "Submit a new event inquiry to Breezit for venue matching",
  inputSchema: {
    type: "object",
    properties: {
      name: { type: "string" },
      surname: { type: "string" },
      email: { type: "string" },
      phone: { type: "string" },
      category: { type: "string", description: "e.g., 'venues', 'catering'" },
      event_date: { type: "string", description: "YYYY-MM-DD" },
      guest_count: { type: "number" },
      budget: { type: "number" },
      location: { type: "string" },
      message: { type: "string" }
    },
    required: ["name", "email", "category"]
  }
}
```

**Notes:** MCP-originated inquiries won't have an authenticated user, so they should be created with `verified: false` and a verification token (matching the existing unauthenticated flow).

---

## 3. Authentication and Authorization Model

### Three-Tier Auth Strategy

| Tier | Tools | Auth Required | Rate Limit |
|------|-------|---------------|------------|
| **Tier 1: Public** | get_venue_info, get_packages_and_pricing, check_date_availability | None (same data is public on breezit.com) | 60 req/min per IP |
| **Tier 2: API Key** | check_appointment_slots | Breezit-issued API key | 120 req/min per key |
| **Tier 3: API Key + Strict Limits** | schedule_appointment, create_inquiry | API key + audit trail | 10 req/min per key |

### API Key Model

```typescript
// New model or PrivateInfo with type: "mcpApiKey"
{
  user: ObjectId,           // Vendor who owns this key
  serviceEventId: ObjectId, // Which venue this key grants access to
  key: string,              // Hashed API key (SHA-256)
  keyPrefix: string,        // First 8 chars for lookup (e.g., "brz_live_")
  permissions: string[],    // ["read", "appointments", "inquiries"]
  rateLimitTier: string,    // "free", "pro", "enterprise"
  enabled: boolean,
  createdAt: Date,
  lastUsedAt: Date,
}
```

Keys passed via `Authorization: Bearer brz_live_xxx` header on HTTP/SSE transport, or via MCP initialization message for stdio.

---

## 4. WebMCP Integration

### What is WebMCP?

WebMCP is the browser-based MCP transport standard that allows web-based AI assistants (ChatGPT in browser, Claude on claude.ai) to connect to MCP servers using HTTP + Server-Sent Events (SSE) instead of stdio.

### Dual Transport Support

The MCP server supports both:

1. **stdio transport** — for CLI tools, local development, desktop AI apps (Claude Desktop, Cursor)
2. **Streamable HTTP transport** — for WebMCP, browser-based AI assistants

The `@modelcontextprotocol/sdk` supports both transports natively:
- `POST /mcp` — JSON-RPC over HTTP (request-response)
- `GET /mcp` — SSE stream for server-initiated messages

### WebMCP Flow

```
Browser AI Assistant
        |
        v
   WebMCP Client
        | HTTP+SSE
        v
   Breezit MCP Server (port 3002)
        | reads/writes
        v
   MongoDB (shared with auth/admin)
```

### Discovery

Per-venue virtual endpoint: `https://api.breezit.com/mcp/{venue-slug}`

The MCP server routes internally based on the venue slug in the URL path. This is better for discoverability than a single global endpoint.

---

## 5. Framework Selection

### Recommendation: `@modelcontextprotocol/sdk` (official TypeScript SDK)

**Why:**
1. **Maintained by Anthropic** — the protocol creators, tracks spec changes
2. **TypeScript-native** — matches Breezit stack
3. **Dual transport** — stdio and HTTP+SSE out of the box
4. **Tool definition API** — clean `server.tool()` with Zod schema validation (Zod already in auth's package.json: `"zod": "^3.24.1"`)
5. **Mature** — v1.11+ with session management, auth hooks, progress reporting

**Alternatives considered:**
- **Custom Express endpoints** — simpler but misses MCP protocol benefits (tool discovery, prompt generation, resource serving). Doesn't work with MCP clients natively.
- **`fastmcp`** — extra dependency layer; official SDK is sufficient.

### Deployment

PM2 process alongside existing auth/admin. The existing PM2 dashboard can monitor it.

---

## 6. SEO / Backlink Strategy

### How Backlinks Work with MCP

MCP tool responses are JSON consumed by AI assistants. When an AI shows information from your MCP tool, it often cites the source.

**Include canonical URLs in every tool response:**

```json
{
  "name": "The Madison Hotel",
  "description": "...",
  "breezit_url": "https://breezit.com/venue/the-madison-hotel",
  "booking_url": "https://breezit.com/venue/the-madison-hotel/book",
  "source": "Breezit - AI-Powered Event Venue Platform",
  "source_url": "https://breezit.com",
  "_links": {
    "self": "https://breezit.com/venue/the-madison-hotel",
    "venue": "https://breezit.com/venue/the-madison-hotel",
    "breezit_home": "https://breezit.com"
  }
}
```

**SEO benefits:**
- AI assistants that respect attribution (Perplexity, Google Gemini) generate actual do-follow citations
- Brand visibility and traffic from AI-assisted search
- MCP tool descriptions appear in AI tool directories/marketplaces
- Photos with Breezit CDN URLs drive additional referral traffic

---

## 7. Open Questions

1. **Is it too early for MCP?** See risk assessment below.
2. **How do backlinks (do-follow) work with MCP?** Indirect — through AI citation of canonical URLs. Primary value is brand visibility and traffic, not traditional backlinks.
3. **How does WebMCP work in practice?** HTTP+SSE transport. Browser AI assistants connect to `https://api.breezit.com/mcp/{venue-slug}` and call tools via JSON-RPC.
4. **Which framework?** `@modelcontextprotocol/sdk` — official, TypeScript, supports both transports.
5. **Per-venue vs multi-tenant?** Multi-tenant. Single server, routing by venue slug.
6. **How to handle venues without Cal.com?** Tools return clear "not available" messages. Only venues with Cal.com configured expose appointment tools.

---

## 8. Risk Assessment: Is It Too Early?

### Assessment: The timing is good, not too early.

**Signals MCP adoption is real:**
- Claude Desktop, Cursor, Windsurf all support MCP natively
- Official SDKs are stable at v1.11+
- Anthropic, Google, and OpenAI have all signaled MCP support
- The "tool use" paradigm is how AI assistants will interact with external services
- Breezit already uses `.mcp.json` for MongoDB dev tooling

**Risks:**

| Risk | Level | Mitigation |
|------|-------|------------|
| Protocol fragmentation | LOW | MCP is becoming the de facto standard |
| Consumer adoption | MEDIUM | Being early means indexed in AI tool registries before competitors |
| Maintenance cost | LOW | ~1000-2000 lines for all 6 tools; same code trivially becomes REST API |
| Security exposure | MEDIUM | Careful rate limiting + input validation on write tools |

**Strategic assessment:** The event/wedding industry is highly search-driven. If AI assistants can query Breezit programmatically via MCP, Breezit becomes the data source that AI recommends. First-mover advantage is significant.

**Downside protection:** Phase 1 (read-only tools) exposes data that is already public. If MCP stalls, you've built a clean API layer that serves REST/CLI use cases.

---

## 9. Phased Rollout

### Phase 1: Read-Only MCP Server (2-3 weeks)
1. Create `foxyBackend/mcp/` module with package.json, tsconfig
2. Install `@modelcontextprotocol/sdk` and `zod`
3. Set up MongoDB connection (reuse MONGO_URI from auth's .env)
4. Import/copy necessary Mongoose models
5. Implement `get_venue_info` (reuse `getServiceBySlug` logic)
6. Implement `get_packages_and_pricing` (query ServiceEvent + format pricing)
7. Implement `check_date_availability` (reuse `isServiceAvailable` logic)
8. Add stdio transport for local testing
9. Add HTTP+SSE transport with rate limiting
10. Test with Claude Desktop via `.mcp.json`
11. Write tests (Vitest)

### Phase 2: Cal.com Integration + Auth (2-3 weeks)
1. Create API key model and management endpoints
2. Implement `check_appointment_slots`
3. Implement `schedule_appointment`
4. Add API key middleware for protected tools
5. Add venue opt-in flag
6. Audit logging for write operations

### Phase 3: Inquiry System + WebMCP (1-2 weeks)
1. Implement `create_inquiry`
2. Email verification flow for MCP-originated inquiries
3. WebMCP discovery endpoint (`.well-known/mcp.json`)
4. CORS configuration for WebMCP
5. Test with browser-based AI assistants

### Phase 4: Vendor Dashboard + Analytics (2-3 weeks)
1. MCP usage analytics (queries per venue, tools used, conversion tracking)
2. Vendor dashboard page showing MCP stats
3. API key management UI
4. MCP tool customization per venue

### Phase 5: REST API + CLI (1-2 weeks)
1. Express routes mirroring MCP tools (`GET /api/v1/venues/:slug`, etc.)
2. OpenAPI/Swagger documentation
3. CLI tool using stdio transport
4. API documentation site

---

## 10. Model Sharing Strategy

The MCP module needs the same Mongoose models as auth:

| Approach | Recommended For | Notes |
|----------|----------------|-------|
| **Direct import** (relative paths) | Phase 1 | Quick, avoids duplication |
| **Extract to `@breezit/models` package** | Phase 2+ | Aligns with `@foxywedding/common` pattern |
| **HTTP proxy to auth** | NOT recommended | Unnecessary network hops and latency |

### Slug Resolution Pattern

Every tool resolves `venue_slug` to `serviceEventId`:

```typescript
async function resolveVenue(venueSlug: string): Promise<ServiceEventDoc> {
  const objectId = mongoose.Types.ObjectId.isValid(venueSlug)
    ? new mongoose.Types.ObjectId(venueSlug) : null;

  const serviceEvent = await ServiceEvent.findOne({
    $or: [
      ...(objectId ? [{ _id: objectId }] : []),
      { slug: venueSlug },
      { slugAlias: venueSlug },
    ],
    disabled: { $ne: true },
    'serviceBaseParams.enabled': true,
    'serviceBaseParams.adminEnabled': { $ne: false },
  }).populate('serviceBase');

  if (!serviceEvent) throw new McpError('Venue not found');
  return serviceEvent;
}
```

---

## 11. Critical Files to Reuse

| File | What to Reuse |
|------|---------------|
| `auth/src/modules/marketplace/calendar/calendarController.ts` | `isServiceAvailable`, `isServiceAvailableForDateRange`, `getAiConfigWithFallback` |
| `auth/src/services/calcomService.ts` | `getAvailableTimesFunction`, `bookEventFunction` |
| `auth/src/modules/vendor/service/serviceEvent/serviceEventModel.ts` | ServiceEvent model with pricing |
| `auth/src/modules/marketplace/service/serviceController.ts` | `getServiceBySlug` aggregation pipeline |
| `auth/src/modules/common/privateInfo/privateInfoModel.ts` | PrivateInfo for aiConfig, Cal.com creds |
| `auth/src/modules/marketplace/inquiries/inquiryModel.ts` | Inquiry model |
| `auth/src/modules/marketplace/inquiries/inquiryController.ts` | `creteNewInquiry` logic |
