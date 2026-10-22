# Advertisement Management Module — BookMyDoctor.mu

## Why this doc exists

The original brief asked for a full enterprise ad-server: video ads, per-device
image-variant generation, IP/device/geo click-fraud analytics, a separate
"Advertiser company" entity with its own billing, Google Ad Manager
integration, an ERD, and a full unit/integration/E2E test suite.

BookMyDoctor.mu is a **single-file, no-build-step, static-hosted app on
Firebase Hosting + Firestore + client-side Storage uploads — there is no
application server.** A chunk of that brief genuinely needs one (real
click-fraud prevention, server-side image resizing, rate-limited APIs,
automated CI test suites). Rather than fake those with client-only code that
would quietly not do what it claims, this doc says plainly what was built,
what was intentionally left out, and why — plus what you'd need to add
(usually: Cloud Functions) to close each gap for real.

**This module extends the existing `adverts` collection and admin UI that was
already built** (Admin → Adverts) rather than introducing a parallel system —
same Firestore collection, same Storage path, same admin page, same
`DoctorCard`/`badge`/`chip`/`modal` visual language as the rest of the app.

---

## What's implemented

### Data model
`adverts/{id}`:
```
{
  title,                              // advertiser/campaign display name, e.g. "Dr. Anil Ramgoolam"
                                      // or "ABC Pharmacy" — free text, not tied to a platform account
  redirectUrl,                       // where a click goes — any URL, including a pasted-in link
                                      // to a doctor's own profile page on this site
  imageUrl,                          // default image, Firebase Storage — fallback for any zone below
  images,                            // { [zoneKey]: url } — optional per-zone overrides, e.g. a square
                                      // crop just for DOCTOR_PROFILE while HOME_HERO uses a wide crop
  altText,                           // accessibility + used as the sponsored-card caption
  zones: string[],                   // one or more of AD_ZONES (see below)
  priority: number,                  // 1–10, used for weighted-random rotation
  targetDevice: "all"|"desktop"|"mobile",
  targetSpeciality: string,          // "" = all specialities
  startDate, endDate,                // ISO date strings
  status: "active"|"inactive",       // manual kill-switch, independent of date range
  impressions: number,               // incremented client-side, viewability-gated
  clicks: number,
  createdAt,

  // Legacy fields from before adverts had a redirectUrl — still read for
  // backward compatibility (adLink()/adTitle() fall back to these), but no
  // longer written by the admin form:
  doctorId, doctorName
}
```

Ads are no longer required to point at a doctor already on the platform —
`redirectUrl` can be any external site (a pharmacy, a lab, an insurer) or a
copy-pasted link to a doctor's own profile page. This is a genuine step
toward the brief's "Advertiser" concept without introducing a whole parallel
identity/auth system for external companies: for now, "who's advertising" is
just a free-text `title`, and billing/contact details for a real external
advertiser would be the next piece to add once BookMyDoctor.mu actually
sells ads to non-doctor businesses.

### Placement zones
A subset of the brief's zones, matched to pages that actually exist in
BookMyDoctor.mu today:

| Zone | Renders on | Component |
|---|---|---|
| `HOME_HERO` | Public homepage | `HeroBannerAd` — full-width, responsive, `object-fit: cover` |
| `HOME_CAROUSEL` | Public homepage | `CarouselAd` — auto-rotates every 5s, manual prev/next, swipe on touch, infinite loop, pause-on-interact |
| `SEARCH_RESULTS` | Doctor search results | `SponsoredCardAd`, woven in after every 6th listing |
| `SIDEBAR` | Search results sidebar | `SidebarAdSlot`, desktop only (auto-hidden under 760px) |
| `DOCTOR_PROFILE` | Public doctor profile | `DoctorProfileSponsor`, below the cabinets card |
| `BOOKING_CONFIRMATION` | After a booking request succeeds | `BookingConfirmationAd` — its own small "Request sent!" modal, shown *after* the booking already succeeded, never blocking or delaying it |
| `MOBILE_STICKY` | Site-wide, mobile only | Fixed bottom bar, close button, 24h `localStorage` dismissal |

Not built: `CATEGORY_PAGE`, `HEALTH_ARTICLES`, `FOOTER` — these pages don't
exist in BookMyDoctor.mu yet. Adding a zone for a page that doesn't exist
would be dead code; add the zone key to `AD_ZONES` and a render call once
that page exists.

