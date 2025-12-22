LOVABLE MASTER PROMPT — UKCarGlass (Best-of-the-best)

Goal
Generate a production-ready Next.js (App Router) + TypeScript + Tailwind project for ukcarglass.co.uk.
Maximise bookings with a single /book flow (steps via /book?step=1..7).
Include trust/money pages + a small evergreen traffic engine (Tools + Guides).
Enforce strict noindex rules for transactional/programmatic pages.

Non-negotiables
- No branches, no local addresses, no location phone numbers.
- Glass type selected first (Windscreen/Rear/Side).
- Registration is primary ID; VIN fallback.
- Never guess silently. Explicit vehicle confirmation when confidence is low.
- Show price estimate BEFORE calendar selection.
- Side windows supported but not promoted (no SEO push).
- One primary CTA per page: “Get price and book”.
- No fake stats/claims. Label placeholders “REAL DATA REQUIRED”.

Tech
- Next.js App Router, TypeScript, Tailwind
- Supabase Postgres
- Stripe Checkout + webhook
- Server Actions / Route Handlers

Routes
Index:
- /, /coverage, /pricing, /how-it-works, /insurance, /adas-calibration, /warranty, /reviews, /contact, /about, /privacy, /cookies, /terms
Traffic engine (index):
- /tools, /tools/windscreen-cost-estimator
- /guides + guide pages:
  - /guides/windscreen-replacement-cost
  - /guides/insurance-vs-private
  - /guides/adas-calibration-explained
  - /guides/can-i-drive-with-a-cracked-windscreen
  - /guides/how-long-does-windscreen-replacement-take
  - /guides/repair-vs-replace
Flow (noindex):
- /book, /book/confirmed
Vehicles (curated; quality-gated):
- /vehicles + dynamic routes (noindex until quality passes)

Booking flow steps (single /book route)
Implement a stepper + state synced to URL search param step:

Step 1: glass type select
- Buttons: Windscreen / Rear / Side (Side is supported but not promoted elsewhere)

Step 2: registration input
- Validate UK reg format (soft)
- Provide “I don’t have it” fallback to manual make/model/year (mark as lower confidence)

Step 3: resolve vehicle + confirm
- call resolveVehicle(reg)
- show Make / Model / Year / Body type (+ chassis only if available)
- require explicit user confirmation if confidence_score < threshold

Step 4: refinements (progressive disclosure)
- ADAS presence yes/no with tooltip “why we ask”
- If Side chosen: show visual selector:
  - Left/Right
  - Front/Rear
  - Fixed/Vent/Drop
  - Not sure (routes to manual review flag)
- Ask one clarifying question max if ambiguous.

Step 5: quote summary (price BEFORE calendar)
- price range (low/high)
- what’s included (3–5 bullets)
- caveats (ADAS/heated/acoustic/stock)
- toggle: insurance vs private (changes explanation)
- CTA: Continue to availability

Step 6: availability select
- date + time window selector
- keep minimal

Step 7: details + address -> Stripe Checkout
- name, phone, email
- service address + notes
- final CTA: “Pay & confirm booking”
- create Stripe Checkout Session server-side

After Stripe:
- return success to /book/confirmed with booking reference
- webhook sets paid status and creates/updates booking

Backend contract (implement real stubs + integration points)
- resolveVehicle(reg) -> { vin, make, model, year, body_type, chassis?, confidence_score }
- getCoverage(postcode) -> { covered, typical_days_min, typical_days_max }
- createQuote(payload) -> { quote_id, estimated_low, estimated_high, currency, caveats[] }
- createBookingDraft(payload) -> { booking_draft_id }
- createCheckoutSession(booking_draft_id) -> { url }
- stripeWebhook() -> handles Stripe events (checkout.session.completed, expired)

