# Advanced Inquiry Form & Embeddable Widget System Implementation Plan

## Context

Breezit needs two interconnected features: (1) an advanced inquiry form with real-time availability checking, budget validation, and tour scheduling, and (2) an embeddable widget system (modeled after Cal.com's embed approach) that lets venues place the form on any website with minimal setup.

**Reference:** Cal.com's embed system offers inline embed, floating pop-up button, pop-up via element click, and email embed — with configuration for window sizing, theme, brand colors, and layout. We adopt this pattern but customize for the event inquiry use case.

---

## 1. Embed Technology Decision: Iframe + JS Loader

### Recommendation: Iframe-Based Architecture (Cal.com Pattern)

**Why NOT Angular Web Components:**
- The existing `embeddable-components` app (`FoxyVendor/apps/embeddable-components/`) is a full Angular application with zone.js, Angular Material, NgRx, etc. Compiling to Web Components would produce a 200KB+ widget.
- Style isolation between host page CSS and Angular Material/Tailwind would be fragile.

**Why NOT plain standalone JS widget:**
- The inquiry form has complex multi-step UX, date pickers, calendar rendering, validation — these benefit from Angular's form system and existing component libraries.
- Rebuilding all this in vanilla JS would duplicate effort.

**Why Iframe + JS Loader (Cal.com pattern):**
- The existing `embeddable-components` app is already served independently with its own build config.
- Iframe provides perfect CSS/JS isolation. No conflicts with host page.
- The backend already sets `Content-Security-Policy: frame-ancestors` in `app.ts` — extend to allow framing from any origin.
- Cross-origin communication via `postMessage` is secure and well-established.
- The existing CORS config already allows any origin.

### Architecture

```
Host Website                          Breezit CDN / Server
┌──────────────────┐                  ┌────────────────────────────┐
│                   │                  │                            │
│ <script src=      │   loads          │ breezit-widget.js (5KB)    │
│  "breezit.js">   │ <-----------     │ - Creates iframe           │
│                   │                  │ - postMessage bridge       │
│ <div id="breezit  │                  │ - Theme/config injection   │
│  -inquiry">       │                  └────────────────────────────┘
│                   │                         │
│ [iframe]          │  iframe src             v
│  ┌──────────────┐ │ -------->        ┌────────────────────────────┐
│  │ Angular App  │ │                  │ embed.breezit.com          │
│  │ (inquiry     │ │                  │ /inquiry/:venueId          │
│  │  form)       │ │                  │ Angular embeddable-        │
│  └──────────────┘ │                  │ components app             │
│                   │                  └────────────────────────────┘
└──────────────────┘                         │
                                             │ API calls
                                             v
                                    ┌────────────────────────────┐
                                    │ api.breezit.com            │
                                    │ /api/marketplace/widget/*  │
                                    │ (Express backend)          │
                                    └────────────────────────────┘
```

---

## 2. Form Architecture: Multi-Step Conversational Flow

### Step-by-Step Sequence

```
Step 1: Event Type Selection
  ├── Wedding / Business Event / Private Event
  └── Maps to serviceEvent.enabledTypes

Step 2: Event Date Selection
  ├── Calendar date picker
  ├── On date selected → API: POST /api/marketplace/widget/check-availability
  ├── IF NOT available:
  │   ├── Show "This date is not available"
  │   ├── Prompt: "Would you like to choose an alternative date?"
  │   └── Return to date picker
  └── IF available → proceed

Step 3: Guest Count
  ├── Numeric input
  └── Validated against serviceEvent.maxBanquetPeopleCount / maxBuffetPeopleCount

Step 4: Budget Indication
  ├── Numeric input or range selector
  ├── On entry → API: POST /api/marketplace/widget/check-budget
  ├── Backend validates against venue minimums (weekday vs weekend, seasonal modifiers)
  ├── IF below minimum:
  │   ├── Show "The minimum budget for [date] is $X"
  │   └── Allow user to adjust or proceed anyway
  └── IF within range → proceed

Step 5: Contact Information
  ├── Name, Surname, Email, Phone
  └── Reuses pattern from existing contact-form.component.html

Step 6: Additional Message (Optional)
  └── Free-text textarea for any questions

Step 7: Smart Completion
  ├── IF date available AND budget within limits:
  │   ├── Show: "Great news! [Date] is available at [Venue Name]"
  │   ├── Show: "Would you like to schedule a venue tour?"
  │   ├── IF yes → fetch Cal.com slots via /api/marketplace/widget/tour-slots
  │   │   ├── Display available tour dates/times
  │   │   └── On selection → book via Cal.com API
  │   └── IF no → submit inquiry
  └── Submit inquiry to backend

Step 8: Confirmation
  ├── "Your inquiry has been submitted!"
  └── Show Breezit backlink (do-follow)
```

### New Angular Components

All in `libs/embeddable/src/lib/inquiry-widget/`:

| Component | Purpose |
|-----------|---------|
| `InquiryWidgetComponent` | Root container, orchestrates steps |
| `EventTypeStepComponent` | Event type selector |
| `DateStepComponent` | Date picker with real-time availability check |
| `GuestCountStepComponent` | Guest count input |
| `BudgetStepComponent` | Budget input with validation feedback |
| `ContactInfoStepComponent` | Name, email, phone |
| `TourSchedulingStepComponent` | Cal.com slot selection |
| `ConfirmationStepComponent` | Success screen + backlink |
| `InquiryWidgetService` | API calls, state management |
| `WidgetConfigService` | Theme/config from query params or postMessage |

### Existing Components to Reuse

- `JarvisBookingFlowStepperComponent` (`libs/booking/src/lib/flows/components/flow-stepper.component.ts`) — step navigation pattern
- Existing inquiry flow components in marketplace — form field patterns
- Get-a-quote and book-a-tour flows — UX patterns

---

## 3. Backend: Widget API Endpoints

### New Module

| File | Path |
|------|------|
| Router | `auth/src/modules/marketplace/widget/widgetRoutes.ts` |
| Controller | `auth/src/modules/marketplace/widget/widgetController.ts` |
| Rate Limiter | `auth/src/modules/marketplace/widget/widgetRateLimiter.ts` |

Register in `auth/src/modules/marketplace/martketplaceRoutes.ts`: `router.use("/widget", widgetRoutes);`

### Endpoints

#### `POST /api/marketplace/widget/check-availability`

```
Body: { serviceEventId, date }
Returns: { available: boolean, reason?: string }
Reuses: isServiceAvailable() from calendarController.ts
```

#### `POST /api/marketplace/widget/check-budget`

```
Body: { serviceEventId, date, budget, guestCount, eventType }
Returns: {
  withinBudget: boolean,
  minimumBudget: number,
  estimatedPrice?: number,
  isWeekend: boolean,
  seasonalModifier?: string
}
Reuses: extractServicesFromAdvert() from pricing-calculator.ts
```

**Budget validation logic:**
1. Load ServiceEvent with pricing for selected eventType
2. Use `extractServicesFromAdvert()` from `auth/src/services/pricingCalculator/pricing-calculator.ts`
3. Apply seasonal pricing modifiers based on date's month
4. Apply weekly pricing modifiers (weekday vs weekend)
5. Calculate minimum budget (considering `globalMinimalBudget` from charges)
6. Compare visitor's budget against calculated minimum

#### `GET /api/marketplace/widget/venue-info/:serviceEventId`

```
Returns: {
  name, slug, category, address,
  enabledTypes, maxGuestCount,
  photos (first 3),
  hasCalcomIntegration: boolean
}
```

#### `POST /api/marketplace/widget/tour-slots`

```
Body: { serviceEventId, dateFrom, dateTo }
Returns: { slots: Array<{ date, time }> }
Reuses: getAvailableTimesFunction() from calcomService.ts
```

#### `POST /api/marketplace/widget/submit-inquiry`

```
Body: {
  serviceEventId, eventType, eventDate, budget, guestCount,
  contact: { name, surname, email, phone },
  message,
  tourSlot?: { date, time },
  widgetSource: string (hostname of embedding site)
}
Returns: { inquiryId, bookingId?, tourBookingId? }
```

**Submit flow:**
1. Validate all inputs
2. Re-check availability server-side (prevent stale data)
3. Create Inquiry document (extended with `source: "widget"`, `widgetOrigin`)
4. Optionally create Booking (reusing `saveBooking` logic)
5. If tour slot selected → call `bookEventFunction()` from calcomService.ts
6. Send notification to venue owner (reuse `notification-center.service.ts`)
7. Send confirmation email to visitor

#### `GET /api/marketplace/widget/config/:serviceEventId`

```
Returns: {
  theme, brandColors, customFields, enabledSteps,
  venueName, logo
}
```

### Inquiry Model Extension

Extend `auth/src/modules/marketplace/inquiries/inquiryModel.ts`:
- Add `source` field: `"marketplace"` | `"widget"` | `"mcp"` | `"live_chat"`
- Add `widgetOrigin` field: hostname where widget was embedded
- Add `serviceEvent` reference (if not already present)

---

## 4. Embed Deployment Options

### 4.1 JavaScript Widget Loader (`breezit-widget.js`)

A standalone vanilla TypeScript file (~5KB minified) compiled with esbuild. Provides the `Breezit` global namespace. No Angular dependency.

**Project location:**
| File | Location |
|------|----------|
| Source | `FoxyVendor/tools/widget-loader/breezit-widget.ts` |
| Build | `FoxyVendor/tools/widget-loader/build.js` (esbuild) |
| Output | `FoxyVendor/dist/widget/breezit-widget.js` → CDN |

### 4.2 Mode 1: Inline Embed

Creates an iframe inside the target element. Auto-resizes height via postMessage.

**Venue code:**
```html
<!-- Breezit Inquiry Widget -->
<div id="breezit-inquiry"></div>
<script src="https://embed.breezit.com/breezit-widget.js"></script>
<script>
  Breezit.inline({
    target: '#breezit-inquiry',
    venueId: '{SERVICE_EVENT_ID}',
    theme: 'light',
    brandColor: '#EF5518'
  });
</script>
```

**Iframe src:** `https://embed.breezit.com/inquiry/{venueId}?theme=light&color=%23EF5518&mode=inline`

### 4.3 Mode 2: Floating Pop-up Button

Injects a floating button (plain DOM, no framework). On click: creates modal overlay with iframe.

**Venue code:**
```html
<script src="https://embed.breezit.com/breezit-widget.js"></script>
<script>
  Breezit.floatingButton({
    venueId: '{SERVICE_EVENT_ID}',
    buttonText: 'Check Availability',
    buttonColor: '#EF5518',
    position: 'bottom-right'
  });
</script>
```

### 4.4 Mode 3: Pop-up via Element Click

Attaches click listener to a specified DOM element. Opens same modal as floating button.

**Venue code:**
```html
<script src="https://embed.breezit.com/breezit-widget.js"></script>
<script>
  Breezit.popup({
    venueId: '{SERVICE_EVENT_ID}',
    trigger: '#my-cta-button'
  });
</script>
```

### 4.5 Mode 4: Email Embed

Static HTML — no JavaScript (email clients strip JS/iframes). Renders a styled CTA button linking to hosted form.

**HTML snippet:**
```html
<table width="100%" cellpadding="0" cellspacing="0" style="max-width:600px;margin:0 auto">
  <tr>
    <td style="padding:20px;text-align:center;background:#f9f9f9;border-radius:8px">
      <h3 style="margin:0 0 8px;font-family:sans-serif">Check Availability at {VENUE_NAME}</h3>
      <p style="margin:0 0 16px;color:#666;font-family:sans-serif">
        See available dates and request a tour
      </p>
      <a href="https://breezit.com/{VENUE_SLUG}?source=email_embed"
         style="background:{BRAND_COLOR};color:#fff;padding:12px 32px;
                text-decoration:none;border-radius:6px;font-family:sans-serif;
                display:inline-block">
        Check Availability
      </a>
    </td>
  </tr>
</table>
```

---

## 5. Auto-Resize Communication (Iframe ↔ Parent)

```javascript
// Inside Angular app (iframe)
const ro = new ResizeObserver(entries => {
  window.parent.postMessage({
    type: 'breezit:resize',
    height: entries[0].contentRect.height
  }, '*');
});
ro.observe(document.body);
```

```javascript
// In breezit-widget.js (parent page)
window.addEventListener('message', (event) => {
  if (event.data.type === 'breezit:resize') {
    iframe.style.height = event.data.height + 'px';
  }
});
```

Additional events via postMessage:
- `breezit:step-change` — form step transitions
- `breezit:inquiry-submitted` — notify parent when inquiry is complete
- `breezit:close` — for modal modes, close the popup

---

## 6. Widget Configuration System

### Storage: PrivateInfo Model

Use existing `PrivateInfo` with `type: "widgetConfig"`:

```typescript
{
  user: ObjectId,
  type: "widgetConfig",
  data: {
    serviceEvent: ObjectId,
    theme: "auto" | "light" | "dark",
    brandColor: "#EF5518",
    accentColor: "#d94d16",
    widgetWidth: "100%",
    widgetHeight: "auto",
    enabledSteps: {
      eventType: true,
      eventDate: true,
      guestCount: true,
      budget: true,
      tourScheduling: true,
      message: true
    },
    customFields: [
      {
        id: string,
        label: string,
        type: "shortText" | "longText" | "select" | "multipleEmails",
        required: boolean,
        options?: string[]  // For select type
      }
    ],
    customCTA: "Check Availability",
    showBranding: true,  // "Powered by Breezit" link
    allowedDomains: string[]  // Optional domain restriction
  }
}
```

### Theme System

Angular inquiry widget accepts theme via URL query params and applies CSS custom properties:

```css
:root {
  --breezit-brand-color: var(--brand-color, #EF5518);
  --breezit-brand-hover: var(--brand-hover, #d94d16);
  --breezit-bg: var(--bg-color, #ffffff);
  --breezit-text: var(--text-color, #1f1f1f);
}
```

The Tailwind `bztt-` prefix classes are encapsulated inside the iframe — no conflicts with host site.

---

## 7. Cross-Origin Security

### Content Security Policy

Update CSP in `auth/src/app.ts` (line 47-53):

```
frame-ancestors 'self' blob: https://development.breezit.com https://platform.breezit.com *
```

The `*` allows any site to embed the widget. Alternatively, check `Referer` header against `allowedDomains` in config for per-venue restriction.

### Widget API Security

1. **No authentication** — visitors on third-party sites won't have Breezit sessions
2. **Rate limiting** — 10 submissions / IP / hour; 60 availability checks / IP / minute
3. **Validate serviceEventId** — ensure venue exists and has widgets enabled
4. **Track origin** — log `Referer` header for analytics
5. **CORS** — existing open policy works; avoid `credentials: true` for widget endpoints

### PostMessage Security

```javascript
// In breezit-widget.js (parent)
window.addEventListener('message', (event) => {
  if (!event.origin.includes('breezit.com')) return;
  // Handle message
});

// In Angular app (iframe)
window.addEventListener('message', (event) => {
  if (!event.data?.type?.startsWith('breezit:')) return;
  // Handle message
});
```

---

## 8. Do-Follow Backlink Strategy for SEO

### The Problem with Iframes

Links inside iframes do NOT pass SEO link equity. Search engines generally do not follow links inside iframes for PageRank purposes.

### The Solution: Inject Link Outside Iframe

The `breezit-widget.js` loader injects a link **in the host page DOM** (outside the iframe):

```javascript
// In breezit-widget.js, after creating the iframe
const backlink = document.createElement('a');
backlink.href = 'https://breezit.com/venues';
backlink.textContent = 'Powered by Breezit - Event Venue Marketplace';
backlink.rel = 'dofollow';
backlink.target = '_blank';
backlink.style.cssText = 'font-size:11px; color:#999; text-decoration:none; display:block; text-align:center; padding:4px 0;';
targetElement.appendChild(backlink);
```

This link lives in the host page's DOM, making it crawlable and passing link equity. The anchor text includes relevant keywords ("Event Venue Marketplace").

### Email Embed Backlinks

The CTA link in email snippets directly links to the venue's Breezit listing page — inherently a backlink when viewed in web-based email clients.

---

## 9. Cal.com Comparison: What to Adopt vs Customize

### Adopt from Cal.com

| Feature | Details |
|---------|---------|
| Iframe-based architecture | Proven pattern, perfect isolation |
| Three embed modes | Inline, floating button, popup via click |
| JS loader pattern | Single script tag, simple API |
| PostMessage communication | Height resizing and events |
| Theme configuration | Auto/light/dark, brand colors |
| Copy-paste code snippets | Low friction for venues to embed |
| Custom questions | Configurable fields with input types |

### Customize for Breezit

| Feature | Cal.com | Breezit Customization |
|---------|---------|----------------------|
| Form content | Calendar slot picker | Multi-step inquiry form with conversational flow |
| Real-time validation | Shows available slots | Checks event date availability + budget validation against venue minimums |
| Pricing logic | N/A | Seasonal/weekly pricing modifiers validated in real-time |
| Tour scheduling | IS the calendar | USES Cal.com as one integration point (optional step after inquiry) |
| Email embed | Time selection inline | Static CTA link (multi-step form can't work inline in email) |
| SEO backlinks | Not emphasized | Do-follow link injected in host DOM |
| Output | Creates booking | Creates inquiry → enters AI pipeline (Agents 0-4) |
| Custom fields | Booking questions | Venue-defined custom fields |

---

## 10. Embed Code Snippet Generation

### Vendor Dashboard UI

New section in `foxy-vendor` app under venue settings:

1. **Preview panel** — live preview of the widget
2. **Configuration panel** — theme, colors, enabled steps, custom fields
3. **Code snippets** — tabbed interface showing code for each embed mode:
   - HTML (iframe)
   - React (iframe)
   - WordPress (instructions)
   - Squarespace/Wix (code injection)
   - Email (static HTML)

### Snippet Templates

Each template has `{SERVICE_EVENT_ID}`, `{THEME}`, `{BRAND_COLOR}` placeholders replaced with the venue's values.

---

## 11. Backend Files to Modify

| File | Change |
|------|--------|
| `auth/src/modules/marketplace/martketplaceRoutes.ts` | Add `router.use("/widget", widgetRoutes)` |
| `auth/src/app.ts` | Update CSP `frame-ancestors` to allow `*` |
| `auth/src/modules/marketplace/inquiries/inquiryModel.ts` | Add `source`, `widgetOrigin`, `serviceEvent` fields |
| `auth/src/modules/marketplace/inquiries/inquiryController.ts` | Add widget-specific inquiry creation with source tracking |

---

## 12. Potential Challenges

| Challenge | Mitigation |
|-----------|------------|
| **Cross-origin cookies** | Widget API endpoints do NOT use cookies. Stateless with serviceEventId. Existing `checkAuthentication` middleware passes through unauthenticated. |
| **Iframe height changes** | ResizeObserver on iframe content. Send height updates on every step transition. |
| **Mobile responsiveness** | Iframe 100% width. Floating button expands to full-screen modal on mobile. |
| **Bundle size** | The Angular embeddable-components app already includes runtime. Inquiry widget adds ~50-100KB of component code. Monitor budget. |
| **Rate limiting** | Public APIs need strict limits: 10 submissions / IP / hour, 60 reads / IP / minute. |
| **Cal.com API keys** | Venue's key in PrivateInfo. Widget endpoint fetches server-side. Never expose to client. |
| **Stale availability** | Server-side re-check on submit, even if client-side check passed during form fill. |

---

## 13. Phased Implementation

### Phase 1: Backend Widget API (Week 1-2)
1. Create widget routes, controller, rate limiter
2. Implement check-availability (wrapper around `isServiceAvailable`)
3. Implement check-budget (new, using pricing calculator)
4. Implement venue-info (read-only, public)
5. Implement tour-slots (wrapper around `getAvailableTimesFunction`)
6. Implement submit-inquiry (Inquiry + optional Booking + optional Cal.com booking)
7. Implement get-config (reads from PrivateInfo)
8. Extend Inquiry model with source/origin fields
9. Update CSP header
10. Add widget-specific rate limiter

### Phase 2: Angular Inquiry Widget (Week 2-4)
1. Create InquiryWidgetComponent and all step components in `libs/embeddable/`
2. Add route in `embeddable-components` app
3. Implement InquiryWidgetService for API calls
4. Implement WidgetConfigService for theme/config
5. Build multi-step conversational form UX
6. Implement real-time availability checking with UI feedback
7. Implement budget validation with dynamic messaging
8. Implement Cal.com tour slot selection step
9. Implement confirmation step with backlink
10. Mobile-first responsive design

### Phase 3: JavaScript Widget Loader (Week 4-5)
1. Create `breezit-widget.ts` standalone loader
2. Build with esbuild → single `breezit-widget.js`
3. Implement `Breezit.inline()` — iframe injection + auto-resize
4. Implement `Breezit.floatingButton()` — floating button + modal
5. Implement `Breezit.popup()` — element click trigger + modal
6. Implement postMessage bridge (resize, events)
7. Implement SEO backlink injection in host DOM
8. Deploy to CDN

### Phase 4: Venue Dashboard (Week 5-6)
1. Widget configuration page in `foxy-vendor` app
2. Configuration UI (theme, colors, steps, custom fields)
3. Code snippet generator with live preview
4. Store config in PrivateInfo

### Phase 5: Email Embed + Polish (Week 6-7)
1. Static HTML email snippet templates
2. Analytics tracking (impressions, submissions, conversion rate)
3. Error handling and edge cases
4. Documentation for venues
5. Cross-browser and cross-device testing

---

## 14. Critical Files to Reuse

| File | What to Reuse |
|------|---------------|
| `auth/src/modules/marketplace/martketplaceRoutes.ts` | Register new widget routes |
| `auth/src/modules/marketplace/calendar/calendarController.ts` | `isServiceAvailable()`, `checkServiceAvailability` |
| `auth/src/services/calcomService.ts` | `getAvailableTimesFunction`, `bookEventFunction` |
| `auth/src/services/pricingCalculator/pricing-calculator.ts` | `extractServicesFromAdvert()`, `AdvertServiceVariant.getFinalUnitPrice` |
| `auth/src/modules/marketplace/inquiries/inquiryModel.ts` | Inquiry model (extend) |
| `auth/src/modules/marketplace/inquiries/inquiryController.ts` | `creteNewInquiry` logic |
| `auth/src/services/notification-center.service.ts` | Vendor notifications |
| `FoxyVendor/apps/embeddable-components/src/app/app.routes.ts` | Add inquiry widget route |
| `FoxyVendor/libs/embeddable/src/lib/embeddable/embeddable.component.ts` | Existing embeddable pattern |
| `FoxyVendor/libs/booking/src/lib/flows/components/flow-stepper.component.ts` | Step navigation pattern |