`AdvertStrip` (the simple horizontal-scroll strip used on the Patient
Overview dashboard) was left as-is — it's a lighter-weight "Featured for you"
treatment for logged-in patients, not one of the formal ad zones, and now
carries the same impression/click tracking as everything else.

### Rotation
One algorithm, used everywhere: **weighted-priority random** —
`pickWeightedAd()` expands each candidate ad into `priority` (1–10) virtual
tickets and picks one at random, so a priority-10 ad is roughly 10× more
likely to be shown than a priority-1 ad in the same zone. "Sequential" and
pure "random" are both degenerate cases of this (set every ad's priority
equal for pure random).

### Targeting
Implemented, because both are derivable purely from the browser/existing data
with no backend needed:
- **Device** — `targetDevice` filtered via `window.innerWidth` at render time.
- **Speciality** — `targetSpeciality` matched against the doctor/search context.

**Not implemented** (all genuinely need server-side session/analytics
infrastructure to do honestly): location/district, language, new-vs-returning
visitor, logged-in-vs-guest. A real version of these needs either Firebase
Analytics + Remote Config's audience targeting, or a Cloud Function sitting
in front of ad selection that reads request headers/geo and a session store —
neither fits a static-hosted, no-backend app. Faking it with
`localStorage`-only heuristics would be trivially wrong (private browsing,
multiple devices, cleared storage) so it was left out rather than shipped
half-working.

### Impression & click tracking
- **Impression** = the ad element actually scrolled into view
  (`IntersectionObserver`, 50% visible threshold) — not just rendered into the
  DOM, per the "count only when visible" requirement.
- **Click** = the ad's link was clicked.
- Both increment atomically via Firestore's `increment(1)`, and tracking is
  de-duplicated per page view (an ad appearing twice on one page — unlikely,
  but possible — only counts one impression).
- **Firestore rules enforce that a write can *only* bump `impressions` or
  `clicks` by exactly 1** — nothing else on the document is touched by that
  code path, and non-admins can't write any other field. That's a basic
  guard against a naive tampering script, **not real fraud prevention** (no
  IP/session rate-limiting is possible from pure client-side rules). If click
  fraud becomes a real problem, the fix is a Cloud Function endpoint that
  does the increment server-side with actual rate-limiting and bot detection
  — happy to build that when there's a Functions plan (Blaze) to deploy it to.

### Analytics
Admin → Adverts → **Analytics** tab: total impressions, total clicks,
overall CTR, and a per-advert table sorted by clicks with its own CTR
column. This reuses the exact same `stat-card`/table styling as the Payments
and Overview reports elsewhere in the app, rather than introducing a new
charting library — no interactive charts (would mean adding Chart.js/Recharts
for a client-side app with a handful of ads; reasonable to add later once ad
volume justifies it).

### Per-zone images
Admin can now upload a **different image for each placement zone** an advert is assigned to (Admin → Adverts → New/Edit advert → "Per-zone images"), not just one shared image cropped by CSS. Each zone gets its own upload slot with a live preview sized to that exact zone's aspect ratio (e.g. a square 1:1 box for the sponsored card and doctor-profile sponsor, a wide ~3.8:1 box for the hero banner), so what's previewed in the form is what actually shows on the site. A **default image** is still required and used as the fallback for any zone that doesn't get its own upload — so a single-image advert continues to work exactly as before, this is purely additive. Stored as `images: { [zoneKey]: url }` alongside the existing `imageUrl` default; `adImageForZone(ad, zone)` picks the override if one exists, otherwise falls back to the default, and every rendering component (hero, carousel, sponsored card, sidebar, profile sponsor, booking confirmation, sticky banner) now calls it with its own zone key.

This still isn't the same as true server-side multi-resolution generation (below) — it's manual, admin-controlled per-zone cropping, not automatic desktop/tablet/mobile variants of a single upload. But it directly solves the common case: a landscape hero image and a tightly-cropped square headshot are genuinely different assets, and now the admin can supply both without fighting CSS `object-fit` to make one image serve both shapes.

**Hero banner gets three images, not one.** `HOME_HERO` is the only zone with three genuinely different recommended sizes (1920×500 desktop / 1200×400 tablet / 1080×600 mobile), so it's special-cased: instead of a single per-zone override, the form shows three separate upload+preview slots (Desktop/Tablet/Mobile), each falling back independently to the default image until uploaded. Stored as `images.HOME_HERO = { desktop, tablet, mobile }` rather than a plain URL. At render time, `heroImageForDevice(ad)` picks the right one for the current `window.innerWidth` (≤760 mobile, ≤1100 tablet, else desktop), falling through desktop → tablet → mobile → default if a particular bucket was never uploaded. This is a viewport check at render time, not a live-resizing `<picture>`/`srcset` — resizing the browser without reloading won't swap the image, matching how the rest of the app's mobile/desktop checks already work.

