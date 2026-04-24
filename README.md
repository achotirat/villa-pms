# Villa PMS

Custom Property Management System + Booking Engine + Channel Manager for pool villas in Ao Nang, Krabi, Thailand.

Replaces eZee Yanolja. Built on a fork of [QloApps](https://qloapps.com/) (PHP/MySQL) with Next.js front-ends and Node.js workers.

---

## Brands & Scale

| Brand | Villas | Target ADR |
|---|---|---|
| Casa de Yim | 9 villas (4 live → 9 by end 2026) | ฿7,000–฿12,500 |
| Brand B (TBC) | 6 villas | TBC — Phase 4 |

Channels: Airbnb + Booking.com (v1). Direct bookings via embedded widget on [casadeyim.com](https://casadeyim.com).

---

## Stack

| Layer | Tech |
|---|---|
| PMS core | QloApps fork (PHP 8, MySQL 8) |
| Channel Manager | QloApps CM addon ($30/mo) |
| Front-end apps | Next.js 15 + React 19 + TypeScript → Netlify |
| Workers | Node.js 22 + TypeScript |
| Payments | Stripe + Xendit (Thai QR/PromptPay) |
| Smart locks | Tuya / TTlock |
| Hosting | DigitalOcean Singapore |

---

## Monorepo Layout

```
villa-pms/
├── docs/
│   ├── costs.md                    ← confirmed cost budget
│   ├── architecture.md             ← system diagram (Phase 0)
│   └── runbooks/                   ← ops procedures
├── qloapps/                        ← forked QloApps
│   └── modules/
│       ├── qlo_pricing_engine/     ← seasons × occupancy × specials
│       ├── qlo_xendit/             ← Xendit payment module
│       ├── qlo_checkin/            ← TM30, agreement, door codes
│       ├── qlo_overbooking_guard/  ← booking holds + collision alerts
│       ├── qlo_housekeeping/       ← housekeeper management
│       ├── qlo_smart_locks/        ← Tuya + TTlock adapters
│       ├── qlo_reports_ex/         ← Occupancy/ADR/RevPAR + FlowAccount XLSX
│       └── qlo_marketing/          ← Mailchimp + WhatsApp + review links
├── apps/
│   ├── booking-widget/             ← embeddable Next.js booking widget
│   ├── housekeeper-pwa/            ← mobile-first PWA
│   └── guest-checkin/              ← self-check-in PWA
├── workers/
│   └── pricing-engine/             ← Node.js cron rate writer
├── scripts/
│   └── ezee-importer/              ← one-off migration CLI
└── infra/
    ├── docker-compose.yml
    └── digitalocean/               ← Terraform
```

---

## Docs

- [Cost budget](docs/costs.md) — confirmed recurring + one-time costs
- [CLAUDE.md](CLAUDE.md) — full project conventions and context for AI-assisted development
- Design spec: `~/.claude/plans/i-m-using-ezee-yanolja-calm-cherny.md`

---

## Key Conventions

- All extensions in `modules/qlo_*` — never patch QloApps core
- Pricing engine runs as a cron worker, writes to QloApps rate tables
- Every customer-facing query filters by `brand_id`
- TM30 passport data encrypted at rest; never logged
- THB is the base currency
- Commit style: Conventional Commits, scope = module name
