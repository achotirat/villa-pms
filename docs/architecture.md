# Villa PMS — Architecture

## System Overview

```
                        ┌─────────────────────────────────────────┐
                        │           GUESTS / DIRECT BOOKINGS       │
                        └────────────────┬────────────────────────┘
                                         │
                              ┌──────────▼──────────┐
                              │   casadeyim.com      │
                              │  (existing site)     │
                              │  <script> embed      │
                              └──────────┬──────────┘
                                         │
                              ┌──────────▼──────────┐
                              │  booking-widget      │
                              │  (Next.js, Netlify)  │
                              └──────────┬──────────┘
                                         │ QloApps API
┌──────────────┐              ┌──────────▼──────────┐              ┌──────────────────┐
│   Airbnb     │◄────────────►│                     │◄────────────►│  guest-checkin   │
│ Booking.com  │  QloApps CM  │   QloApps Core      │  QloApps API │  PWA (Netlify)   │
└──────────────┘              │   (PHP 8, MySQL 8)  │              └──────────────────┘
                              │   DigitalOcean SG   │
┌──────────────┐              │                     │              ┌──────────────────┐
│   Stripe     │◄────────────►│                     │◄────────────►│ housekeeper-pwa  │
│   Xendit     │              └──────────┬──────────┘              │  (Netlify)       │
└──────────────┘                         │                         └──────────────────┘
                                         │
                              ┌──────────▼──────────┐
                              │  pricing-engine      │
                              │  (Node.js cron)      │
                              │  writes rate tables  │
                              └─────────────────────┘
```

## Module Map

### QloApps Core Modules (`qloapps/modules/`)

| Module | Purpose | Build vs Buy |
|---|---|---|
| `qlo_pricing_engine` | Admin UI for pricing rules (seasons, occupancy, specials, min-stay); YAML import | Custom |
| `qlo_xendit` | Xendit payment gateway (auth-at-book, charge-at-T-10, non-refundable variant) | Custom |
| `qlo_reports_ex` | Occupancy / ADR / RevPAR reports + FlowAccount XLSX export | Custom |
| `qlo_housekeeping` | Housekeeper management + cleaning status admin | Custom |
| `qlo_checkin` | Guest check-in: TM30 data capture, agreement, door codes | Custom |
| `qlo_overbooking_guard` | Booking holds + pre-commit recheck + collision alerts | Custom |
| `qlo_brands` | Multi-brand config + per-brand theming | Custom |
| `qlo_smart_locks` | Lock abstraction layer + Tuya adapter + TTlock adapter + manual fallback | Custom |
| `qlo_extras` | Extras catalog (tours, transfers, experiences) — may be replaced by T&P plugin | Custom (or Buy) |
| `qlo_marketing` | Mailchimp webhook, review-link dashboard, WhatsApp config | Custom |
| QloApps Front Desk System | Drag-and-drop calendar, walk-in bookings | Buy ($125) |
| QloApps Tours & Packages | Package creation + dynamic pricing | Buy ($99) — replaces `qlo_extras` if sufficient |
| QloApps Abandoned Cart | Email reminders for abandoned bookings | Buy ($49) |
| QloApps Stripe | Stripe payment gateway | Buy ($75) |
| QloApps Channel Manager | Airbnb + Booking.com sync ($30/mo) | Buy (subscription) |

### Front-End Apps (`apps/`)

| App | Tech | Deployed | Purpose |
|---|---|---|---|
| `booking-widget` | Next.js 15 + React 19 | Netlify | Embeddable `<script>` widget for casadeyim.com |
| `housekeeper-pwa` | Next.js 15 | Netlify | Mobile-first cleaning status + task management |
| `guest-checkin` | Next.js 15 | Netlify | Self-check-in: TM30, agreement, door codes, extras |

### Workers (`workers/`)

| Worker | Tech | Schedule | Purpose |
|---|---|---|---|
| `pricing-engine` | Node.js 22 + TypeScript | Daily cron | Reads YAML rules → writes rates to QloApps rate tables |

### Scripts (`scripts/`)

| Script | Purpose |
|---|---|
| `ezee-importer` | One-off TypeScript CLI: migrate historical bookings + forward bookings from eZee |

---

## Data Flow: Direct Booking

```
Guest → casadeyim.com → booking-widget (Next.js)
  → QloApps API: check availability
  → QloApps API: create booking + hold
  → Stripe/Xendit: authorize card
  → QloApps: confirm booking
  → Email: confirmation (SendGrid/Postmark via Netlify function)
  → Mailchimp: add/tag contact
  [T-10 days] → Xendit cron: charge full amount
```

## Data Flow: OTA Booking

```
Guest → Airbnb / Booking.com
  → QloApps Channel Manager: receive booking
  → qlo_overbooking_guard: pre-commit recheck
  → QloApps: confirm + block dates
  → Collision alert if overlap detected
```

## Data Flow: Pricing Engine

```
Daily cron (pricing-engine worker)
  → Read config.yaml rules (seasons, occupancy, specials, lead-time)
  → Read current occupancy from QloApps
  → Calculate BAR per villa per date
  → Write rates to QloApps rate tables
  → QloApps CM pushes updated rates to Airbnb + Booking.com
```

---

## Infrastructure

| Component | Provider | Spec |
|---|---|---|
| QloApps + MySQL | DigitalOcean Singapore | 4–8 GB droplet |
| Backups | DigitalOcean Spaces | 250 GB, nightly DB snapshot |
| Front-ends | Netlify | CDN, serverless functions |
| DNS | Existing registrar | casadeyim.com |

---

## Brand Structure

| Brand | Hotels in QloApps | Villas | Tiers |
|---|---|---|---|
| Casa de Yim | Hotel instance 1 | 9 (4 live → 9 by end 2026) | A (฿7k), B (฿10k), C (฿12.5k) |
| Brand B (TBC) | Hotel instance 2 | 6 (coming online through 2026) | TBC |

Every customer-facing query filters by `brand_id`. No hard-coded brand strings in templates.

---

## Security Notes

- TM30 passport data: encrypted at rest (MySQL InnoDB encryption or AES column-level). Never logged.
- THB is the base currency. Multi-currency conversion handled by payment gateway only.
- Booking holds expire after 15 minutes if payment not completed.