### House ad — no placement zone is ever empty
When no real advertiser has an active ad in a given zone, that zone falls back to a **self-promotional BookMyDoctor.mu ad** instead of showing nothing (or, for the mobile sticky banner, instead of not appearing at all) — a "Find a doctor" call-to-action linking to `#/search`. This applies to every zone: hero banner, carousel (homepage *and* patient Overview — the Overview carousel wasn't even zone-filtered before this, it's fixed to match the homepage now), sponsored search cards, sidebar, doctor-profile sponsor, booking confirmation, and mobile sticky banner.

It's `houseAdForZone(zone)` — a plain inline SVG (`houseAdSVGDataUri(ratio)`), not an uploaded/stored asset, so no Storage upload and no render cost. **Generated at the exact aspect ratio of the zone asking for it** (read from `AD_ZONES[...].aspect`), not a binary wide/square choice — a first version used just two shapes and force-cropped the wide 3.84:1 hero variant into the sidebar's ~1.25:1 box, which is what caused the zoomed, cut-off "ctor.mu" look. There are three layout tiers based on how wide the ratio is (`>= 2.2` gets a full headline + tagline + CTA, `1.15–2.2` gets a compact logo + wordmark, `< 1.15` gets a stacked square layout), each with its own font/spacing formulas scaled to the target box so text never overflows or gets cropped away regardless of shape. It behaves exactly like a real ad object everywhere it's used (same `adLink`/`adTitle`/`adImageForZone` calls), except its impressions/clicks are never written to Firestore (`recordAdImpression`/`recordAdClick` both skip writes for `id: "house-ad"`, since there's no real document to increment).

**The container CSS had the same bug independent of the house ad** — `CarouselAd` and `SidebarAdSlot` used a *fixed pixel height* (`height:260px` / `height:160px`) at `width:100%`, so on a narrow phone screen the effective on-screen ratio was nowhere near the zone's declared aspect, force-cropping *any* image (real or house ad) into whatever ratio the viewport happened to produce. Both now use CSS `aspect-ratio` (matching `AD_ZONES`) instead of a fixed height, so the box scales with viewport width while keeping the same ratio everywhere — this was the real fix; the per-ratio SVG generation above just makes the house ad specifically render correctly *within* that now-consistent box.

If you'd rather a given zone stay empty instead of showing the house ad once nothing real is booked into it, remove that zone's `|| houseAdForZone(...)` fallback at its call site (each one lives right next to its `getAdsForZone`/`pickWeightedAd` call).

### Responsive images within a zone
Within a single zone, an advertiser's uploaded image is displayed responsively via CSS
(`object-fit: cover` inside sized containers) rather than generating three
separate desktop/tablet/mobile files server-side. Real multi-resolution
generation needs something to run the resize — either a Cloud Function
triggered on Storage upload (using `sharp`), or the official **Firebase
Extension "Resize Images"**, which is a few minutes to install once the
project's on the Blaze plan and needs zero app-code changes (it writes
resized copies alongside the original in the same Storage bucket). This
wasn't wired in because it adds a paid Firebase tier requirement the rest of
this app doesn't otherwise need.

To help admins choose good source images without that pipeline, the New/Edit
Advert form now shows **recommended dimensions per zone** (updates live as
you pick zones) and a **live dual-shape preview** — the moment an image is
chosen, it's shown cropped both as a wide banner and as a square thumbnail,
so you can see exactly how object-fit: cover will crop it before publishing,
without waiting for the upload to finish (it uses a local object URL first,
then swaps to the hosted one).

### Security (what applies here)
- Upload validation: 5MB max, `image/*` only — same rule pattern as the
  existing doctor-photo and subscription-proof uploads.
- Firestore field-scoped writes for impressions/clicks (above).
- Admin-only create/edit/delete, enforced by Firestore rules, not just UI.
- **Not implemented**: rate-limited APIs (there's no API layer to rate-limit —
  it's direct Firestore access from the browser, governed by security rules
  instead), malicious-file content scanning beyond MIME-type/size checks
  (needs a Cloud Function or a service like Cloud Vision SafeSearch),
  full audit logs of every ad edit (Firestore's own `createdAt`/manual fields
  are the only history kept today — a proper audit log would be a Cloud
  Function writing to a separate `auditLogs` collection on every write).

