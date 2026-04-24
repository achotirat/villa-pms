# Plan: Custom PMS / Booking Engine / Channel Manager for Pool Villas

## Context

The user runs a pool villa rental business in Ao Nang, Krabi, Thailand — currently 4 villas ("Casa de Yim" is the flagship), scaling to ~15 villas by end of 2026. Today the business runs on **eZee Yanolja** (commercial PMS + channel manager pushing to Airbnb and Booking.com) with **Stripe + Xendit** handling direct card payments. Payment policy: authorize at booking, charge 100% at T-10 days, non-refundable rate tiers available. Min stay is set to 2 nights.

**The pain is the eZee UI**, not its capabilities. The user wants to replace it with a purpose-built system that matches their workflow, saves OTA commission via a stronger direct-booking funnel, and produces cleaner reports.

### Existing assets discovered in user's folders (must be preserved/extended, not replaced)

| Asset | Location | What it does | Plan impact |
|---|---|---|---|
| **Casa de Yim marketing site** | `Document/Claude_cowork/PROJECTS/Casa de Yim/Casa de Yim Website/` | Deployed at `casadeyim.com` via Netlify. Static HTML + Netlify Functions. | Keep as-is; embed booking engine + link to self-check-in |
| **Guest services / self-check-in page** | `Document/VS-code/Guest booking framework/guest-services.html` + Netlify functions `create-payment.js`, `get-booking.js` | Already handles TM30 immigration forms, bed setup, island-tour booking with Xendit checkout, airport transfer. Reads booking from eZee API. Falls back to WhatsApp (+66635614563). | **This IS the self-check-in app.** Refactor for multi-villa and swap eZee API → QloApps API. Do NOT rebuild from scratch. |
| **Dynamic rate manager** | `.../Casa de Yim/casa_de_yim_rate_manager/` | Python + Selenium automation that reads occupancy from eZee and pushes calculated rates. Config in `config.yaml` with real 2025 data: seasons (High Nov–Jan base 9,500; Mid Feb–Apr+Oct base 7,500; Low May–Sep base 5,500), occupancy-threshold multipliers (0.85×→1.40×), special dates (NYE 14,000). | **Pricing logic already exists.** Port `config.yaml` semantics into the QloApps pricing-engine module; retire the Selenium script once QloApps CM is live. |
| **Accounting pipeline** | `.../Casa de Yim/casa-accounting/` | Python scripts convert Airbnb + Booking.com XLSX exports to **FlowAccount** format (Thai accounting SaaS). Monthly markdown reports. | Reports module must export FlowAccount-compatible CSV/XLSX to preserve this workflow. |
| **Direct Booking Plan 2026** | `.../Casa de Yim Website/Casa_de_Yim_Direct_Booking_Plan_2026.docx` | User's own strategic plan doc for 2026 direct-booking push. | Referenced as strategic context; aligns with scope. |

**Critical correction to earlier scope:** the user's existing rate manager already uses **occupancy-pressure pricing** (forward 14-day occupancy → multiplier 0.85×–1.40×). Earlier in this brainstorm I listed occupancy-pressure as "out of scope" — that was wrong. It's in. The pricing engine combines: season tier × occupancy multiplier × special-date override × min/max clamp.

### Reconciliation with `Casa_de_Yim_Direct_Booking_Plan_2026.docx`

The user's 2026 plan is a 12-month marketing-driven roadmap to grow direct bookings from **<5% to 25% of revenue** (target ฿3.4M direct revenue / year). 2025 baseline: ฿7.48M gross, 249 bookings, 100% OTA (Booking.com 54%, Airbnb 46%, 3 direct). Marketing budget ฿10,000/month. Owner in Bangkok, visits Krabi 2×/month × 1–2 nights. On-site caretaker does not speak English.

**This PMS replacement must enable every tactic in the 2026 plan.** Marketing goals are the "why" — system capabilities are the "how." The following are non-negotiable v1 requirements derived from the 2026 plan and folded into this plan:

- **Direct-only rate plan** at **8% below BAR**, hidden from OTA channels
- **Premium pricing for Villa A4** (separate room type, +฿800–1,200 vs A1–A3)
- **Packages / add-ons**: Romance (flower decor, ฿500 add), Family Splash (pool toys + early check-in, ฿300), Long Stay 5+ nights (**free airport transfer** — user confirmed this is part of direct strategy), Group Party, Low Season Escape, Early Bird Peak
- **Min-stay restrictions**: 2 nights standard, 3 nights Dec 20–Jan 5 and Songkran
- **Group booking flow** (2+ villas, 8+ guests) — direct quote response within 1h, ฿500–1,000/villa/night below OTA
- **Google Hotel Ads / Free Booking Links** connectivity (QloApps CM + Google connector)
- **Email capture at booking** + Mailchimp webhook for post-stay automation
- **Guest nationality tagged on every booking** (enables segmented campaigns)
- **Review aggregation** (link-out or embedded view of Booking.com + Airbnb + Google reviews per villa) — eZee Critique replacement
- **WhatsApp / LINE contact buttons** on booking engine and confirmations (owner WhatsApp: +66635614563)
- **Smart-lock vendor pivot:** user is migrating away from Wishome to **Tuya** or **TTlock**. Abstraction layer stays; v1 adapters target Tuya + TTlock (both have well-documented REST APIs and Thai availability). Wishome adapter dropped.
- **Rate-parity alert** — dashboard flag if any OTA is showing a rate below the direct rate (inverse of common use; here the direct IS meant to be lowest, but we still watch for anomalies)
- **Remote-friendly admin** — everything accessible from phone (owner works from Bangkok + Krabi visits)

### Direct Pricing & Rate Parity Strategy (user-confirmed)

Unlike the default "match OTA pricing to stay in compliance" assumption: **the user intentionally prices direct LOWER than OTAs** (8% below BAR) or matches a promotional OTA price and layers **free airport transfer** on top as the differentiator. This is a known-but-accepted rate-parity risk with Booking.com.

**System support required:**
- Direct-only rate plans invisible to OTAs (QloApps CM needs per-plan channel gating)
- "Free airport transfer" as an auto-attached add-on to any direct booking ≥ 5 nights (tracked in `extra_service_booking` with price 0 and note "long-stay perk")
- Reporting that tracks when OTA BAR drops below our direct rate so we can adjust or trigger promotions

