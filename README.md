# BookMyDoctor.mu

A responsive Progressive Web App for booking independent doctors in Mauritius.
Single-file frontend (`public/index.html` + `public/app.js`), Firebase Auth +
Firestore backend, no build step required.

## What's included in this build

- **Public site** — hero landing page with a randomly featured verified doctor's photo, doctor search/filter by speciality, public doctor profile pages with photo + "View on map" cabinet links
- **Patient account** — search & book, live slot generation from a doctor's real schedule, my appointments (upcoming/past, cancel), favourites, profile
- **Doctor account** — profile setup (speciality, fees, languages, bio, **photo upload**…), unlimited consultation cabinets (address, **map link**, working days, hours, lunch break, holidays, max patients/day), a **calendar view** of all appointments with a **per-cabinet filter** (FullCalendar, colour-coded by status, click a day's event for details/actions), an **approve/decline** workflow for incoming booking requests, appointments list with mark-done/cancel, private SOAP consultation notes per appointment, and a **subscription** page (Free Trial / Basic / Professional / Premium)
- **Admin account** — platform KPIs, doctor verification queue (approve/reject), doctor & patient directories, speciality management, and a **subscription management** page (review/approve doctor plan requests)
- **Real-time in-app notifications** — a bell icon with unread badge on every dashboard; patients are notified when a doctor approves/declines/cancels, doctors are notified on new requests and cancellations, both are notified on subscription approval. Appointment-related notifications deep-link to that *specific* appointment, not just the page: a doctor gets the full detail modal opened automatically; a patient gets scrolled to and shown a highlighted row (switching Upcoming/Past tabs first if needed). Notifications sent before this shipped just mark as read with no navigation, since they don't have a route stored. Falls back to a native browser notification when the tab isn't focused and permission has been granted.
- **Today's appointments popup** — whoever's logged in (doctor or patient) gets a small dismissible card in the bottom-right the first time they open their Overview each day, if they have any pending/confirmed appointments today. Clicking a row deep-links to that exact appointment the same way notifications do; the × dismisses it until tomorrow (tracked in `localStorage`, scoped per account so switching users on the same device doesn't hide it for the next person).
- **PWA** — installable, offline app-shell caching, manifest + icons generated from your logo, branded loading screen
- Mobile-responsive top bar, grids, and stat cards (fixed after a phone-view overflow bug)
- Firestore security rules enforcing per-role access (patients can't see other patients' bookings, doctors only see their own, consultation notes stay private to the doctor, notifications are readable only by their recipient) + Storage rules scoping photo uploads to their owning doctor
- Sample data seed script (admin / doctor / patient test accounts)

### How booking approval works now
Booking a slot creates an appointment with status `pending`, not an instant confirmation. The doctor sees it in **Appointments** or **Calendar** with **Approve**/**Decline** buttons; approving notifies the patient and moves it to `confirmed`, declining frees the slot back up and notifies the patient to pick another time. Step 2 of the booking wizard is a real month calendar (weekday headers, full month name, prev/next navigation) rather than a flat list of dates — days the cabinet isn't open, holidays, and past dates are greyed out and unclickable.

### How subscriptions work
The doctor Subscription page now mirrors StyleUp's flow: a status banner (Live / Expiring soon / Expired / Not yet verified), a Listing Status + Subscription Plan card pair with expiry date, and a single **Request Renewal/Upgrade** form (plan, payment method, amount paid, payment reference, optional message, and a required proof-of-payment upload to Firebase Storage) instead of per-plan buttons. Admin reviews requests from **Admin → Subscriptions**, now showing the payment method, message, and a "View proof of payment" link. Approving activates the plan for 30 days and notifies the doctor.

**Renewal reminders:** whenever a doctor opens their Overview with 7 or fewer days left on their plan (or it's already expired), they get an in-app notification nudging them to renew — capped at once per day via a `lastRenewalReminderAt` field so it doesn't spam on every page load.

**Plan model, simplified to two tiers:**
- **Free Trial** — all features unlocked, free for 3 months. Activated automatically on signup (`subscriptionStatus: "active"`, `subscriptionExpiry` set 3 months out) — a doctor doesn't need to submit a request for this one, just get verified by admin.
- **Premium** — all features, Rs 6,000/year. Requested and activated through the renewal flow above.

The plans live in Firestore (`subscriptionPlans/{id}`), not hardcoded — **Admin → Subscriptions → Manage plans** has full **Create / Edit / Delete**, so you can rename, reprice, or add more tiers later without touching code. The first time an admin opens that page, the two defaults are auto-seeded into Firestore; after that it's entirely admin-controlled. Doctors only ever read this collection.

**Subscription report:** Admin → Subscriptions now opens on a report with three tabs — **Paid** (doctors currently on a paid, active plan, sorted by expiry), **Pending** (requests awaiting review), and **Ending soon** (active subscriptions expiring within 30 days, soonest first, red badge inside 7 days).

### Vacation periods
Doctor → Profile has a new "Vacation periods" card — log a date range (with an optional reason) and it blocks bookings across **all** of that doctor's cabinets for those dates, distinct from a cabinet's own `holidays[]` which only blocks that one cabinet. Vacation dates are greyed out in the patient booking calendar the same way non-working days are, and the doctor sees a reminder banner on their Overview while a vacation is currently active.

### Appointment filtering & sorting
Both **Doctor → Appointments** and **Patient → My appointments** now have a filter bar: the doctor filters by patient + date, the patient filters by doctor + date, and both can sort by date (newest/oldest first) or by name. Filtering re-renders just the list, not the whole page, and keeps all the existing row actions (approve/decline/complete/cancel/edit/notes) working exactly as before.

### Doctor: read-only "My Patients"
New **My Patients** page for doctors — every patient they've had at least one appointment with, searchable by name, click through to a read-only profile (contact info, insurance if the patient's provided it, and full appointment history with that doctor specifically). Deliberately built **without any new Firestore read permissions**: it's assembled entirely from data the doctor already legitimately has access to — their own appointment records, which now carry a snapshot of the patient's email/insurance info taken at booking time (the same way `patientName`/`patientPhone` were already denormalized onto appointments). This means contact details reflect the patient's profile *as of their most recent booking*, not live — the modal says so explicitly. A tighter, live-synced version would need either a Cloud Function to maintain a verified doctor-patient relationship record, or loosening the `users` collection's read rule, both of which felt like the wrong trade-off for what's fundamentally a "let the doctor see who they're seeing" feature.

The "Consultation notes & appointment history" list inside that same modal has its own **sort** (newest/oldest) and **filter** (by status) controls, independent of the rest of the app. Each row shows a preview of that visit's private SOAP notes (or "No notes yet") and a "Last updated" date if any exist — clicking a row opens the same notes editor used everywhere else (fully editable, not just read-only, since these are the doctor's own clinical notes), and saving refreshes the list in place so the new preview/timestamp shows immediately without having to close and reopen the modal. Notes now track `notesUpdatedAt` so "last modified" is always visible, and the save button reads "Update notes" instead of "Save notes" once a note already exists.

The consultation notes editor also shows the appointment's current status, and — if it's still **confirmed** — a **"Mark as completed"** button right there, so a doctor writing up notes doesn't need to close the popup and hunt for a separate button elsewhere. It reuses the same "does this patient need a follow-up?" prompt as every other "mark done" action in the app, for consistent behaviour everywhere.

### Doctor-initiated follow-ups & fee adjustments
Clicking **Mark done** on a confirmed appointment (Appointments list, Overview, or the Calendar detail view) no longer closes it silently — it asks **"Does this patient need a follow-up visit?"** with two choices: **Yes** marks the visit completed and drops straight into the follow-up scheduler (cabinet → date → live-generated slot, created as `confirmed` immediately, no approval step, patient notified); **No** just marks it completed. Doctor Overview also shows a **Follow-ups scheduled** stat and an **Upcoming follow-ups** list whenever any exist, so they don't get lost among regular bookings. A scheduled follow-up is a normal appointment from then on — it can be **Marked done** (asking the same follow-up-or-closed question again, so a chain of visits can keep going), **Cancelled**, or, since the doctor booked it in the first place, **Edited** (reschedule cabinet/date/time, or change the reason/fee) without cancelling and starting over. Separately, **Edit fee** lets a doctor adjust the `fee` recorded on any completed visit (e.g. extra procedures billed during the visit) without touching their base consultation fee.

**Fixed a rules bug:** doctor-created follow-ups were being silently rejected by `firestore.rules` — the `appointments` create rule only ever allowed a patient to create their own booking, so a doctor creating one on a patient's behalf had no path through. The rule now also allows a signed-in doctor to create an appointment where they're the `doctorId` on it.

### Payment methods accepted
Doctor → Profile has a new multi-select: Cash, Juice, MCB Juice, Bank Transfer, Debit/Credit Card. Whatever's selected shows as badges on the public profile and in the admin doctor-detail modal, so patients know how to pay at the cabinet before they arrive.

### Revenue reports (doctor + admin), with Excel/PDF export
Doctor gets a new **Reports** page; Admin's **Payments** page got the same treatment. Both share: quick filters (Day / Month / Year / Custom range), a revenue summary, a breakdown table (by cabinet for doctors, by doctor for admin), and **Export → Excel** (via SheetJS) or **Export → PDF** (via jsPDF + autoTable) buttons that download exactly what's currently filtered on screen.

### Admin detail views
Clicking any row in **Admin → All doctors** or **All patients** opens a full detail modal — for a doctor: photo, contact info, bio, qualifications, fee/duration/buffer, ticketing status, subscription plan, payment methods, and their full cabinet list; for a patient: contact info, favourites count, and their recent appointment history with status.

### Admin panel, extended (inspired by styleup.mu's admin)
A few sections were carried over from the admin patterns used on your other platforms, adapted to a booking business instead of a marketplace one:
- **Overview** shows the same shape as StyleUp's dashboard: platform-wide stat cards (total doctors, pending review, awaiting subscription, active listings, customers), a **Bookings: Daily/Monthly/Total** breakdown with revenue and pending/completed/cancelled counts per period, and a **Recent bookings** feed.
- **Service Providers** (renamed from "All doctors") — now a full table like StyleUp's Service Providers screen: search + speciality + status + credit filters, and per-row **View**, **Verify/Unverify**, **Assign Credit**, and **Suspend/Reactivate** actions. A **+ New Provider** button creates a doctor account on someone's behalf (see below).
- **All bookings** — every appointment platform-wide, filterable by status.
- **Payments** — now two tabs: the existing **Revenue report** (appointment fees, filterable, Excel/PDF export) and a new **Subscription payments** ledger matching StyleUp's Payments screen: stat cards (Total recorded / Payments logged / Awaiting payment), an editable "Payment Details (shown on invoices)" panel (bank, account, BIC, Juice/MCB Juice number, my.t money, address, phone), a **+ Record Payment** button for logging payments manually (with an option to activate the plan immediately), and a payment history table with a per-row **Download invoice** (PDF) button.
- **Adverts** — full placement engine now, not just a homepage strip: hero banner + auto-rotating carousel on the homepage, and that same **full-size carousel component reused on the patient Overview dashboard** (swipe + auto-play + dot indicators, same 260px height and layout as the homepage — no separate compact version anymore, since a shrunk-down copy just crops the same image differently and looks off), sponsored cards woven into search results, a desktop sidebar slot, a doctor-profile sponsor block, a post-booking recommendation, and a mobile-only sticky bottom banner. Every one of those zones now falls back to a **self-promotional BookMyDoctor.mu house ad** whenever there's no real advertiser booked into it, instead of showing nothing — a wide or square inline SVG depending on the zone's shape, no upload needed. Each advert has placement zones, a 1–10 priority (weighted-random rotation), optional device/speciality targeting, and tracked impressions (viewability-based) + clicks with a CTR analytics tab in Admin → Adverts. **Per-zone images**: instead of one shared image cropped by CSS everywhere, admin can now upload a different image for each zone an advert is assigned to, with a live preview shaped to that exact zone's aspect ratio — a default image still covers any zone left un-customized. **See [`ADVERTISEMENT_MODULE.md`](./ADVERTISEMENT_MODULE.md) for the full schema, what's implemented, and — just as importantly — what was deliberately left out because this is a static, no-backend app (video ads, real click-fraud prevention, server-side image resizing, a separate advertiser-billing system, and automated tests all need a real backend to do honestly).**
- **Subscriptions** — the report now has three tabs (**Paid** / **Pending** / **Ending soon**, the last sorted soonest-first with a red badge inside 7 days), credit requests show an **admin note to provider** field before approving/rejecting, and **Manage plans** now includes a duration-in-days field so approvals/Assign Credit compute the correct expiry automatically.
- **Admin accounts** — grant or revoke admin access for any existing account by email, so this no longer requires hand-editing Firestore.

**Assign Credit vs Record Payment vs Credit Requests — when to use which:**
- **Credit Requests** (Subscriptions page) is the self-serve path: a doctor submits payment proof, you approve or reject it.
- **Assign Credit** (Service Providers row) instantly activates a plan for a doctor with no payment request needed — for comps, goodwill extensions, or fixing a mistake.
- **Record Payment** (Payments → Subscription payments) logs money received (e.g. over the counter, bank transfer you noticed) into the ledger, with an option to activate the plan in the same step.

**"+ New Provider"** creates a real Firebase Auth account for a doctor from the admin panel without logging the admin out — it briefly opens a second, isolated Firebase app instance to run `createUserWithEmailAndPassword`, then tears it down. You'll need to share the generated temporary password with the doctor yourself (there's no email-invite flow wired in yet).

**Suspend/Reactivate** is separate from Verify/Unverify: verification is about "has an admin checked their MCR registration," suspension is "temporarily hide this listing" (e.g. a complaint under review) — a doctor can be verified but suspended, and the public site checks both.

**Standing Order Plans was intentionally left out** — that's StyleUp's recurring-service billing model (customers auto-charged for repeat beauty appointments), which doesn't map to how independent doctor consultations are billed (pay-at-cabinet, one-off visits). If you later want recurring billing for something like a monthly check-up plan, that would slot in as its own feature rather than reusing this one.

### How the optional ticketing system works
Off by default. A doctor turns it on from **Profile → Patient queue tickets**. Once enabled, every new booking for that doctor is assigned a sequential ticket number (per cabinet, per day) the moment the patient submits their request. Patients never get read access to each other's appointment records — that stays as private as everywhere else in the app — so ticket numbers are handed out via a small shared counter document (`ticketCounters/{doctorId}_{cabinetId}_{date}`) that holds nothing but numbers, not names or appointment details. On the doctor's **Overview**, a "Today's queue" panel shows "Now serving #X of Y" per cabinet with a **Call next patient** button; patients see their own ticket number and how many people are ahead of them on their appointment card, updating each time they reload it.

### Doctor photos & cabinet map links
A doctor uploads their photo from **Profile** — it goes straight to Firebase Storage under `doctorPhotos/{uid}_{timestamp}.ext` and is publicly readable (needed so patients can see it in search results without logging in), but a doctor can only ever write files prefixed with their own uid, enforced by `storage.rules`. Cabinets get an optional "Map link" field — paste a Google Maps share link and a "View on map" button appears on the cabinet card and the public profile. No Google Maps API key needed since it's just a link, not an embedded map.

### Performance: heavy libraries are lazy-loaded
FullCalendar, jsPDF (+autoTable), and SheetJS (Excel export) together add several hundred KB of JS. They used to load in `<head>` on *every* page — including the public homepage and booking flow, for people who'd never touch Calendar or Reports — which was the main cause of slow first loads, especially over mobile networks (this is what fixed the "slow on iOS" report). They're now fetched on first actual use via `loadScriptOnce()` in `app.js`, with a brief "Loading calendar…" / "Preparing PDF…" state while they download, and cached for the rest of the session so it only happens once. Font Awesome and Google Fonts still load upfront since they're used everywhere. The service worker's cache name was bumped (`bmd-shell-v2`) so this reaches users after one reload instead of two.

### Fixed: Favourites page stuck on "Loading…" forever
The real bug was `DAL.listAllDoctors()` — an unfiltered query of the entire `doctors` collection — being called from the *patient-facing* Favourites page. Firestore rules only let a patient read `doctors` documents that are individually `verified: true` (or their own, if they were a doctor); a broad, unconstrained `list` query can't satisfy that rule for every possible document, so Firestore rejects the whole query with `permission-denied`. That rejection wasn't wrapped in a `try/catch`, so it threw partway through building the page — after the "Loading…" spinner was shown, but before the real content ever replaced it. Fixed two ways: Favourites now fetches each favourited doctor **individually** (`DAL.getDoctor(id)`, one read per doctor — allowed, since a single-document read only needs to satisfy the rule for that one document), and the router now wraps every Patient/Doctor/Admin page render in a try/catch that shows a "Something went wrong — Retry" screen instead of leaving a spinner stuck forever, so this entire class of bug can't hard-hang the app again even if something else similar slips through later.

### Not included yet (natural next iterations)
This spec covers a very large surface area (live payment gateway integration, SMS/WhatsApp/push notification delivery, live chat, patient queue display, reviews, drag-and-drop calendar rescheduling, multi-language switch, family member profiles). The architecture below is built to extend into all of these — happy to add them one at a time, the same way the rest of your projects have grown.

## 1. Create your Firebase project

1. Go to [console.firebase.google.com](https://console.firebase.google.com) → **Add project**
2. Enable **Authentication → Sign-in method → Email/Password**
3. Enable **Firestore Database** (start in production mode — the rules file below handles security)
4. Enable **Storage** (Build → Storage → Get started) — this is what stores doctor profile photos
5. Project settings → General → "Your apps" → Add a **Web app** → copy the config object

## 2. Configure the app

`public/app.js` already has `projectId`, `authDomain`, and `storageBucket` filled in for the **bookmydoctormu** Firebase project (also set as the default alias in `.firebaserc`). Open the `firebaseConfig` object near the top of `app.js` and fill in the three remaining placeholders — `apiKey`, `messagingSenderId`, `appId` — from Firebase console → Project settings → General → Your apps → SDK setup and configuration. This config is safe to expose publicly — it does not grant access by itself, `firestore.rules` does.

## 3. Deploy

**Option A — npm (Mac/Linux/Windows with Node.js already installed):**
```bash
npm install -g firebase-tools
firebase login
firebase use --add                # pick your project, alias it "default" (already set to bookmydoctormu in .firebaserc)
firebase deploy --only hosting,firestore:rules,firestore:indexes,storage
```

**Option B — Windows, no Node.js required:**
Download the standalone CLI from `https://firebase.tools/bin/win/instant/latest` (this is `firebase-tools-instant-win.exe`). Run it once from the `bookmydoctor` project folder — it drops a `firebase.exe` you can then use exactly like the `firebase` command above:
```
firebase login
firebase use --add
firebase deploy --only hosting,firestore:rules,firestore:indexes,storage
```
Note: this standalone CLI only gives you the `firebase` command itself — it doesn't install Node.js/npm system-wide, so you'll still need Node.js separately for the seed script in step 4.

Your app will be live at `https://<your-project-id>.web.app`.

## 4. Seed sample accounts (optional but recommended for testing)

```bash
cd seed
npm install
# Download a service account key from
# Firebase console → Project settings → Service accounts → Generate new private key
# Save it as seed/serviceAccountKey.json (already excluded by .gitignore at the project root — never commit it)
node seed.js
```

This creates:

| Role | Email | Password |
|---|---|---|
| Admin | superadmin@bookmydoctor.mu | 123456 |
| Doctor | doctor@bookmydoctor.mu | 123456 |
| Patient | patient@bookmydoctor.mu | 123456 |

The seeded doctor comes pre-verified with two working cabinets (Port Louis & Curepipe) so you can test the full booking flow immediately.

## 5. Local testing

Any static file server works, e.g.:

```bash
cd public
npx serve .
```

Note: Firebase Auth requires the app to run on `http://localhost` or a domain you've added under Authentication → Settings → Authorized domains — opening `index.html` directly via `file://` will not work.

## Project structure

```
bookmydoctor/
├── firebase.json          # Hosting + Firestore + Storage config
├── firestore.rules        # Role-based security rules
├── firestore.indexes.json # Composite indexes
├── storage.rules           # Doctor photo upload rules
├── .firebaserc            # Project alias (bookmydoctormu)
├── .gitignore              # Excludes serviceAccountKey.json, node_modules, etc.
├── public/
│   ├── index.html          # Shell + design tokens (CSS)
│   ├── app.js               # Entire application logic
│   ├── manifest.json         # PWA manifest
│   ├── sw.js                  # Service worker (offline app-shell cache)
│   └── assets/
│       ├── logo-full.png        # Full lock-up logo
│       └── icons/                # Generated PWA icon set
└── seed/
    ├── seed.js               # Sample account + speciality seeder
    └── package.json
```

## Data model (Firestore)

- `users/{uid}` — `{ fullName, email, phone, role: "patient"|"doctor"|"admin", favourites: [], insuranceCompany, insurancePolicyNumber }` — the two insurance fields are patient-only and optional
- `doctors/{uid}` — profile fields + `verified: boolean`, `suspended: boolean`, `subscriptionPlan: "trial"|"premium"`, `subscriptionStatus`, `subscriptionExpiry`, `paymentMethods[]`, `juicePhoneNumber`, `lastRenewalReminderAt`, `vacations: [{id, start, end, reason}]`
  - `doctors/{uid}/cabinets/{cabinetId}` — `{ name, address, workingDays[], hours{start,end,lunchStart,lunchEnd}, maxPatients, holidays[] }`
- `appointments/{id}` — `{ doctorId, patientId, cabinetId, date, time, status: "pending"|"confirmed"|"completed"|"cancelled"|"declined", fee, reason, privateNotes, notesUpdatedAt, patientName, patientPhone, patientEmail, patientInsuranceCompany, patientInsurancePolicyNumber }` — the last three are a snapshot of the patient's profile at booking time, denormalized so the doctor can read them via their own appointment records (see "Doctor: read-only My Patients" below)
- `specialities/{name}` — `{ name }`
- `notifications/{id}` — `{ userId, title, message, relatedApptId, route, read: boolean, createdAt }` — real-time via `onSnapshot`, readable only by `userId`; `route` is where clicking the notification navigates to
- `subscriptionPlans/{id}` — `{ name, price, period, durationDays, features[] }` — admin-managed, publicly readable
- `subscriptionRequests/{id}` — `{ doctorId, doctorName, plan, price, paymentMethod, paymentRef, message, proofUrl, adminNote, status: "pending"|"approved"|"rejected", createdAt }`
- `ticketCounters/{doctorId}_{cabinetId}_{date}` — `{ doctorId, cabinetId, date, count, nowServing }` — no patient data, just numbers
- `payments/{id}` — `{ doctorId, doctorName, amount, method, plan, reference, recordedBy, recordedByName, source: "manual"|"credit_request", createdAt }`
- `settings/paymentDetails` — `{ bankName, accountName, accountNumber, bic, juiceNumber, mtMoneyNumber, companyAddress, contactPhone }` — shown on generated invoices
- `adverts/{id}` — `{ doctorId, doctorName, imageUrl, altText, zones[], priority, targetDevice, targetSpeciality, startDate, endDate, status: "active"|"inactive", impressions, clicks, createdAt }` — see `ADVERTISEMENT_MODULE.md` for the full placement-engine writeup

Slots are generated on the fly from a cabinet's `workingDays` + `hours` + the doctor's `consultationDuration`/`bufferTime`, minus already-booked times for that cabinet and date (excluding cancelled/declined ones) — there is no separate "slots" collection to keep in sync.

---
Developed by Drivesh JUGOO.
