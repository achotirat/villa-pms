# Villa PMS — Claude Instructions

Custom PMS + Booking Engine + Channel Manager for pool villas in Ao Nang, Krabi, Thailand.
Replaces eZee Yanolja. Built on a fork of QloApps + Next.js front-ends + Node.js workers.

**Design spec:** `/Users/chotiratapiwattanapong/.claude/plans/i-m-using-ezee-yanolja-calm-cherny.md` (source of truth until Phase 0 lands)
**Cost budget:** `docs/costs.md`

---

## Monorepo Layout (target — populated during Phase 0)

```
villa-pms/
├── CLAUDE.md                       ← this file
├── docs/
│   ├── costs.md                    ← recurring + one-time cost budget (extracted from plan)
│   ├── architecture.md             ← system diagram + module map
│   └── runbooks/                   ← ops procedures (cutover, overbooking resolution, backups)
├── qloapps/                        ← forked QloApps (PHP/MySQL) — pinned commit
│   └── modules/
│       ├── qlo_pricing_engine/     ← seasons × occupancy × specials × clamp; YAML import
│       ├── qlo_xendit/             ← custom Xendit payment module
│       ├── qlo_reports_ex/         ← Occupancy / ADR / RevPAR + FlowAccount XLSX export
│       ├── qlo_housekeeping/       ← housekeeper management + cleaning status admin
│       ├── qlo_checkin/            ← TM30, agreement, codes
│       ├── qlo_overbooking_guard/  ← booking holds + pre-commit recheck + alerts
│       ├── qlo_brands/             ← multi-brand config + theming
│       ├── qlo_smart_locks/        ← Tuya + TTlock adapters + manual-code fallback
│       ├── qlo_extras/             ← extras catalog (if Tours & Packages plugin not sufficient)
│       └── qlo_marketing/          ← Mailchimp, review-link dashboard, WhatsApp
├── apps/
│   ├── booking-widget/             ← Next.js embeddable booking widget; embedded into casadeyim.com
│   ├── housekeeper-pwa/            ← Next.js mobile-first PWA
│   └── guest-checkin/              ← refactor of guest-services.html → multi-villa PWA
├── workers/
│   └── pricing-engine/             ← Node.js cron; port of rate_manager/config.yaml
├── scripts/
│   └── ezee-importer/              ← one-off TypeScript CLI
└── infra/
    ├── docker-compose.yml          ← local dev (qloapps + mysql + redis)
    └── digitalocean/               ← Terraform for Singapore droplet + Spaces + DNS
```

---

## Existing Assets (preserved & extended, not replaced)

| Asset | Path | Role |
|---|---|---|
| Casa de Yim marketing site | `~/Document/Claude_cowork/PROJECTS/Casa de Yim/Casa de Yim Website/` | Keep; embed booking widget via `<script>` tag |
| Guest services / self-check-in | `~/Document/VS-code/Guest booking framework/guest-services.html` + Netlify functions | Refactor into `apps/guest-checkin/`; swap eZee API → QloApps API |
| Dynamic rate manager | `~/Document/Claude_cowork/PROJECTS/Casa de Yim/casa_de_yim_rate_manager/` | Port `config.yaml` semantics into `qlo_pricing_engine`; retire Selenium script |
| Accounting pipeline | `~/Document/Claude_cowork/PROJECTS/Casa de Yim/casa-accounting/` | Keep; `qlo_reports_ex` produces FlowAccount-compatible XLSX that feeds it |
| Direct Booking Plan 2026 | `~/Document/Claude_cowork/PROJECTS/Casa de Yim/Casa de Yim Website/Casa_de_Yim_Direct_Booking_Plan_2026.docx` | Strategic driver for all v1 features |

---

## Core Business Context

- **Scale:** 4 villas today → 15 villas by end of 2026
- **Brands:** 2 (Casa de Yim with 9 villas, Brand B TBC with 6 villas — Brand B site deferred to Phase 4)
- **Tiers:** A (A1–A4, target ADR ฿7,000) · B (B1–B4, ฿10,000) · C (7 premium units, ฿12,500)
- **Payment policy:** authorize at booking → charge 100% at T-10 days → non-refundable rate variants charge at book
- **Min stay:** 2 nights standard; 3 nights Dec 20–Jan 5 and Songkran
- **Direct rate:** 8% below BAR (or match OTA promo) + free airport transfer for bookings ≥ 5 nights
- **Channels:** Airbnb + Booking.com only (no Agoda/Expedia in v1)
- **Owner location:** Bangkok; visits Krabi 2×/month. Caretaker does not speak English.
- **Owner WhatsApp:** +66635614563

