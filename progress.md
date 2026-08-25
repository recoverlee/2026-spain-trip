# 2026 Spain Trip App Progress

Last updated: 2026-08-24 (BCN-PMI baggage info card added)

## Project Overview

This repository powers the `2026 Spain Trip` web app connected to GitHub repository `recoverlee/2026-spain-trip`.

The app started as a single-page Firebase-backed checklist and is now being expanded into a mobile-first travel management web app with three tabs:

1. Schedule
2. Expenses
3. Checklist

The app is intentionally still kept mostly inside `index.html` to avoid a large structural migration while the feature set is stabilizing.

## Current Git State

Latest pushed commit on `main`:

- `48b951a feat: add BCN-PMI round trip baggage info card to 항공 checklist section`

Current change set: working tree clean, all features committed and pushed directly to `main`

Workflow note (as of 2026-08-24):

- Development now happens directly on `main` and is pushed immediately, at the user's explicit request. The previously used feature branch `claude/progress-md-review-23dfmg` is no longer the active development branch; it may be behind `main` going forward.
- Because GitHub Pages serves `main`, this means every push to `main` deploys immediately. Verify changes locally before pushing when possible.

Historical note:

- `02f4ce0` was pushed with the placeholder subject `변경 내용 설명`. It is already on `main`, so it is left as is rather than rewriting published history.

## Authorized Users

Only these Google accounts are intended to use the app:

- `jhlee8379@gmail.com` -> 재환
- `sweetylove0116@gmail.com` -> 혜리

The client keeps the local display-name mapping in `USERS`.

Security must not rely on the client only. The new `firestore.rules` file also restricts Firestore access to the same two accounts.

## Firebase Data Model

Existing checklist data remains compatible with the current production Firestore document.

### Checklist

Path:

```text
trip/checklist
```

Document fields:

- `state`: object keyed by checklist item index
- `checkedBy`: object keyed by checklist item index, value is `재환` or `혜리`
- `updatedAt`: Firestore `serverTimestamp()`
- `updatedBy`: display name

Important compatibility note:

- Checklist item count is now `119` (was `118` through `d0c2a4d`).
- Checklist item count was previously preserved to protect existing Firestore checkbox keys. As of `425b7ee`, a new leaf item (`바르셀로나 공항 택시/버스 이용법`, a link-only reference item) was appended at the end of the `🗺️ 8. 이동` section, which is not the end of the whole checklist, so it shifts flattened indexes for all later sections.
- When checklist text is intentionally corrected, keep the flattened item count stable unless a key migration is planned, or accept and document the drift as done here.

Booking info is display-only:

- Accommodation booking details live in the client constant `ACCOMMODATION_BOOKINGS` in `index.html`.
- They are rendered as a card under the matching checklist accommodation heading and are not checkboxes.
- Nothing about them is read from or written to Firestore, so checkbox keys are unaffected.

Section reference-info cards are also display-only:

- `SECTION_INFO_CARDS` (e.g. Barcelona transit ticket prices) render at the bottom of a matching checklist section and are not checkboxes.
- A checklist leaf item can now optionally carry a link: `[text, url]` instead of `[text]`. When present, a `🔗 참고 링크` link renders under the item text and opens in a new tab. This does not add a Firestore field; the link is purely client-side.

Known index drift at `02f4ce0`:

- Removing `Colon Hotel Barcelona` (2 items) and adding 2 items to `Casp 74 Apartments` kept the total at `118`, but shifted the meaning of one flattened index.
- Flattened index `23` changed from `예약 확인` (Colon Hotel Barcelona) to `9/7(월) 체크아웃 준비 확인` (Casp 74 Apartments).
- The other `117` indexes keep their previous labels.
- Any `state` or `checkedBy` value already stored for index `23` now applies to the new item, so that one checkbox should be reviewed in the app.

Known index drift at `425b7ee`:

