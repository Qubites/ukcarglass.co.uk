# UKCarGlass — Full Automation Spec (Lovable Upload)

This document describes the minimum complete workflow to automate UKCarGlass end‑to‑end:
- Customer books and pays (Stripe)
- System offers the job to technicians (accept/decline/propose)
- Customer and technician manage everything via short-link portals (email stays short)
- Technician completes job with required evidence (photos + odometer + damage notes)
- Admin is alerted for exceptions (margin, delays, changes)

---

## 0) Principles / Non‑Negotiotiables
1) Email is short (status + 1 button). All details live on the web portal.
2) Magic links for customer and technician portals (no passwords required).
3) Deterministic state machine for bookings + job offers.
4) Margin guardrails: if tech cost breaks margin, booking becomes `needs_admin`.
5) Editable templates stored in DB (simple HTML + text; deliverability first).

---

## 1) State Machines

### 1.1 Booking status (`bookings.status`)
- `draft` (in booking flow)
- `pending_payment`
- `paid`
- `awaiting_tech` (offers sent)
- `scheduled` (time agreed)
- `in_progress`
- `completed`
- `cancelled`
- `needs_admin` (margin/time/exception)

### 1.2 Technician offer status (`job_offers.status`)
- `offered`
- `accepted`
- `declined`
- `proposed_new_times`
- `expired`

---

## 2) Web Routes (UI)

### Customer
- `/book` (existing 7-step flow; noindex)
- `/b/:token` (customer portal; magic link)
  - view status, address, vehicle, service, schedule, price breakdown
  - request cancel/reschedule (simple form)
  - see technician updates and completion evidence

### Technician
- `/t/:token` (technician portal; magic link)
  - Accept / Decline
  - Propose new time slots (2–5 options)
  - Enter tech service cost + remote fee
  - Upload evidence:
    - 3 photos (front vehicle, odometer, damage)
    - odometer_km
    - damage_notes
  - Mark job complete (enabled only when required evidence is present)

### Admin
- `/admin` (protected)
- `/admin/bookings` (filters: awaiting_tech, needs_admin, scheduled, completed)
- `/admin/bookings/:id` (approve price, change tech, change date, overrides, audit)
- `/admin/templates` (edit email templates)
- `/admin/settings` (SLA timers, margin rules, offer expiry)

---

## 3) Database (Supabase) — Tables

### 3.1 `bookings`
- `id` (uuid, PK)
- `ref` (text, unique short ref e.g. UKCG-9F3K)
- `status` (text)
- customer: `customer_name`, `customer_email`, `customer_phone`
- address: `address_line1`, `address_line2`, `city`, `postcode`
- vehicle: `vehicle_reg`, `vehicle_make`, `vehicle_model`, `vehicle_year`, `vin` (optional)
- `glass_position` (front/rear/side)
- schedule: `scheduled_start` (timestamptz), `scheduled_window` (text)
- prices:
  - `base_price_pence` (int)
  - `zone_surcharge_pence` (int default 0)
  - `customer_price_pence` (int)  // base + surcharge + approved adjustments
- Stripe:
  - `stripe_session_id`, `stripe_payment_intent_id`
- assignment:
  - `current_technician_id` (int nullable)
- `customer_portal_token` (text unique)
- `created_at`, `updated_at`

### 3.2 `job_offers`
- `id` (uuid, PK)
- `booking_id` (uuid FK -> bookings.id)
- `technician_id` (int)
- `status` (text)
- `expires_at` (timestamptz)
- `responded_at` (timestamptz nullable)
- `token` (text unique)  // used by /t/:token
- `created_at`

### 3.3 `job_proposals`
- `id` (uuid, PK)
- `job_offer_id` (uuid FK -> job_offers.id)
- `proposed_slots` (jsonb)  // array of {start,end,label}
- `selected_slot` (jsonb nullable)
- `selected_at` (timestamptz nullable)

### 3.4 `job_pricing`
- `booking_id` (uuid PK/FK)
- `technician_cost_pence` (int nullable)
- `remote_fee_pence` (int nullable)
- `margin_pence` (int nullable)
- `margin_pct` (numeric nullable)
- `requires_admin_approval` (bool default false)
- `approved_by_admin` (bool default false)
- `approved_at` (timestamptz nullable)