---

## Stack

| Layer | Tech | Notes |
|---|---|---|
| PMS core | QloApps (PHP 8, MySQL 8, Prestashop-based) | Fork pinned at a specific commit (recorded here after Phase 0) |
| Channel Manager | QloApps native CM addon | ~$30/mo × 2 hotels = ~$60/mo. Fallback: custom iCal poller (3-min) |
| Front-end apps | Next.js 15 + React 19 + TypeScript | Deployed to Netlify |
| Cron workers | Node.js 22 + TypeScript | Deployed on same droplet as QloApps |
| Payment gateways | Stripe (official module) + Xendit (custom module) | Both required: cards + Thai QR / PromptPay |
| Smart locks | Tuya Cloud API and/or TTlock Open Platform API | Phase 0 decides primary; abstraction layer supports both |
| Hosting | DigitalOcean Singapore (droplet + Spaces for backups) | Netlify CDN for front-ends |
| Email (transactional) | Netlify functions → SendGrid or Postmark | TBD in Phase 1 |
| Email (marketing) | Mailchimp Essentials | Net-new account; webhook on booking |

---

## Critical Conventions

1. **Upstream QloApps stays upgradable.** All extensions live in `modules/qlo_*` directories. Never patch core QloApps files — override via hooks/overrides instead.
2. **Pricing engine is a worker, not a module.** The engine (`workers/pricing-engine/`) runs on cron and writes directly to QloApps rate tables. Admin UI lives in `modules/qlo_pricing_engine/` as a thin editor for YAML rules.
3. **Booking-widget is embeddable.** It must work as a `<script>` include on `casadeyim.com` without requiring a framework on the host page.
4. **Multi-brand from day one.** Every customer-facing query filters by `brand_id`. No hard-coded "Casa de Yim" strings in templates.
5. **Overbooking defense is non-negotiable.** See `modules/qlo_overbooking_guard/` — booking holds + pre-commit recheck + collision alert + 24h arrival buffer.
6. **TM30 data is sensitive.** Encrypt at rest (MySQL InnoDB encryption or column-level via AES). Passport numbers never logged.
7. **THB is the base currency.** Stripe/Xendit present USD/EUR to guests via gateway-side conversion. Never store multi-currency amounts server-side.
8. **FlowAccount is the accounting target.** `qlo_reports_ex` XLSX export must match the columns the `casa-accounting` scripts expect. Changes to either require coordinated updates.
9. **Commit style:** Conventional Commits. Scope = module name (e.g., `feat(qlo_pricing_engine): add lead-time multiplier`).

---

## How to Run Locally (stub — filled in during Phase 0)

```bash
# Bring up local QloApps + MySQL
docker compose -f infra/docker-compose.yml up -d

# Seed test data
npm run db:seed

# Start booking widget (Next.js dev server)
cd apps/booking-widget && npm run dev

# Start pricing engine worker once
cd workers/pricing-engine && npm run once
```

---

## How to Test Overbooking Scenarios

See `docs/runbooks/overbooking-test.md` (to be written in Phase 1). Acceptance tests 9a–9d in the design spec cover:
- Booking-hold race (two browsers, same dates)
- Pre-commit recheck (staged Airbnb iCal conflict → payment voided)
- Collision alert (overlapping OTA booking → alert within 1 min)
- Arrival buffer (same-day direct booking rejected without override)

---

## Non-Goals (Phase 4 / deferred)

- Brand B marketing site
- Digital rental agreement + e-signature (needs lawyer)
- Agoda / Expedia channels
- Maintenance / inventory / payroll modules
- Unified guest-messaging inbox (admin continues to use Airbnb + Booking.com extranets)
- Automated TM30 online submission
- Competitor rate scraping
- Loyalty program
- Per-villa microsites

---

## Reference

- Design spec: `/Users/chotiratapiwattanapong/.claude/plans/i-m-using-ezee-yanolja-calm-cherny.md`
- 2026 Direct Booking Plan: `~/Document/Claude_cowork/PROJECTS/Casa de Yim/Casa de Yim Website/Casa_de_Yim_Direct_Booking_Plan_2026.docx`
- TM30 portal: https://tm30.immigration.go.th/
- QloApps: https://qloapps.com/
- Webkul QloApps store: https://store.webkul.com/Qloapps.html