- Added `바르셀로나 공항 택시/버스 이용법` as flattened index `97`, at the end of the `🗺️ 8. 이동` section.
- Total item count grew from `118` to `119`.
- Every flattened index from `97` onward shifted by `+1` relative to before this change. This affects the remaining items in `🗺️ 8. 이동` after index 97 (none, since it was appended last) and all items in `🩹 9. 개인용품` and `🔐 10. 출국 직전`.
- Any `state` or `checkedBy` values already stored for indexes `97` and above now apply to a different item one position later, so checked items in those two sections should be reviewed and re-checked in the app if needed.

### Schedule

Path:

```text
tripData/schedule/items/{id}
```

Fields:

- `date`: `YYYY-MM-DD`
- `time`: optional `HH:mm`
- `title`
- `place`: optional
- `note`: optional
- `link`: optional
- `order`: number
- `createdAt`: Firestore `serverTimestamp()`
- `createdBy`: display name
- `updatedAt`: Firestore `serverTimestamp()`
- `updatedBy`: display name

Current UI:

- `일정` tab
- date-grouped timeline for `2026-08-28` through `2026-09-08`
- daily accommodation link shown beside dates where a hotel is assigned
- changeover days display as `숙소1 -> 숙소2`
- `+ 일정 추가`
- add/edit/delete forms
- realtime Firestore subscription via `onSnapshot`

### Expenses

Path:

```text
tripData/expenses/items/{id}
```

Fields:

- `amount`: number
- `currency`: `EUR`, `KRW`, or `USD`
- `dateTime`: local datetime string
- `category`: one of `식비`, `교통`, `숙박`, `관광`, `쇼핑`, `통신`, `기타`
- `title`
- `note`: optional
- `createdAt`: Firestore `serverTimestamp()`
- `createdBy`: display name
- `updatedAt`: Firestore `serverTimestamp()`
- `updatedBy`: display name

Current UI:

- `지출` tab
- `+ 지출 추가`
- mobile-friendly quick input form
- add/edit/delete forms
- realtime Firestore subscription via `onSnapshot`
- summary cards for:
  - total expenses
  - today expenses
  - category totals

## Firestore Security Rules

Rules file:

```text
firestore.rules
```

Firebase config file:

```text
firebase.json
```

Current rules intent:

- signed-in users only
- only `jhlee8379@gmail.com` and `sweetylove0116@gmail.com`
- allow `trip/checklist`
- allow `tripData/schedule/items/{id}`
- allow `tripData/expenses/items/{id}`
- deny all other document paths

Note:

- Firebase CLI is not currently available in this local environment, so rules were statically reviewed but not deployed or compiled with Firebase CLI.
- Rules still need to be deployed separately before they protect production Firestore.

## Hosting and Deployment

The static site (`index.html`, `manifest.webmanifest`, `sw.js`, `icons/`) is served by **GitHub Pages** from this repository, not by Firebase Hosting.

- `firebase.json` has no `hosting` key and there is no `.firebaserc` in this repo. Firebase is used only for Auth and Firestore, not for serving the site.
- There is no `.github/workflows/` file in this repo. GitHub Pages is configured directly in the repo's Settings -> Pages, most likely serving the `main` branch.
- A merge to `main` is picked up and published automatically. No separate `firebase deploy` step is needed for site changes.
- Firestore Rules are a separate concern from Pages hosting: pushing `firestore.rules` changes to `main` does **not** deploy them to Firestore. They still require an explicit `firebase deploy --only firestore:rules` (or equivalent) run by someone with the Firebase CLI and project access. See the Firestore Security Rules note above.

## PWA Status

PWA support is already present and should be preserved.

Files:

- `manifest.webmanifest`
- `sw.js`
- `icons/icon-192.png`
- `icons/icon-512.png`

Current service worker behavior:

- document requests use network-first behavior
- static same-origin assets are cached
- current cache name is `spain-trip-pwa-v3` (bumped for login page + booking updates)

When changing app shell behavior, consider bumping the cache version if stale installed-app behavior is likely.

## Completed Work History

### Firebase Auth and Checklist Sync