### 3.5 `job_reports`
- `booking_id` (uuid PK/FK)
- `odometer_km` (int)
- `damage_notes` (text)
- `photo_front_url` (text)
- `photo_odometer_url` (text)
- `photo_damage_url` (text)
- `completed_at` (timestamptz)

### 3.6 `email_templates`
- `key` (text PK)
- `subject` (text)
- `preheader` (text)
- `body_html` (text)
- `body_text` (text)
- `updated_at`

### 3.7 `notification_events` (audit log)
- `id` (uuid, PK)
- `booking_id` (uuid nullable)
- `actor` (text)  // system/customer/tech/admin
- `type` (text)
- `payload` (jsonb)
- `created_at`

### 3.8 `postcode_cache`
- `postcode` (text PK)
- `lat` (float)
- `lng` (float)
- `updated_at` (timestamptz)

---

## 4) Access Model
- Customer portal: access via `customer_portal_token` only.
- Technician portal: access via `job_offers.token` only.
- Admin: Supabase Auth role or allowlist.

---

## 5) Automation (Edge Functions / Background Jobs)

### 5.1 Stripe webhooks
On `checkout.session.completed`:
- booking.status -> `paid`
- create `customer_portal_token` if missing
- trigger `dispatch_offers(booking_id)`
- send `customer_booked_paid` email

### 5.2 Offer dispatch (`dispatch_offers`)
- select eligible technicians by coverage zone
- create N offers (start N=2 or 3)
- `expires_at = now + OFFER_TTL_MINUTES`
- send `tech_offer` email to each
- booking.status -> `awaiting_tech`

### 5.3 Offer expiry worker
Runs every 5–10 min:
- expire offers past TTL
- if booking still `awaiting_tech` and no accepted offers:
  - try next technician batch
  - if none -> booking.status=`needs_admin`, notify admin

### 5.4 SLA watchdog
Runs every 10–30 min:
- if `awaiting_tech` > X minutes -> admin email
- if `paid` but no offers created -> admin email
- optional: scheduled but not started by time+buffer -> admin email

### 5.5 Booking change notifier
When important fields change (schedule, price, delays, cancel):
- send `customer_booking_updated`
- send `tech_booking_updated`
- write `notification_events`

---

## 6) Technician Actions

### Accept
- offer.status -> accepted
- booking.current_technician_id set
- booking.status -> `scheduled` (or keep scheduled if time already set)
- send customer email `customer_tech_accepted`

### Decline
- offer.status -> declined
- if all offers declined -> dispatch next techs or admin alert

### Propose new times
- offer.status -> proposed_new_times
- create `job_proposals` with 2–5 slots
- email customer `customer_time_proposal`
Customer selects in `/b/:token`:
- booking schedule set
- email tech `tech_time_selected`

### Enter pricing (cost + remote fee)
- compute margin
- if below thresholds:
  - `job_pricing.requires_admin_approval=true`
  - booking.status=`needs_admin`
  - notify admin

### Complete job
Require 3 photos + odometer + damage notes:
- create `job_reports`
- booking.status -> completed
- email customer `customer_completed`

---

## 7) Email Templates (Editable, deliverability-first)
Templates to create:
Customer:
- `customer_booked_paid`
- `customer_tech_accepted`
- `customer_time_proposal`
- `customer_booking_updated`
- `customer_completed`

Technician:
- `tech_offer`
- `tech_time_selected`
- `tech_booking_updated`
- `tech_cancelled`

Admin:
- `admin_needs_admin`
- `admin_unconfirmed_timeout`
- `admin_no_tech_available`

Rules:
- mostly text, 1 logo max
- 1 primary button to portal
- include plain-text version
- transactional provider recommended

---

## 8) Config (Admin Settings)
- `OFFER_TTL_MINUTES` (default 60)
- `AWAITING_TECH_TIMEOUT_MINUTES` (default 60)
- `MIN_PROFIT_PENCE` (example 5000)
- `MIN_MARGIN_PCT` (example 0.15)

---

## 9) Implementation Order (fast)
1) Create tables + admin list
2) Build `/t/:token` portal (accept/decline/propose)
3) Dispatch offers after Stripe paid
4) Add pricing entry + margin alerts
5) Add completion evidence upload + completion email
6) Add proposal slot selection in customer portal
