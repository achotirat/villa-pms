# Villa PMS — Cost Budget

Living budget doc. Update quarterly as real invoices arrive.

All figures in USD unless noted. **THB↔USD assumed ~฿34/$1.** Estimates, not quotes — confirm each item in Phase 0.

---

## One-Time Costs

| Item | Low | Likely | High | Notes |
|---|---|---|---|---|
| QloApps Front Desk System plugin | $125 | $125 | $125 | Webkul, one-time |
| QloApps Tours and Packages plugin | $99 | $99 | $99 | Webkul, one-time — only if Phase 0 evaluation passes |
| QloApps Abandoned Cart Reminder plugin | $49 | $49 | $49 | Webkul, one-time |
| QloApps Channel Manager addon setup/license | $0 | ~$50 | $200 | Webkul to confirm; some CM configs have a one-time setup fee separate from subscription |
| Xendit custom payment module dev | $0* | $0* | $0* | Internal dev time (~1 week) |
| Pricing engine (port of `rate_manager/config.yaml`) | $0* | $0* | $0* | Internal dev time (~1 week) |
| Housekeeper PWA | $0* | $0* | $0* | Internal dev time (~1 week) |
| Self-check-in PWA refactor | $0* | $0* | $0* | Internal dev time (~2 weeks) |
| Full Phases 0–3 development | $0* | $0* | $0* | Internal dev time (~11–15 weeks total) |
| Smart-lock hardware (Tuya or TTlock) per villa | $60 | $100 | $150 | Only if replacing Wishome locks across 15 villas: **$900–$2,250** total |
| Brand B logo / visual identity | $200 | $500 | $1,500 | Freelance designer, deferred Phase 4 |
| Brand B domain registration | $10 | $15 | $40 | Annual renewal listed separately |
| Thai rental-agreement template (lawyer) | ฿15,000 (~$440) | ฿25,000 (~$735) | ฿40,000 (~$1,175) | Deferred Phase 4 |
| Photography / listing refresh (optional) | $0 | ฿20,000 (~$590) | ฿50,000 (~$1,470) | Per 2026 plan, for direct-site conversion |
| **One-time subtotal (cash, v1 core)** | **~$275** | **~$325** | **~$475** | Excludes hardware, lawyer, branding, photography |
| **One-time subtotal (with optional v1 hardware for 15 villas)** | **~$1,175** | **~$2,575** | **~$4,225** | If migrating all 15 villas to new locks |

> *Internal dev time — zero cash cost if user/team builds. At contractor rate ($40–80/hr × ~500 hrs) the build would run **$20,000–$40,000**.

---

## Recurring Costs (Monthly and Annualized)

| Item | Monthly | Yearly | Notes |
|---|---|---|---|
| QloApps Channel Manager subscription | $30 | $300 | $30/mo per account (all hotels), or $300/yr if paid annually |
| DigitalOcean Droplet (Singapore, 4–8 GB) | $24–$48 | $288–$576 | Size depends on QloApps + MySQL load; start at 4 GB |
| DigitalOcean Managed Database (optional) | $15 | $180 | **Skip in v1** — self-hosted MySQL on droplet until scale forces move |
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
| **Recurring subtotal (lean v1)** | **~$75–$115** | **~$900–$1,380** | Core: CM ($30) + droplet + Mailchimp + domains + backups |
| **Recurring subtotal (with Netlify Pro + Sentry paid)** | **~$120–$160** | **~$1,440–$1,920** | Add-ons for scaled usage |

---

## Costs Eliminated / Avoided

| Item | Savings | Notes |
|---|---|---|
| eZee Absolute (PMS, SUB-59391) | $54/quarter | Quarterly subscription, billed monthly; eliminated at cutover |
| eZee Reservation (Booking Engine, SUB-59389) | $51/quarter | Quarterly subscription, billed monthly; eliminated at cutover |
| eZee Centrix (Channel Manager, SUB-59390) | $36/quarter | Quarterly subscription, billed monthly; eliminated at cutover |
| **eZee total** | **$141/quarter = ~$47/mo = $564/yr** | All 3 subscriptions; next billing 11 May 2026 |
| Selenium rate manager maintenance | Internal time | Retired once QloApps pricing engine is live |
| OTA commission on direct-booking share | Target **~฿540,000/yr (~$15,900)** | From 2026 plan: 25% direct × ฿3.4M × avg 15% commission saved |

---

## ROI Sanity Check

- **Year-1 cash cost (lean v1):** ~$325 one-time + ~$1,500/yr recurring ≈ **~$1,825**
- **Year-1 offsets:** eZee subscriptions eliminated ($564/yr confirmed) + commission savings (~$15,900 target)
- **Net Year-1 cash:** potential **net savings of $15,000+** even in the conservative case, provided the direct-booking target is hit.
- **Dev time is the real cost.** At contractor rates the $20k–$40k build eats ~2 years of offset. At internal/owner time, the cash math is dominated by commission savings from day one of cutover.

---

## Assumptions & Confirm-in-Phase-0

1. QloApps CM subscription: **$30/mo per account** (covers all hotels) or $300/yr if paid annually — confirmed.
2. DO Singapore 4 GB droplet is sufficient for QloApps + MySQL — load-test during Phase 1.
3. **eZee cost confirmed: $141/quarter (~$47/mo, $564/yr) across all 3 subscriptions.** Next billing 11 May 2026; cancel before then to avoid renewal.
4. **Lock replacement count** — inventory which villas keep Wishome vs switch to Tuya/TTlock.
5. **Brand B launch timing** — drives whether its domain, Mailchimp list, and CM hotel instance start billing in Year 1 or Year 2.

---

## Change Log

| Date | Change | Source |
|---|---|---|
| 2026-04-24 | Initial budget extracted from design spec | `/Users/chotiratapiwattanapong/.claude/plans/i-m-using-ezee-yanolja-calm-cherny.md` |
| 2026-04-24 | Updated eZee confirmed costs ($141/quarter = $564/yr); QloApps CM $30/mo or $300/yr per account | eZee billing portal + Webkul |