- Google login flow cleaned up.
- Only the two allowed accounts can use the app.
- Unauthorized Google accounts are signed out before checklist UI is shown.
- Public checklist uses one shared Firestore document.
- `checkedBy` records `재환` or `혜리`.
- `updatedAt` uses `serverTimestamp()`.
- Firebase internal errors are replaced with user-friendly messages.
- Unused `getDoc`, `storageKey`, and related obsolete code removed.

Committed and pushed:

- `1e491e9 fix: improve Firebase auth and checklist sync handling`

### PWA Installation Support

- Added manifest and app icons.
- Added service worker registration.
- Copied provided icon files into `icons/`.

Committed and pushed:

- `bcb4e5e feat: add PWA installation support`

### Date and Accommodation Improvements

- Added weekday notation to visible dates in checklist items.
- Added Google Maps links to accommodation section headings.
- Preserved checklist item order and count.

Committed and pushed:

- `0f219f1 fix: include weekday in first accommodation date`
- `afb82eb feat: add weekdays and accommodation map links`

### PWA Refresh Behavior

- Changed service worker document handling to network-first.
- Bumped cache from `spain-trip-pwa-v1` to `spain-trip-pwa-v2`.

Committed and pushed:

- `75128af fix: refresh PWA content from network`

### Header Flight Summary

- Added top header flight summary for OZ511 and OZ512.
- Simplified route display to a plain arrow.

Committed and pushed:

- `1bb4def feat: add flight summary to header`
- `55b2e71 fix: simplify flight route arrow`

### Travel Management Expansion

Committed and pushed:

- `0f26a09 feat: expand trip app with schedule and expenses`

- Added tabs: `일정`, `지출`, `체크리스트`.
- Moved existing checklist UI into the `체크리스트` tab.
- Added Schedule CRUD with Firestore realtime sync.
- Added Expenses CRUD with Firestore realtime sync.
- Added expense summary UI.
- Added `firestore.rules`.
- Added `firebase.json`.

### Expense Payer Removal

Committed and pushed:

- `ce4828f fix: remove payer from expenses`

- Removed `결제자` from the expense form.
- Stopped writing `payer` on new or edited expense documents.
- Removed payer tag from expense list items.
- Removed `재환 지출` and `혜리 지출` summary cards.
- Kept total, today, and category expense summaries.

### Schedule Accommodation Links

Committed and pushed:

- `46070fe feat: add accommodation links to schedule`

- Added daily accommodation links to the schedule tab date headers.
- Reused the existing Google Maps accommodation URL mapping.
- Mapped multi-night stays to each applicable date.
- Left `2026-09-08` without accommodation because it is the return-arrival day.

### Accommodation Range Fixes

Committed and pushed:

- `02f4ce0 변경 내용 설명`

- Removed `Colon Hotel Barcelona` from checklist and schedule accommodation mapping.
- Extended `Casp 74 Apartments` stay through `9/7(월)` checkout.
- Updated checklist accommodation ranges to include checkout dates.
- Updated schedule headers so handoff days show `숙소1 -> 숙소2`.

### Accommodation Booking Info

Committed and pushed:

- `44e8c91 feat: show accommodation booking info in checklist (#3)`

Implementation details:

- Added `ACCOMMODATION_BOOKINGS`, keyed by the same heading strings used by `ACCOMMODATION_MAP_URLS`.
- Added `createBookingCard()` and rendered a booking card under matching accommodation headings in the checklist tab.
- Card shows reservation number, stay dates and nights, room type, occupancy, rate, guest name, and phone.
- Breakfast status is shown as a colored badge: included, excluded, or unknown.
- Added an optional cancellation note line.
- Entered complete booking details for `Alberg Centre Esplai` (with payment confirmation), `Gran Hotel Sóller` and `Meliá Palma Marina` from the supplied confirmations.
- `Casp 74 Apartments` still has no confirmation, so it renders heading and checkboxes only.
- `Alberg Centre Esplai` breakfast status confirmed as included (조식 포함).
- `Meliá Palma Marina` parking fee added: €29/night (not included in room rate, pre-booking recommended, height limit 1.90m, key pickup at front desk).