Robots/meta + sitemap
- /book and /book/confirmed: <meta name="robots" content="noindex,follow">
- /vehicles/*: if page_quality < 70 then noindex,follow else index,follow
- Generate sitemap.xml including ONLY indexable pages.
- Do NOT create step pages like /step-1 etc.

Schema JSON-LD
- Global layout: Organization + WebSite
- Home: Service
- Coverage: FAQPage
- Vehicle pages: BreadcrumbList + FAQPage
Only add Review/AggregateRating schema if real, verifiable, and present in UI.

ENV VARS (README + code)
NEXT_PUBLIC_SITE_URL=
SUPABASE_URL=
SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
(optional) VEHICLE_LOOKUP_API_KEY=
(optional) WHATWINDSCREEN_API_KEY=
(optional) AUTOGLASSCRM_API_KEY=

Deliverables
- Full Next.js project
- Supabase SQL schema + seed example for coverage_rules
- Stripe checkout + webhook
- Robots/noindex gating + sitemap
- JSON-LD injection components
- README with setup + env vars

────────────────────────────────────────────────────────────
ADDITIONAL REQUIREMENTS (implement now, not later)

1) Tracking plan (events + drop-off dashboard)
Implement analytics event tracking for every step in /book.
- Use a simple internal table `analytics_events` in Supabase (server-side insert) AND optionally window.dataLayer for future GTM.
- Track these events (minimum):
  - booking_step_view (step)
  - booking_step_complete (step)
  - booking_error (step, field, error_code)
  - quote_created (quote_id, glass_type, price_low, price_high)
  - calendar_viewed
  - slot_selected (slot_id)
  - checkout_started (booking_draft_id, amount_estimate)
  - checkout_completed (booking_id, amount_paid)
  - checkout_failed / checkout_expired
  - support_click (channel=phone|chat|email, step)
- Add a simple admin dashboard at /admin/analytics (protect with basic auth env flag) showing:
  - funnel conversion per step
  - median time per step
  - top errors
  - checkout completion rate
  - coverage->book clickthrough
- A/B testing: implement feature flag variants via query param or cookie:
  - ab_variant = A|B (persist)
  - Example experiments: hero copy, trust chip order, step 2 helper text.
  - Log ab_assigned + ab_exposed (page/step) + conversion.

2) Review engine (capture + hygiene)
Create a post-job review capture flow:
- After booking status becomes completed, trigger SMS + email: “How did we do? Leave a review”
- Provide 2 links: Trustpilot + Google (if available).
- Build /reviews page to display embedded/retrieved reviews only when real.
- Add internal table `review_invites` (booking_id, channel, sent_at, provider, status).
- Templates must be honest: do not claim ratings you don’t have.

3) Ops workflow (post-payment -> dispatch)
Implement booking status lifecycle + notifications:
Statuses:
- draft -> awaiting_payment -> paid
- paid -> pending_ops -> scheduled
- scheduled -> completed | cancelled | reschedule_requested | manual_review
Operational steps:
- On webhook paid: create booking + set pending_ops
- Notify customer: confirmation SMS/email + reference + next steps
- Notify ops/dispatcher: internal email/slack stub (or dashboard queue) with booking details
- Dispatch: assign technician (placeholder table `assignments`)
- Reschedule/cancel:
  - customer can request reschedule via tokenized link or contact support
  - cancellation policy text lives on /terms and in confirmation email

4) Linkable tool launch plan (traffic engine)
Launch /tools/windscreen-cost-estimator as your first linkable asset.
- Must output honest ranges and explain the top 3 price drivers.
- Include FAQ schema and clear “Get price and book” CTA.
- Outreach starter list categories:
  - UK motoring blogs/forums
  - insurance advice sites
  - fleet management blogs
  - local news “cost of…” roundup pages
  - car enthusiast communities
- Create a one-page press angle: “New free tool: UK windscreen cost estimator (transparent ranges)”
- Track:
  - tool_usage
  - tool_to_booking_click

5) Vehicle page list (curated, quality-gated)
Start with 20–40 high-demand UK models and build truly unique pages only.
Suggested starter set (edit later):
Ford: Fiesta, Focus, Kuga
Vauxhall: Corsa, Astra
Volkswagen: Golf, Polo, Tiguan
BMW: 3 Series, 1 Series
Audi: A3, A4
Mercedes: C-Class, A-Class
Nissan: Qashqai, Juke
Toyota: Yaris, Corolla
Kia: Sportage, Ceed
Hyundai: i10, i20, Tucson
Peugeot: 208, 2008
Renault: Clio, Captur
MINI: Hatch
Skoda: Octavia, Fabia
SEAT: Ibiza, Leon
Only flip to index if page_quality >= 70.
