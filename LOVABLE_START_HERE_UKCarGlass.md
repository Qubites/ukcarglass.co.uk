# LOVABLE_START_HERE — UKCarGlass (use this file as the single source of truth)

## What we’re building
ukcarglass.co.uk is a **mobile-first UK booking platform** that dispatches vetted mobile technicians to the customer’s address.  
**No branches. No local addresses. No multiple location phone numbers.**

## Non‑negotiables (must implement)
- Glass type selected first: **Windscreen / Rear / Side**
- Registration (VRM) is primary ID; VIN fallback
- **Never guess silently**: show vehicle confirmation when confidence is low
- **Show price estimate BEFORE calendar**
- Side windows supported in flow but **not promoted**
- One primary CTA per page: **“Get price and book”**
- No fake stats/claims. Use “REAL DATA REQUIRED” placeholders

## Routes (what to build)
### Indexable money + trust pages
- / (Home)
- /coverage (postcode checker; explain mobile techs not branches)
- /pricing
- /how-it-works
- /insurance
- /adas-calibration
- /warranty
- /reviews
- /contact
- /about
- /privacy /cookies /terms

### Evergreen traffic engine (index)
- /tools
- /tools/windscreen-cost-estimator
- /guides (hub)
- /guides/windscreen-replacement-cost
- /guides/insurance-vs-private
- /guides/adas-calibration-explained
- /guides/can-i-drive-with-a-cracked-windscreen
- /guides/how-long-does-windscreen-replacement-take
- /guides/repair-vs-replace

### Transactional flow (NOINDEX)
- /book (single route; steps via /book?step=1..7)
- /book/confirmed

### Vehicles program (quality-gated)
- /vehicles
- /vehicles/[make]
- /vehicles/[make]/[model]
- /vehicles/[make]/[model]/[yearRange] (optional)
Default: **NOINDEX** until page_quality >= 70.

## Booking flow (single /book route)
Step 1: glass type  
Step 2: VRM input (+ “I don’t have it” fallback)  
Step 3: vehicle lookup + explicit confirmation  
Step 4: refinements (ADAS + side-window visual selector if side)  
Step 5: quote summary (**price before calendar**)  
Step 6: availability select  
Step 7: customer details + address -> **Stripe Checkout**  
Return to /book/confirmed; Stripe webhook finalizes booking.

## Stripe (required)
- Server-side Checkout Session creation
- Webhook handling:
  - checkout.session.completed => mark booking paid, create booking
  - checkout.session.expired => keep draft, allow retry

## UK Vehicle Data (UKVD) integration (server-side only)
Create server endpoint:
- GET /api/vehicle/lookup?vrm=AA12AAA
Rules:
- **Never expose UKVD key in the browser**
- Normalize VRM (trim/upper/remove spaces)
- Cache 24h success; cache NO_MATCH 1h
- Handle 429 nicely

## Robots + sitemap rules (must implement)
- /book, /book/confirmed and /book?step=* => **noindex,follow**
- /vehicles/*: if page_quality < 70 => **noindex,follow**
- sitemap.xml must include ONLY indexable pages

## Schema JSON-LD
- Global: Organization + WebSite
- Home: Service
- Coverage: FAQPage
- Vehicle: BreadcrumbList + FAQPage
No Review/AggregateRating unless real and shown.

## Tracking + ops (must implement)
- Track events for each booking step + checkout
- Create /admin/analytics funnel dashboard
- Implement booking lifecycle + notifications (customer + ops queue)

## ENV VARS (placeholders; add real values later)
NEXT_PUBLIC_SITE_URL=
SUPABASE_URL=
SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
UKVD_API_BASE_URL=https://uk.api.vehicledataglobal.com
UKVD_API_KEY=
UKVD_PACKAGE_NAME=VehicleDetailsWithImage
(optional) WHATWINDSCREEN_API_KEY=
(optional) AUTOGLASSCRM_API_KEY=