### Accessibility
Every ad image has an `alt` attribute (from the advertiser's alt-text field,
falling back to the doctor's name). Close/nav buttons on the sticky banner
and carousel have `aria-label`s. Everything is a real `<button>`/`<a>`, so
keyboard focus and screen readers work the same as the rest of the app — no
custom widgets that break native semantics.

### UX guardrails (never breaks booking)
- The booking-confirmation ad only ever appears in its *own* modal, opened
  **after** the appointment request has already been written to Firestore and
  the patient notified — a failure loading the ad is silently swallowed and
  never surfaces an error for something cosmetic.
- No ad auto-plays audio, opens an unexpected popup, or sits on top of a
  primary action button.
- Frequency caps are enforced structurally by where the render calls live in
  the code (one hero, one carousel, sponsored cards only every 6th search
  result, one profile sponsor, one post-booking recommendation, one sticky
  mobile banner) rather than by a separate "frequency capping" config layer.

---

## What was deliberately not built, and why

| Brief item | Why it's out of scope here | What it would take |
|---|---|---|
| Video ads | Needs streaming-friendly hosting/transcoding; images cover the same placements adequately for now | Storage + a `<video>` element + transcoding pipeline (Cloud Function or third-party like Mux) |
| Separate "Advertiser" company entity + billing | Ads now take a free-text title + any redirect URL, so external businesses *can* be advertised — but there's still no login, contact record, or self-serve portal for them | A real `advertisers` collection + its own auth flow, if BookMyDoctor.mu starts selling ads directly to non-doctor businesses rather than admin creating them on their behalf |
| Campaign billing/invoicing | The Payments ledger (Admin → Payments → Subscription payments) already exists for subscriptions; ad billing could reuse it with `source: "advertising"` | Small addition once you decide how ads are priced (see Revenue Module below) |
| Google Ad Manager / third-party ad exchange integration | No traffic volume yet to justify it, and it needs a backend to broker the exchange's bidding APIs | A Cloud Function proxy once you're ready |
| ERD / formal database schema diagram | Firestore is a NoSQL document store — there's no relational schema to diagram in the traditional sense | The collection/field table above *is* the schema; a visual ERD-style diagram of the same information could be generated on request |
| Full click-fraud detection (IP, device fingerprint, velocity checks) | Needs server-side request inspection | Cloud Function-fronted tracking endpoint |
| Automated unit/integration/E2E test suite | This app has no build step, no bundler, no test runner installed — adding Jest/Playwright is a real infrastructure decision, not a drop-in | See "Manual QA checklist" below for what to check by hand today; a real suite is a good next investment once the codebase stabilizes further |
| Revenue packages (Bronze/Silver/Gold/Platinum/Diamond) | No pricing decided yet, and it's a business decision as much as a technical one | Once you tell me the tiers/prices, this slots in exactly like the existing subscription plans (Firestore-backed, admin CRUD) |

---

## Manual QA checklist

Since there's no automated test suite, check these by hand after any change
to the ad engine:

- [ ] Create an advert with 2+ zones, priority 8 — confirm it shows more often than a priority-2 ad sharing a zone (refresh the homepage ~10 times)
- [ ] Set `targetDevice: mobile` — confirm it's absent on desktop width and present under 760px (resize the browser or use dev tools device mode)
- [ ] Set `targetSpeciality` — confirm it only shows in search results filtered to that speciality
- [ ] Confirm impressions only increment once an ad scrolls into view, not immediately on page load (scroll down slowly and watch Admin → Adverts → Analytics)
- [ ] Click an ad — confirm the click count increments and you land on the redirect URL you set (external links open in a new tab)
- [ ] Dismiss the mobile sticky banner — confirm it stays hidden on reload, then reappears after 24h (or after clearing `localStorage.bmd_sticky_ad_hidden_until` to test faster)
- [ ] Complete a real booking — confirm the "Request sent!" modal with the recommendation shows *after* the toast, and that skipping/closing it never blocks the actual booking
- [ ] Deactivate an advert — confirm it stops appearing everywhere within one page reload
- [ ] Try editing an advert's `impressions` field directly in the Firestore console as a non-admin test account — confirm the rules reject anything that isn't a same-collection, +1-only update