**Scope (confirmed through brainstorming):**
- **Booking engine** integrated with the **existing marketing website** (user has already built it — we augment, not replace; see "Web Surface" section)
- **Two-brand, three-tier architecture**: 9 villas under Casa de Yim + 6 villas under Brand B (name TBC, site build deferred to later phase). Three room tiers across both brands:
  - **Tier A** (A1–A4, 4 villas): target ADR **฿7,000** — entry level
  - **Tier B** (B1–B4, 4 villas): target ADR **฿10,000** — mid
  - **Tier C** (7 units: premium, twin house, others): target ADR **฿12,500** — premium
  
  Modeled as **2 QloApps "hotels"** (one per brand), each with the relevant tiers as QloApps room types. Per-tier base rates and season multipliers; per-brand marketing sites, rate plans, email templates.
- **Admin dashboard** — calendar, bookings, rates, reports, OTA task list
- **Housekeeper dashboard** — mobile-first PWA, today's check-ins / check-outs / stayovers, cleaning status per villa
- **Self-check-in app** for guests — pre-arrival info collection (TM30 Thai immigration reporting), digital rental agreement, door/gate code issuance, in-stay property guide, checkout flow
- **Extra services / upsells** — airport transfers, private chef, breakfast, boat tours, spa, etc. — bookable at checkout and during stay via the self-check-in app
- Channel sync via **QloApps' Channel Manager module** (API-based → closes the 15-min iCal gap; see "Channel Sync" section)
- Dynamic pricing: **seasonal tiers + forward-occupancy multipliers + special-date overrides + min/max clamps + min-stay rules** (port of user's existing `rate_manager/config.yaml` logic). No weekend premium, no event calendar beyond special-date overrides, no competitor scraping.
- Reports: Occupancy %, ADR, RevPAR per villa and aggregate
- Migration: bring eZee bookings + historical revenue into the new system so reports have real history on day one
- Build approach: **fork an open-source PMS (QloApps) and customize**

**Out of scope for v1:**
General guest messaging / CRM, weekend/occupancy-based pricing, event pricing, Agoda/Expedia channels, loyalty program, multi-currency display beyond THB + the guest's home currency shown by Stripe/Xendit, inventory/maintenance/payroll ops. (Housekeeping, self-check-in, and extras are **in** scope as lean focused modules.)

---

## Important Caveat on "Fork an OSS PMS"

Most mature OSS PMS options are PHP/MySQL with dated UIs (Hotel Druid, QloApps, SolidRes). Forking one and then *replacing its UI* — which is the user's primary motivation, since eZee's UI is the thing that hurts — often ends up as a larger rewrite than starting greenfield. **The plan below uses QloApps as the fork target** because it has the best booking engine + multi-property + payment gateway foundation of any current OSS option, but a **Phase 0 "spike"** is included to validate the fork is less work than a greenfield modern stack (Next.js + Postgres) before committing.

---

## Recommended Base: QloApps

**Why QloApps:**
- Full booking engine with calendar, rate plans, cancellation policies, non-refundable rates — matches the user's existing payment policy out of the box
- Multi-property / multi-room support built in
- Payment gateway modules (Stripe has an official module; Xendit requires a custom module)
- Multi-language / multi-currency front office
- Active community, AGPL-3.0 licensed, Prestashop-based so theming is well-documented
- Has a channel manager module concept (though its native CM is paid) — we will replace that layer with our own iCal sync + task-list generator

**Alternatives surveyed:**
- **Hotel Druid** — simpler, PHP, but dated UI and limited extensibility
- **Greenfield Next.js + Postgres** — full control and modern UI, but ~2–3× the build time; reconsidered after Phase 0 spike if QloApps fork proves heavier than expected

---

## Architecture

```
  casadeyim.com (Netlify, existing)     guest-checkin.casadeyim.com (Netlify)
  ┌─────────────────────────────┐     ┌──────────────────────────────────┐
  │ Marketing site (existing)   │     │ Guest self-check-in PWA          │
  │  + embedded booking widget  │     │ (refactored guest-services.html) │
  │  + Xendit/Stripe checkout   │     │ TM30 · agreement · door codes    │
  └──────────────┬──────────────┘     │ · extras catalog · property guide│
                 │                    └──────────────┬───────────────────┘
                 │                                   │
                 ▼                                   ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │                    QloApps Core (forked)                          │
  │  Properties · Rate Plans · Seasons · Bookings · Guests ·          │
  │  Payments (Stripe + custom Xendit) · Extras · Users · TM30 data  │
  └─────┬─────────────────┬────────────────────┬──────────────────┬──┘
        │                 │                    │                  │
        ▼                 ▼                    ▼                  ▼
  ┌───────────┐  ┌────────────────┐  ┌─────────────────┐  ┌──────────────┐
  │ Channel   │  │ Pricing Engine │  │ Check-in Engine │  │ Reports +    │
  │ Manager   │  │ (cron)         │  │ (door codes,    │  │ FlowAccount  │
  │ addon     │  │ Season × occ × │  │  TM30 export,   │  │ XLSX export  │
  │ ↔ Airbnb  │  │ specials →     │  │  reminders)     │  │ (feeds       │
  │ ↔ Booking │  │ rates, real-   │  │                 │  │ casa-        │
  │ real-time │  │ time via CM    │  │                 │  │ accounting)  │
  └───────────┘  └────────────────┘  └─────────────────┘  └──────────────┘
        │                 │                    │                  │
        └─────────────────┼────────────────────┼──────────────────┘
                          ▼                    ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │                 Admin Dashboard (QloApps back office)             │
  │  Calendar · Bookings · Rates · Reports · Extras · Housekeeping   │
  └──────────────────────────────────────┬───────────────────────────┘
                                         ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │           Housekeeper Dashboard (mobile-first PWA)                │
  │  Today's arrivals · departures · stayovers · clean/dirty state   │
  └──────────────────────────────────────────────────────────────────┘
```

**Key architectural choices:**
- The forked QloApps is the source of truth for properties, bookings, payments, extras, and users
- **Channel sync uses QloApps' Channel Manager module (API-based)** — eliminates the 15-min iCal polling gap the user raised. See "Channel Sync" section.
- The **booking engine embeds into the existing marketing website** rather than replacing it — user has already invested in the marketing surface
- Pricing engine and Check-in engine are external workers (cron) reading/writing QloApps' DB directly — keeps upstream QloApps upgradable
- Admin dashboard extends QloApps' back office with custom modules for pricing, reports, extras catalog, housekeeping, and check-in workflow — everything else uses native QloApps admin

---

## Core Modules

| # | Module | Location | Notes |
|---|--------|----------|-------|
| 1 | Property catalog | QloApps native | Villas, photos, amenities, capacity |
| 2 | Rate plans | QloApps native + custom | "Standard" + "Non-refundable" tier already supported; extend with season × lead-time logic |
| 3 | Booking engine | Custom Next.js front-end → QloApps API | The main UX investment |
| 4 | Payment orchestrator | QloApps Stripe module + **custom Xendit module** | Replicate current policy: auth at book → charge at T-10 → handle non-refundable as charge-at-book |
| 5 | **Channel sync** | **QloApps Channel Manager module (API-based)** | Real-time API sync with Airbnb + Booking.com via QloApps' CM addon (or its partner middleware, e.g. STAAH / SiteMinder). **Closes the 15-min iCal overbooking window** per user's concern. iCal kept only as fallback. |
| 7 | Pricing engine | New (Node.js cron) | Port of user's existing `rate_manager/config.yaml` logic: season tier × forward-occupancy multiplier × special-date override × min/max clamp; daily recompute across all villas × next 365 days; writes to QloApps rate tables |
| 8 | OTA rate-update task list | New (QloApps module) | Daily "update these villas to these rates on Airbnb / Booking.com" checklist, with copy buttons |
| 9 | Reports | New (QloApps module) | Occupancy %, ADR, RevPAR — per villa, per month, aggregate. **Exports FlowAccount-compatible CSV/XLSX** (matches user's existing accounting pipeline in `casa-accounting/`). Historical data from eZee import populates reports on day one. |
| 10 | **Housekeeper dashboard** | **New (Next.js route, mobile-first PWA)** | Today's arrivals / departures / stayovers per villa; cleaning state machine (dirty → cleaning → clean → inspected); simple per-housekeeper login; no payroll / task detail in v1 |
| 11 | **Self-check-in app** | **Refactor of existing `guest-services.html` + Netlify functions** | Multi-villa support; TM30 data collection + submission-ready PDF (manual submit by owner, per user); digital rental agreement with signature (**new — no template exists today, requires lawyer engagement**); door/gate code issuance via **Wishome API** (with abstraction layer to swap lock brands later as user may migrate); in-stay property guide; **extras catalog** bookable with Xendit; WhatsApp fallback preserved. Backend endpoint swaps eZee API → QloApps API. |
| 12 | **Extra services catalog** | New (QloApps module) | Admin CRUD for extras (airport transfer, private chef, island tours, spa, breakfast, welcome flowers, pool toys). Exposed to booking-engine checkout and self-check-in app. Xendit handles payment. Per-villa availability and pricing. Auto-attach free airport transfer to direct bookings ≥ 5 nights. |
| 13 | **Smart-lock integration** | New (abstraction layer + Tuya + TTlock adapters) | Interface: `issue_code(booking, start, end) → code` / `revoke_code(code)`. Adapter targets: **Tuya Cloud API** and **TTlock Open Platform API** — user is migrating off Wishome to one of these two. Phase 0 decides which becomes v1 adapter (likely TTlock: more mature lock-specific API, free developer account, PMS integrations already documented). Fallback: manual code rotation tracked in admin if API not available for a villa. |
| 14 | **Marketing integrations** | New (QloApps module + Netlify functions) | Mailchimp webhook on booking create (captures email + nationality → triggers post-stay automation); Google Hotel Ads / Free Booking Links connector (via QloApps CM); review-aggregation dashboard linking Booking.com + Airbnb + Google reviews per villa. |
| 15 | eZee import | One-off script | CSV export from eZee → transform → QloApps DB |

---

## Data Model (new / extended entities)

Existing QloApps entities used as-is: `hotel_branch_info` (property), `htl_room_type`, `htl_booking_detail`, `customer`, `orders`, `ps_order_invoice`.

**New tables:**
- `season_tier` — `(id, name, months_json, base_rate, min_rate, max_rate, property_id)` — mirrors user's existing `config.yaml` season structure (High/Mid/Low with months, base, min/max clamp)
- `occupancy_rule` — `(id, occupancy_pct, multiplier, property_id, lookahead_days)` — mirrors `config.yaml` occupancy thresholds (0%→0.85×, 25%→0.95×, 50%→1.05×, 75%→1.20×, 100%→1.40×)
- `special_date` — `(id, date, override_rate, note, property_id)` — NYE, Songkran, etc.
- `min_stay_rule` — `(id, season_tier_id, min_nights, property_id)`
- `pricing_run_log` — `(id, started_at, finished_at, rates_written_count, errors_json)` — replaces `rate_log.csv`
- `extra_service` — `(id, name, description, price_thb, category [transfer|tour|meal|spa|other], property_ids_json, active)`
- `extra_service_booking` — `(id, booking_id, extra_service_id, qty, scheduled_date, price_thb, xendit_invoice_id, status)`
- `housekeeper` — `(id, name, phone, pin_hash, assigned_property_ids_json, active)`
- `cleaning_status` — `(id, property_id, date, state [dirty|cleaning|clean|inspected], set_by_housekeeper_id, set_at)`
- `booking_hold` — `(id, property_id, start_date, end_date, session_id, created_at, expires_at)` — short-lived holds for overbooking protection
- `overbooking_alert` — `(id, property_id, dates_json, detected_at, source_a, source_b, resolved_at, resolution_note)`
- `guest_checkin` — `(id, booking_id, tm30_submitted_at, agreement_signed_at, door_code, door_code_expires_at, wifi_password_issued, status)`
- `tm30_guest_entry` — `(id, guest_checkin_id, name, passport_no, nationality, dob, arrival_date, issued_to_immigration_at)` — Thai immigration reporting
- `brand` — `(id, name, domain, logo_url, primary_color, email_from_address, whatsapp_number)` — two brands: Casa de Yim (live) + Brand B (stub, marketing site deferred)
- `property_brand` — `(property_id, brand_id)` — maps each villa to its brand
- `tier` — `(id, code [A|B|C], name, target_adr_thb, base_rate_thb)` — three tiers: A (฿7k), B (฿10k), C (฿12.5k); seasonal multipliers applied on top of tier base rate
- `property_tier` — `(property_id, tier_id)` — each villa belongs to one tier
- `smart_lock` — `(id, property_id, lock_provider [tuya|ttlock|manual], external_device_id, last_synced_at)`
- `package` — `(id, name, brand_id, includes_json, addon_price_thb, requires_min_nights, active)` — Romance / Family / Long Stay / Group / Low Season / Early Bird
- `review_link` — `(id, property_id, channel [booking|airbnb|google], url, last_scraped_avg_rating, last_scraped_count, last_scraped_at)`

---

## Web Surface Strategy (Two-Brand)

**Confirmed:** 9 villas under Casa de Yim brand, 6 villas under a second brand (name TBC — "Brand B" below).

**Layout:**
- `casadeyim.com` (existing, Netlify) — keeps its current pages; new booking widget embedded via `<script>` tag, scoped to the 9 Casa de Yim villas
- `brand-b.com` (new, Netlify) — new marketing site, same widget component, scoped to its 6 villas
- `checkin.casadeyim.com` and `checkin.brand-b.com` — two deployments of the refactored self-check-in PWA, brand theming via `brand_id` from booking-ref
- `admin.casadeyim.com` — one shared admin surface (QloApps back office on a VPS) for the owner managing both brands
- Shared Netlify project serves both front-ends from the same booking-widget build; brand theming via env vars + CSS variables

**Guest geography note (from user):** mostly European high season, Asian mid/low, 1% Thai. **Hosting decision:** DigitalOcean Singapore region for QloApps core + MySQL. Netlify CDN handles front-end globally. Singapore gives:
- ~60–70ms latency to Bangkok caretaker
- ~180ms to Europe (acceptable for admin; guests hit CDN-cached front-ends)
- Good connectivity to Chinese / Malaysian / Indian guests in mid/low season

**Per-brand configuration (in `brand` table):** logo, primary color, domain, WhatsApp number, email sender identity, rate-plan defaults, package catalog. Everything else (calendar, pricing, bookings, reports) is unified across brands at the admin level — owner sees one dashboard with brand filter.

---

## Channel Sync (Overbooking Protection)

**Approach:** use QloApps' Channel Manager module, which syncs with Airbnb and Booking.com via their APIs (directly or through partner middleware such as STAAH / SiteMinder / Bookinglayer — QloApps supports all three via its native CM addon). This is API-based and pushes updates in near-real-time (seconds, not 15 min), which directly addresses the overbooking concern.

**What this gives us:**
- Real-time inventory updates when an OTA booking comes in → direct site availability reflects the block within seconds
- Real-time rate push → when our pricing engine computes new rates, they're live on Airbnb / Booking.com automatically (no more manual "update OTA task list")
- Restrictions (min stay, closed-to-arrival) sync too

**What this costs (verified by user with QloApps support):**
- QloApps pricing is **per "hotel / property instance"** in QloApps, **not per room/villa**. A single QloApps hotel can host multiple room types and unlimited rooms.
- **Plan:** model each brand as one QloApps hotel → **2 QloApps hotels** (Casa de Yim + Brand B). CM subscription = **~$60/month total** ($30 × 2). At $300/yr/property annual billing: **~$600/year total**.
- Confirm final pricing and integration architecture (direct API vs middleware) with QloApps support during Phase 0.
- **Budget sanity check:** user's commission-savings target is ~฿540,000/year (per 2026 plan). QloApps CM at ~฿22,000/year → **~25× ROI**. Very acceptable.
- Initial setup: OTA account linking, rate-plan mapping, test bookings in each direction (Phase 0 work).

**Defense-in-depth for edge cases** (even with API CM, sync isn't truly instantaneous):
1. **Direct-booking holds.** When a guest reaches checkout on the public site, insert a short-lived `booking_hold` row (15-min TTL) that blocks those dates for *other direct bookings* immediately — before payment completes. Released if payment fails or times out.
2. **Pre-commit recheck.** Just before confirming payment on a direct booking, call the QloApps CM module to recheck OTA availability for the requested villa/dates. If an OTA has just claimed them, abort and void the Stripe/Xendit auth.
3. **Collision detection on inbound webhook.** When the CM receives an inbound OTA booking that overlaps an existing direct booking, write an `overbooking_alert` row and notify the owner within 1 min (email + push). Owner resolves manually.
4. **Arrival buffer.** Direct site refuses bookings with check-in less than **24 hours** away unless manually opened — configurable per villa.

**Fallback:** if QloApps' CM module proves unreliable or unaffordable, plan B is a custom iCal poller at 3-min intervals (documented as deferred alternative, not v1 path).

**Test plan for overbooking protection:** see Verification section.

---

## Migration from eZee

eZee supports CSV / Excel exports from its reports module.

1. Export from eZee: **reservations** (past + future), **guest list**, **rate setup**, **property list**, **payments**
2. One-off Node.js importer transforms CSVs → QloApps DB inserts:
   - Properties → `hotel_branch_info`
   - Rates → seed one "Standard" plan per property as baseline
   - Past bookings → `htl_booking_detail` with a `source=ezee_import` flag, no payment state (historical reports only)
   - Forward bookings → full booking records with payment state reconciled against Stripe/Xendit
3. Run import in staging, validate occupancy and revenue totals match eZee's reports for the same period before cutover
4. Cutover: freeze eZee edits → re-run import for final delta → switch DNS / OTA iCal URLs → monitor 72h

---

## Phases

**Phase 0 — Fork & Plugin Spike (1–2 weeks, de-risk)**
- Fork QloApps, run locally
- **Email Webkul support first** to confirm each plugin's last release date and current-version compatibility (see freshness caveat above). Skip any plugin untouched >18 months.
- **Install the three vetted paid plugins** (~$273 total): Front Desk System, Tours & Packages, Abandoned Cart Reminder.
- Add one villa, one rate plan, take one test booking with the official Stripe module
- Install **QloApps Channel Manager addon**, link test Airbnb + Booking.com accounts, verify one round-trip booking sync in each direction
- Test plugin capabilities against our requirements (see "Phase 0 plugin-evaluation checklist" above) — decide buy-vs-build for extras / front desk / CM path
- **Decision gate:** back-office UX is tolerable after plugins AND QloApps native CM works end-to-end → proceed. Otherwise pivot to greenfield Next.js + custom iCal CM (3-min polling).

**Phase 1 — Foundation (4–5 weeks)**
- All 4 current villas modeled in QloApps under Casa de Yim brand; brand schema + Brand B scaffold ready
- Booking widget built as an embeddable React component + checkout pages, deployed to Netlify, integrated into existing `casadeyim.com`
- Direct-only rate plan (8% below BAR) configured and gated out of OTA channels
- Packages catalog seeded: Romance / Family Splash / Long Stay / Early Bird (per 2026 plan)
- Villa A4 modeled as premium room type with +฿800–1,200 premium
- Stripe module + **custom Xendit module** (auth-at-book, charge-at-T-10 cron, non-refundable variant matching user's current policy)
- QloApps Channel Manager module live for Airbnb + Booking.com (replaces Selenium rate-push and iCal sync)
- **Overbooking protection:** booking holds + pre-commit recheck + collision alerts
- Basic admin calendar usable in QloApps back office
- `casa-accounting` Python scripts re-pointed at QloApps export format (or kept as-is consuming the FlowAccount XLSX export from the new reports module)

**Phase 2 — Pricing, Reports, Housekeeping, Extras (3–4 weeks)**
- Pricing engine: port `rate_manager/config.yaml` semantics to QloApps module (seasons + occupancy multipliers + special dates + min/max clamps); daily cron; writes rates through CM module to OTAs
- Admin UI for pricing rules (YAML-importable so user can keep their `config.yaml` as source of truth)
- Occupancy / ADR / RevPAR reports module + **FlowAccount XLSX export**
- **Housekeeper dashboard (mobile PWA)**
- **Extra services catalog** (admin CRUD + checkout surface + self-check-in surface)
- eZee CSV importer (historical bookings + forward bookings)
- Retire Selenium `rate_manager_selenium.py` once QloApps pricing engine is pushing live
- **Marketing integrations:** Mailchimp webhook (capture email + nationality on booking), Google Hotel Ads connector via QloApps CM, review-aggregation dashboard per villa

**Phase 3 — Self-Check-in Refactor + Cutover (3–4 weeks)**
- Refactor existing `guest-services.html` → multi-villa, multi-brand self-check-in PWA
  - Swap eZee API calls for QloApps API in Netlify functions
  - Generalize hard-coded Casa de Yim content → per-villa content pulled from QloApps property fields, per-brand theming
  - **Door/gate code issuance via Tuya or TTlock API** (decided in Phase 0; fallback is shared-code-pool where owner generates in vendor app and pastes into admin)
  - Surface extras catalog in-stay; keep Xendit tour payment flow
  - Preserve WhatsApp fallback (owner: +66635614563)
- **Digital rental agreement:** deferred to later phase (no template exists today; requires lawyer work). Check-in flow v1 ships without e-signature step.
- **TM30 immigration reporting**: per-guest submission-ready PDF generated from check-in data; owner submits manually at tm30.immigration.go.th
- Full eZee migration import (bookings + historical revenue)
- Parallel run week (eZee shadow mode)
- DNS + OTA relinking to new system, eZee shutdown
- Onboard villas 5–15 under correct brand (9 Casa de Yim, 6 Brand B) as they come online

**Phase 4 — Deferred (not in v1)**
- **Brand B marketing site** (when brand identity is finalized)
- **Digital rental agreement + e-signature** (requires lawyer engagement for Thai short-term-rental template)
- Agoda / Expedia channels
- Full ops workflow (maintenance tickets, inventory, staff payroll)
- General CRM / guest messaging (beyond transactional email + WhatsApp links)
- Automated TM30 online submission (if Thai immigration offers an API — currently manual)
- Competitor rate scraping
- Per-villa microsites

---

## Critical Files / Locations (once QloApps fork exists)

> These paths become real after Phase 0. Listing them now so the plan is executable later.

- `modules/qlo_pricing_engine/` — new module, admin UI for seasons / occupancy rules / special dates / min-stay; YAML import compatible with user's existing `config.yaml`
- `modules/qlo_extras/` — new module, extra-services catalog + admin
- `modules/qlo_xendit/` — new payment module (copy structure from `modules/qlo_stripe/`)
- `modules/qlo_reports_ex/` — new module, Occupancy / ADR / RevPAR + FlowAccount XLSX export
- `modules/qlo_housekeeping/` — new module, housekeeper management + cleaning status admin
- `modules/qlo_checkin/` — new module, guest check-in workflow (TM30, agreement, codes)
- `modules/qlo_overbooking_guard/` — new module, booking holds + pre-commit recheck + alert UI
- `modules/qlo_brands/` — new module, multi-brand configuration + per-brand theming
- `modules/qlo_smart_locks/` — new module, lock abstraction + Tuya adapter + TTlock adapter + manual-code fallback
- `modules/qlo_marketing/` — new module, Mailchimp webhook, review-link dashboard, WhatsApp contact config
- (QloApps CM addon) — installed as-is, configured to link Airbnb + Booking.com accounts for both brands
- `apps/booking-widget/` — Next.js embeddable booking widget + checkout, deployed to Netlify, embedded into `casadeyim.com`
- `apps/housekeeper-pwa/` — Next.js mobile-first PWA
- `apps/guest-checkin/` — refactor of existing `Document/VS-code/Guest booking framework/guest-services.html` into a multi-villa Next.js PWA; keep Netlify functions pattern (`create-payment.js`, `get-booking.js` repurposed to call QloApps API)
- `workers/pricing-engine/` — Node.js cron, standalone; port of `rate_manager/config.yaml` logic
- `scripts/ezee-importer/` — one-off TypeScript CLI

---

## Verification

End-to-end acceptance tests for v1:

1. **Direct booking happy path:** guest picks dates on the public site → Stripe authorizes card → booking appears in admin and on the iCal export feed within 1 min → Airbnb's calendar reflects the block within 15 min (their next iCal poll).
2. **OTA → our system path:** make a test Airbnb booking → within 15 min our iCal worker marks those dates blocked in QloApps → the public site shows the villa unavailable for those dates.
3. **Payment policy:** booking with standard rate shows "authorized THB 0 now, THB X on [T-10 date]"; booking with non-refundable shows full charge at booking; T-10 cron correctly charges standard bookings.
4. **Pricing engine:** set a peak-season tier with 1.5× multiplier and a -7-day lead-time 1.2× bump → run engine → check computed rate in DB and on the public site matches `base × 1.5 × 1.2`.
5. **OTA task list:** after a pricing run, task list shows the correct new rates per villa per channel; marking a row "done" persists and doesn't re-appear tomorrow for unchanged rates.
6. **Reports:** with historical eZee data imported, Occupancy / ADR / RevPAR for the last 12 months match eZee's own reports within ±1%.
7. **Min stay:** attempt to book 1 night on a date where min-stay rule says 2 nights → booking rejected with clear error.
8. **15-villa load:** pricing engine completes full recompute for 15 villas × 365 days in < 5 min; iCal ingest for 15 villas × 2 channels finishes in < 2 min.
9. **Overbooking protection:**
   - *Booking-hold race:* open two direct checkout sessions for the same villa and dates in two browsers → second session sees "held" and cannot proceed until the first's hold expires or releases.
   - *Pre-commit recheck:* stage an Airbnb iCal fixture with a conflicting block → begin a direct checkout for the same dates → on final payment click, system detects conflict, voids Stripe auth, shows clear error.
   - *Collision alert:* manually create an overlapping OTA booking via test iCal feed → ingest worker writes `overbooking_alert` row and the owner receives an email + push notification within 1 min.
   - *Arrival buffer:* attempt a same-day direct booking on a villa without manual override → rejected with "contact host for same-day bookings."
10. **Housekeeper dashboard:**
    - Housekeeper logs in on phone with 4-digit PIN → sees only today's check-ins, check-outs, and stayovers for properties they're assigned to.
    - Marking a villa "cleaning → clean" updates state in DB and shows the new state to admin within 5 s.
    - Admin sees a matrix: villas × next 7 days, color-coded by cleaning state and arrival/departure events.

---

## Risks & Open Items

- **QloApps UI ceiling:** if the native back office still feels "a mess" after theming, we've only solved half the problem. Mitigation: Phase 0 spike, hard decision gate.
- **Xendit custom module:** no off-the-shelf QloApps Xendit module exists; budget ~1 week of module development for this.
- **eZee data quality:** CSV exports may have inconsistent date formats / encoding; budget time to clean.
- **OTA rate parity:** as long as the direct site is priced the same as OTAs, Airbnb and Booking.com are happy. If the user ever wants to underprice direct, that's a separate legal/policy decision.
- **Thai compliance:** PDPA (Thai data protection) applies to guest data; the QloApps default consent flow will need a quick review.
- **Overbooking residual risk:** simultaneous bookings on two different OTAs within the 3-min poll window remain possible. Mitigations above narrow it substantially; owner-alert flow is the accepted manual resolution path.

---

## Guest Messaging Capability (Gap)

**QloApps does NOT have a unified guest-messaging inbox.** Channel Manager syncs rates/inventory/bookings, but admin still needs to log into **airbnb.com** and **booking.com** extranets to reply to guest messages.

**Impact:** this matches the user's current workflow (the 2026 plan's daily task schedule says *"Reply to all OTA messages + direct inquiries via email or Instagram DM"*, implying native-extranet messaging today). Not a regression — parity with eZee, which also lacks a unified inbox on the tier the user runs.

**Mitigation for v1:**
- Keep native OTA extranet messaging workflow
- Surface a **"messages check" reminder** on the admin dashboard home (links out to both extranets + WhatsApp)
- Direct-booking guests and in-stay guests use **WhatsApp** (+66635614563) which the owner already uses

**Phase 4 / deferred:** evaluate third-party unified inbox tools (Hostify, Hospitable, Hostaway have this as a paid layer at $10–30/month per property) if the volume at 15 villas makes extranet-hopping painful.

---

## QloApps Plugin Evaluation (webkul.com/Qloapps.html)

Buy-vs-build decisions based on publicly listed modules. All prices one-time.

### Recommend buying for v1 (~$273 total one-time)

| Plugin | Price | Why buy |
|---|---|---|
| **QloApps Front Desk System** | **$125** | Upgrades native back-office UI for daily operations (check-in / check-out / room status). Directly addresses the "eZee UI is a mess" pain. Buy before writing custom admin modules — spec our custom work around what this already covers. |
| **QloApps Tours and Packages** | **$99** | Potentially replaces our `qlo_extras` module for tour/transfer/experience add-ons. Phase 0 evaluation: if it supports our 6+ package types + Xendit payment, we drop the custom module and save 1–2 weeks of dev. If it's too rigid, we fall back to custom. |
| **QloApps Abandoned Cart Reminder** | **$49** | Direct-conversion win for the 2026 plan. Cheap, plug-and-play — no need to build. |

**Myallocator Connector ($49) — REJECTED.** User flagged Myallocator as not meaningfully updated in ~8 years. Removed as a CM alternative. Fallback path if QloApps native CM proves unreliable = custom iCal poller (3-min intervals), not Myallocator.

**Webkul store freshness caveat:** WebFetch on 2026-04-24 of `store.webkul.com/Qloapps.html` found no per-module "last updated" dates or changelogs exposed on listing pages. Copyright footer shows 2010–2026 (store operational). **Phase 0 mitigation:** before buying any plugin, email Webkul support to request (a) last release date of each module, (b) compatibility with latest QloApps version, (c) support status. If any module has been untouched >18 months, drop it and build custom.

### Recommend skipping (low priority or wrong fit)

| Plugin | Price | Why skip |
|---|---|---|
| QloApps 2Checkout / PayU / Adyen Payment | $75–99 each | Wrong gateways. User uses Stripe (free official module) + Xendit (must build custom — no module exists). |
| QloApps Room Type Q&A | $49 | Nice-to-have for direct-site conversion but not v1-critical. Revisit in Phase 4. |
| QloApps Marketplace | $199 | Multi-vendor marketplace — overkill. We need an internal extras catalog, not a third-party-operator marketplace. |
| QloApps Hotel Virtual Tour | $125 | Marketing nice-to-have. The 2026 plan uses Instagram Reels for this purpose; defer. |
| QloApps Marketplace On Desk Booking | $99 | Aimed at front-desk walk-ins; doesn't match villa-rental workflow. |
| QloApps Cloudkul Platinum Plan | Quote | Managed hosting alternative to DO Singapore. Defer — DO is cheaper and gives more control. |

### Still needs custom build (no plugin available)

- **Xendit payment module** (no Xendit plugin listed — must build; user already has Xendit account and the pattern from existing Netlify `create-payment.js` to port)
- **Dynamic pricing engine** (no match — port of user's `rate_manager/config.yaml` is the cheapest path)
- **Housekeeper dashboard** (no match — lightweight custom build)
- **Self-check-in flow with TM30 + extras + Wishome codes** (refactor of existing `guest-services.html`)
- **FlowAccount XLSX export** (custom report format)
- **Multi-brand configuration** (custom — QloApps out-of-box is single-brand per hotel)
- **Overbooking defense layer** (booking holds + pre-commit recheck — custom)

### Phase 0 plugin-evaluation checklist

Before writing any custom QloApps module, spend 2 days installing the three "recommend buy" plugins and validating they deliver. Specifically:
1. **Front Desk System** — does it give the back-office UX we want, or do we still need heavy theming?
2. **Tours and Packages** — can we configure our 6 package types with Xendit as payment? If yes, scrap `qlo_extras`.
3. **QloApps native Channel Manager** — does it sync cleanly with Airbnb + Booking.com in both directions on a test pair? If not, fall back to custom iCal poller (not Myallocator).

Total plugin budget: ~$273 (Front Desk + Tours & Packages + Abandoned Cart).

---

## Decisions Resolved (from this session)

| Question | Decision |
|---|---|
| QloApps CM pricing | **~$30/mo per QloApps hotel instance** (not per room). Model 2 hotels (Casa de Yim + Brand B) = **~$60/mo or ~$600/yr total**. ROI ~25× vs commission-savings target. **Accept.** |
| Brand structure | **2 brands, 3 tiers, 15 villas total.** Casa de Yim (9 villas) + Brand B (6 villas, name TBC, site build deferred). Tiers: A (฿7k ADR), B (฿10k ADR), C (฿12.5k ADR). |
| Brand B marketing site | **Deferred** — Brand B not yet built. Stub brand record in PMS; marketing site built later when brand identity is finalized. |
| Smart locks | **Migrating from Wishome to Tuya or TTlock.** v1 ships adapters for both (Phase 0 picks primary — likely TTlock for lock-specific API maturity). Abstraction layer preserved for future vendor changes. Manual-code fallback per villa when API unavailable. |
| Rental agreement | Template does not exist today. **Deferred to a later phase** — not required for v1 cutover. (When addressed: engage Thai short-term-rental specialist.) |
| TM30 submission | **Manual** via [tm30.immigration.go.th](https://tm30.immigration.go.th/). v1 generates per-guest submission-ready PDF + pre-filled data. |
| Hosting | **DigitalOcean Singapore** for QloApps core + MySQL. Netlify CDN for all front-ends. |
| 2026 Direct Booking Plan | **Reconciled.** Every tactic in the 2026 plan is reflected in this plan. |
| Rate parity | **Direct priced LOWER than OTAs** (8% below BAR) or match OTA promo + free airport transfer as differentiator. Direct-only rate plan invisible to OTAs. |
| Mailchimp | **Net-new** — Phase 2 includes account creation, list import, and automation setup. |
| Google Hotel Ads | Already connected via eZee Centrix **but never converted bookings**. Cutover re-links the listing to QloApps CM; the 2026 plan's conversion-improvement work (website polish, reviews, response time) drives the actual booking lift. |

## Remaining Open Questions (non-blocking for plan approval)

1. **Brand B name / domain / logo** — needed when the Brand B marketing site is built in a later phase. v1 ships with Brand B stubbed in PMS.
2. **Which 6 villas move to Brand B** — can be decided at onboarding time (villas 5–15 are still coming online through 2026).
3. **Tuya vs TTlock selection** — Phase 0 evaluates both APIs with a test lock. TTlock is the leading candidate (mature lock-specific Open Platform, free developer tier, documented PMS integrations). Tuya is the fallback (broader smart-home reach, but lock features sometimes require device-maker cooperation).

---

## Cost Breakdown (will be extracted to `docs/costs.md` on approval)

All figures in USD unless noted. THB↔USD assumed ~฿34/$1. Estimates, not quotes — confirm each item in Phase 0.

### One-Time Costs

| Item | Low | Likely | High | Notes |
|---|---|---|---|---|
| QloApps Front Desk System plugin | $125 | $125 | $125 | Webkul, one-time |
| QloApps Tours and Packages plugin | $99 | $99 | $99 | Webkul, one-time — only if evaluation passes |
| QloApps Abandoned Cart Reminder plugin | $49 | $49 | $49 | Webkul, one-time |
| QloApps Channel Manager addon setup/license | $0 | ~$50 | $200 | Webkul to confirm; some CM configs have a one-time setup fee separate from subscription |
| Xendit custom payment module dev | $0* | $0* | $0* | Internal dev time (~1 week) |
| Pricing engine (port of `rate_manager/config.yaml`) | $0* | $0* | $0* | Internal dev time (~1 week) |
| Housekeeper PWA | $0* | $0* | $0* | Internal dev time (~1 week) |
| Self-check-in PWA refactor | $0* | $0* | $0* | Internal dev time (~2 weeks) |
| Full Phases 0–3 development | $0* | $0* | $0* | Internal dev time (~11–15 weeks total) |
| Smart-lock hardware (Tuya or TTlock) per villa | $60 | $100 | $150 | Only if replacing Wishome locks across 15 villas: $900–$2,250 total |
| Brand B logo / visual identity | $200 | $500 | $1,500 | Freelance designer, deferred Phase 4 |
| Brand B domain registration | $10 | $15 | $40 | Annual renewal listed separately |
| Thai rental-agreement template (lawyer) | ฿15,000 (~$440) | ฿25,000 (~$735) | ฿40,000 (~$1,175) | Deferred Phase 4 |
| Photography / listing refresh (optional) | $0 | ฿20,000 (~$590) | ฿50,000 (~$1,470) | Per 2026 plan, for direct-site conversion |
| **One-time subtotal (cash, v1 core)** | **~$275** | **~$325** | **~$475** | Excludes hardware, lawyer, branding, photography |
| **One-time subtotal (including optional v1 hardware for 15 villas)** | **~$1,175** | **~$2,575** | **~$4,225** | If migrating all 15 villas to new locks |

*Internal dev time — zero cash cost if user/team builds, otherwise estimate at contractor rate (e.g., $40–80/hr × ~500 hrs = $20,000–$40,000).

### Recurring Costs (Monthly and Annualized)

| Item | Monthly | Yearly | Notes |
|---|---|---|---|
| QloApps Channel Manager subscription | ~$60 | ~$720 | 2 hotel instances × ~$30/mo; confirm with Webkul |
| DigitalOcean Droplet (Singapore, 4–8 GB) | $24–$48 | $288–$576 | Size depends on QloApps+MySQL load; start at 4 GB |
| DigitalOcean Managed Database (optional) | $15 | $180 | Skip in v1 — self-hosted MySQL on droplet until scale forces move |
| DigitalOcean Spaces (backups) | $5 | $60 | 250 GB tier; needed for nightly DB snapshots |
| Netlify (Pro, if free tier exceeded) | $0–$19 | $0–$228 | Free tier likely fine until traffic grows |
| Domain: casadeyim.com | ~$1.25 | ~$15 | Renewal |
| Domain: brand-b.com (when launched) | ~$1.25 | ~$15 | Deferred Phase 4 |
| Mailchimp Essentials (500–2,500 contacts) | $13–$45 | $156–$540 | Scales with list size; 2026 plan targets list growth |
| Stripe transaction fees | var. | var. | ~3.65% + ฿10 per intl card charge — pass-through, not incremental |
| Xendit transaction fees | var. | var. | ~3.5% + ฿10 per charge — pass-through, not incremental |
| TTlock / Tuya developer API | $0 | $0 | Free developer tiers |
| UptimeRobot monitoring | $0 | $0 | Free tier (50 monitors) |
| Sentry error tracking | $0–$26 | $0–$312 | Free tier covers ~5k events/mo |
| Google Workspace (optional, per user) | $6 | $72 | Only if not already on GWS |
| Email deliverability (SendGrid/Postmark, transactional) | $0–$15 | $0–$180 | Free tier on both handles low volume |
| **Recurring subtotal (lean v1)** | **~$105–$145** | **~$1,260–$1,740** | Core: CM + droplet + Mailchimp + domains + backups |
| **Recurring subtotal (with Netlify Pro + Sentry paid)** | **~$150–$190** | **~$1,800–$2,280** | Add-ons for scaled usage |

### Costs Eliminated / Avoided

| Item | Savings | Notes |
|---|---|---|
| eZee Yanolja Absolute subscription | ~$100–$200/mo = ~$1,200–$2,400/yr | Estimated; user to confirm current eZee bill. Eliminated at cutover. |
| eZee Connectivity API fee (if considered) | $70/mo = $840/yr | Never paid — avoided by custom pricing engine + QloApps CM |
| Selenium rate manager maintenance | Internal time | Retired once QloApps pricing engine is live |
| OTA commission on direct-booking share | Target ~฿540,000/yr (~$15,900) | From 2026 plan: 25% direct × ฿3.4M × avg 15% commission saved |

### ROI Sanity Check

- **Year-1 cash cost (lean v1):** ~$325 one-time + ~$1,500/yr recurring ≈ **~$1,825**
- **Year-1 offsets:** eZee subscription eliminated (~$1,200–$2,400) + commission savings (~$15,900 target)
- **Net Year-1 cash:** potential **net savings of $15,000+** even in the conservative case, provided the direct-booking target is hit.
- **Dev time is the real cost** — at contractor rates the $20k–$40k build eats ~2 years of offset. At internal/owner time, the cash math is dominated by commission savings from day one of cutover.

### Assumptions & Confirm-in-Phase-0

1. QloApps CM subscription price is $30/mo/hotel at 2 hotels — **confirm with Webkul quote before Phase 0 decision gate**
2. DO Singapore 4GB droplet is sufficient for 2 QloApps hotels + MySQL — load-test during Phase 1
3. eZee current monthly cost — user to pull from eZee invoices to finalize savings figure
4. Number of locks needing replacement vs reusable across Wishome→TTlock transition — user to inventory
5. Whether Brand B launches within Year 1 — drives whether its domain, Mailchimp list, and CM hotel instance start billing in Year 1 or Year 2

---

## Post-Acceptance TODO

On plan approval, before starting Phase 0:

1. **Create project `CLAUDE.md`** in the new repository root documenting: fork commit of QloApps used, monorepo layout (`modules/` for QloApps extensions, `apps/` for Next.js front-ends, `workers/` for cron jobs), existing-asset references (paths to `guest-services.html`, `rate_manager/config.yaml`, `casa-accounting/`), coding conventions, how to run local dev stack, how to run migrations, how to test overbooking scenarios.
2. **Extract `docs/costs.md`** from the "Cost Breakdown" section of this plan — that becomes the living budget doc; update quarterly as real invoices arrive.
3. Initialize the new repo (separate from `tradingview-mcp`), set up CI, commit Phase 0 spike.
4. Review `Casa_de_Yim_Direct_Booking_Plan_2026.docx` with user and align plan with any specifics not captured here.
