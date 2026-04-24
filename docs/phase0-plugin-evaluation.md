# Phase 0 — Plugin Evaluation Checklist

Spend 2 days installing and testing these plugins before writing any custom module.
Last fetched from Webkul store: 2026-04-24.

---

## Plugins to Buy (~$348 total)

| Plugin | Price | Version | Last Updated | Status |
|---|---|---|---|---|
| Front Desk System | $125 | 4.0.1 | Mar 2026 | ✅ Buy |
| Tours and Packages | $99 | 4.0.0 | Oct 2025 | ✅ Buy |
| Abandoned Cart Reminder | $49 | 1.0.2 | Aug 2025 | ✅ Buy |
| Stripe Payment | $75 | 4.0.0 | Dec 2025 | ✅ Buy |

> Confirm freshness and QloApps 1.7.x compatibility with Webkul support before purchasing.
> Skip any plugin untouched >18 months.

## Plugins to Evaluate (don't buy yet)

| Plugin | Price | Version | Last Updated | Question |
|---|---|---|---|---|
| Revenue Management System | $125 | 4.0.0 | Feb 2026 | Does it cover seasons + occupancy + lead-time multipliers + min/max clamps? If yes, replaces custom `workers/pricing-engine`. |

---

## Evaluation Tests

### 1. Front Desk System
**Goal:** Decide if back-office UX is tolerable without heavy custom theming.

- [ ] Install on local QloApps
- [ ] Add 4 test villas (mirror Casa de Yim A1–A4)
- [ ] Create a walk-in booking via the calendar UI
- [ ] Drag-and-drop a booking to a different date
- [ ] Swap a room assignment
- [ ] Filter bookings by date range and villa
- [ ] **Decision:** Is the UX good enough for a non-English-speaking caretaker? → Yes/No

### 2. Tours and Packages
**Goal:** Decide if it replaces custom `qlo_extras` module (saves ~1–2 weeks dev).

- [ ] Install on local QloApps
- [ ] Create these 6 package types:
  - Romance Package (flowers + dinner + late checkout)
  - Family Splash (pool toys + extra beds)
  - Long Stay (7+ nights, free airport transfer)
  - Early Bird (60-day advance, 10% off)
  - Airport Transfer (standalone add-on)
  - Extra Breakfast (per person per day)
- [ ] Attach a package to a room booking at checkout
- [ ] Verify Xendit can be set as payment method for packages (or Stripe)
- [ ] Test admin CRUD — can caretaker add/edit packages without code?
- [ ] **Decision:** Covers all 6 package types with correct payment flow? → Yes (drop `qlo_extras`) / No (build custom)

### 3. Channel Manager (subscription — $30/mo)
**Goal:** Verify end-to-end sync with Airbnb + Booking.com before committing.

- [ ] Install QloApps CM addon
- [ ] Connect test Airbnb account (use staging/test property if available)
- [ ] Connect test Booking.com account
- [ ] Push a rate update from QloApps → verify it appears on Airbnb
- [ ] Create a test booking on Airbnb → verify it appears in QloApps calendar
- [ ] Block dates in QloApps → verify they're blocked on Booking.com
- [ ] Simulate an OTA booking that conflicts with an existing QloApps booking → verify collision alert fires
- [ ] **Decision:** Round-trip sync works reliably? → Yes (proceed) / No (fall back to custom iCal poller)

### 4. Revenue Management System (evaluate before buying)
**Goal:** Decide if it replaces the custom `workers/pricing-engine` (saves ~1 week dev).

- [ ] Request demo access or trial from Webkul
- [ ] Check if it supports: season date ranges, occupancy-based multipliers, lead-time discounts, special date overrides, per-villa min/max rate clamps
- [ ] Check if it can import rules from a YAML/CSV file (we have existing `config.yaml`)
- [ ] Check if it pushes rates to the CM addon automatically
- [ ] **Decision:** Covers all our pricing rules + YAML import? → Yes (buy $125, drop custom worker) / No (build custom)

### 5. Stripe Payment
**Goal:** Confirm it supports auth-at-book + charge-at-T-10 workflow.

- [ ] Install on local QloApps
- [ ] Take a test booking — card authorized but not charged
- [ ] Trigger a manual capture via admin (simulating T-10 cron)
- [ ] Test refund flow from admin
- [ ] **Decision:** Auth + delayed capture works? → Yes / No (build custom Stripe module)

---

## Decision Gate

After completing all evaluations, answer:

1. **Is QloApps back-office UX tolerable** (with Front Desk System) for a non-English caretaker?
2. **Does QloApps CM sync reliably** end-to-end with Airbnb + Booking.com?

**Both YES → proceed to Phase 1.**
**Either NO → convene with owner. Options: (a) heavy theming spike, (b) pivot to greenfield Next.js + custom iCal CM.**

---

## Notes

- All plugins compatible with QloApps 1.6.x and 1.7.x (confirmed 2026-04-24)
- Webkul support email: support@webkul.com
- QloApps GitHub: https://github.com/webkul/hotelcommerce