### GitHub Pages Hosting Documentation

Committed and pushed:

- `d952876 docs: record GitHub Pages as the site's hosting mechanism`

Clarified in `progress.md` that the static site is served by GitHub Pages, not Firebase Hosting. This documentation aligns with the repository setup where `firebase.json` has no `hosting` key and there is no `.firebaserc`.

### Login Page Implementation

Committed and pushed:

- `940fb63 feat: add login page - show app only to authenticated users`

Implemented a dedicated login page that is shown before authentication. Key changes:
- Added full-screen login page with Google authentication button
- App UI (tabs, checklist, schedule, expenses) is hidden until user logs in
- Smooth transition: after successful login, app content becomes visible
- Unauthorized users (not in USERS list) are automatically signed out
- Loading state displayed during login process
- Gradient background and styled login card for better UX
- Repository remains public; personal information is now protected by authentication, not repository privacy

### Barcelona Public Transit Ticket Info

Committed and pushed:

- `3094b04 feat: add Barcelona public transit ticket info card to 이동 checklist section`

- Added `SECTION_INFO_CARDS`, a generic map keyed by checklist section title (e.g. `🗺️ 8. 이동`) for reference-only info cards shown at the bottom of a checklist section.
- Added `createTransitCard()` to render a reusable ticket list card (name, description, price per row).
- `renderChecklist()` now appends the matching section info card after a section's items, if one exists in `SECTION_INFO_CARDS`.
- Entered Barcelona transit passes: T-casual (€13), T-usual (€22.80), T-familiar (€11.50), T-grup (€91), T-dia (€12), with ride counts, validity, and transfer window for each.
- This card is display-only, same pattern as `ACCOMMODATION_BOOKINGS`; nothing is read from or written to Firestore.

### Airport Taxi/Bus Guide Link

Committed and pushed directly to `main`:

- `425b7ee feat: add Barcelona airport taxi/bus guide link to 이동 checklist`

- Extended checklist leaf items to optionally support a link: `[text, url]` in addition to the existing `[text]` form.
- When `item[1]` is a string, `renderChecklist()` renders a `🔗 참고 링크` link (opens in a new tab) under the item text, reusing the `.item-link` style used elsewhere.
- The link click has `stopPropagation()` so tapping the link does not also toggle the item's checkbox.
- Added a new checklist item `바르셀로나 공항 택시/버스 이용법` at the end of the `🗺️ 8. 이동` section, linking to `https://m.blog.naver.com/eudemonic005/224333827362`.
- This increased the flattened checklist item count from `118` to `119` and shifted indexes `97` and above. See "Known index drift at `425b7ee`" above.

### BCN-PMI Baggage Info Card

Committed and pushed directly to `main`:

- `48b951a feat: add BCN-PMI round trip baggage info card to 항공 checklist section`

- Added `SECTION_BOOKING_STYLE_CARDS`, a second generic map keyed by checklist section title, rendered with `createBookingCard()` (the booking-card look, reused for non-accommodation reference info).
- Generalized `createBookingCard()` to accept an optional `cardTitle` field; falls back to `예약 정보` when absent, so existing `ACCOMMODATION_BOOKINGS` cards are unaffected.
- `renderChecklist()` now also appends a `SECTION_BOOKING_STYLE_CARDS` card after a section's items and after any `SECTION_INFO_CARDS` transit card, if one exists for that section.
- Entered Air Europa BCN ↔ PMI round-trip baggage rules for 3 passengers (성인 2 + 아동 1): personal item (4kg, 40×30×15cm, ×3), cabin bag (10kg, 55×35×25cm, ×3), checked bag not included in the base Economy Lite fare, and 2 extra checked bags already purchased (23kg, 158cm combined dimensions each, one per direction for 재환) with the owned suitcase size noted as within limits (74×47×31cm = 152cm).
- This card is display-only, same pattern as `ACCOMMODATION_BOOKINGS` and the transit ticket card; nothing is read from or written to Firestore.

## Local Verification Results

Latest local verification after adding accommodation booking info:

- JavaScript module syntax check: PASS
- `sw.js` syntax check: PASS
- `manifest.webmanifest` JSON parse: PASS
- `firebase.json` JSON parse: PASS
- `git diff --check`: PASS
- duplicate HTML id check: PASS
- checklist data compatibility check: checklist count increased from `118` to `119` at `425b7ee`, an intentional new item, see index drift note above
- checklist index stability check: WARNING, `1` of `118` flattened indexes changed meaning at `02f4ce0`
- checklist index stability check: WARNING, indexes `97` and above shifted by `+1` at `425b7ee` (new item appended mid-checklist, not at the very end)
- checklist item link rendering check: PASS, `🔗 참고 링크` renders under the new taxi/bus item and opens in a new tab without toggling its checkbox
- baggage info card render check: PASS, card renders at the bottom of `✈️ 1. 항공` using the booking-card style with custom `cardTitle`
- `createBookingCard()` title regression check: PASS, existing `ACCOMMODATION_BOOKINGS` cards still show `예약 정보` (no `cardTitle` set)
- booking key match check: PASS, every `ACCOMMODATION_BOOKINGS` key matches an existing checklist heading
- booking card render check: PASS, `2` cards render at `430px` and `900px` viewports via headless Chromium
- payer removal check: PASS, no `payer` or `결제자` reference remains in `index.html`
- schedule accommodation link check: PASS, daily accommodation mapping is present for hotel nights
- accommodation range fix check: PASS, `Colon Hotel Barcelona` is removed and handoff days are mapped
- service worker cache name check: PASS, `spain-trip-pwa-v3`
- Firebase init code presence: PASS
- Google Auth flow code presence: PASS
- checklist Firestore sync code presence: PASS
- Schedule CRUD code presence: PASS
- Expense CRUD code presence: PASS
- Firestore realtime sync code presence: PASS
- Firestore Rules static review: PASS
- Firebase CLI rules compile/deploy: NOT RUN, Firebase CLI not installed
- Live Firestore CRUD write/delete test: NOT RUN, to avoid touching production data during review

## Future Work Notes

Recommended next steps:

1. **Test login page UI** — Verify login page displays before authentication and app shows after login with both desktop and mobile viewports.
2. Verify booking info UI rendering with the two allowed accounts in the production app.
3. Verify checklist index `23` in the live app, since its meaning changed at `02f4ce0`, and indexes `97`+ in `🩹 9. 개인용품` / `🔐 10. 출국 직전` since they shifted at `425b7ee` — re-check any previously checked items in those two sections.
4. Add booking info for `Casp 74 Apartments` once the confirmation is available.
5. Confirm the breakfast policy for `Meliá Palma Marina` and update its badge from `조식 미확인`.
6. Test Schedule and Expense CRUD with the two allowed Google accounts to ensure realtime sync works correctly.
7. Confirm whether seed schedule items should be added automatically or entered manually in the app.
8. **Deploy Firestore Rules after review.** They are still not deployed, so production Firestore is not yet protected by them. Use `firebase deploy --only firestore:rules` with Firebase CLI access.

Potential future improvements:

- split `index.html` into smaller JS/CSS files once features stabilize
- add itinerary seed/import feature
- add expense currency conversion or per-currency grouping policy
- add receipt photo storage only after Firebase Storage rules are designed
- add offline write UX if PWA offline use becomes important

## Maintenance Rule

Whenever future work changes behavior, data structure, Firebase rules, deployment flow, or major UI, update this `progress.md` in the same change set.

At minimum, update:

- Current Git State
- Firebase Data Model, if changed
- Firestore Security Rules, if changed
- PWA Status, if changed
- Completed Work History
- Local Verification Results
- Future Work Notes
