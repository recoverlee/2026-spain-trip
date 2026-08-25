# 2026 Spain Trip App Progress

Last updated: 2026-08-24 (PWA cache v3 for login page)

## Project Overview

This repository powers the `2026 Spain Trip` web app connected to GitHub repository `recoverlee/2026-spain-trip`.

The app started as a single-page Firebase-backed checklist and is now being expanded into a mobile-first travel management web app with three tabs:

1. Schedule
2. Expenses
3. Checklist

The app is intentionally still kept mostly inside `index.html` to avoid a large structural migration while the feature set is stabilizing.

## Current Git State

Latest pushed commits:

- `d952876 docs: record GitHub Pages as the site's hosting mechanism`
- `44e8c91 feat: show accommodation booking info in checklist (#3)`
- `0b3d923 docs: sync progress.md with accommodation range commits (#2)`

Current change set: working tree clean, all features committed and merged to main

The accommodation booking info feature (commit `44e8c91`) added read-only booking info cards to the checklist accommodation headings. The flattened checklist item count and all item labels remain unchanged.

The hosting documentation update (commit `d952876`) clarified that GitHub Pages serves the static site, not Firebase Hosting.

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

- Checklist item count remains `118`.
- Checklist item count is preserved to protect existing Firestore checkbox keys.
- When checklist text is intentionally corrected, keep the flattened item count stable unless a key migration is planned.

Booking info is display-only:

- Accommodation booking details live in the client constant `ACCOMMODATION_BOOKINGS` in `index.html`.
- They are rendered as a card under the matching checklist accommodation heading and are not checkboxes.
- Nothing about them is read from or written to Firestore, so checkbox keys are unaffected.

Known index drift at `02f4ce0`:

- Removing `Colon Hotel Barcelona` (2 items) and adding 2 items to `Casp 74 Apartments` kept the total at `118`, but shifted the meaning of one flattened index.
- Flattened index `23` changed from `예약 확인` (Colon Hotel Barcelona) to `9/7(월) 체크아웃 준비 확인` (Casp 74 Apartments).
- The other `117` indexes keep their previous labels.
- Any `state` or `checkedBy` value already stored for index `23` now applies to the new item, so that one checkbox should be reviewed in the app.

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

## Local Verification Results

Latest local verification after adding accommodation booking info:

- JavaScript module syntax check: PASS
- `sw.js` syntax check: PASS
- `manifest.webmanifest` JSON parse: PASS
- `firebase.json` JSON parse: PASS
- `git diff --check`: PASS
- duplicate HTML id check: PASS
- checklist data compatibility check: PASS, flattened checklist count remains `118`
- checklist index stability check: PASS for the current change set, `0` of `118` labels changed
- checklist index stability check: WARNING, `1` of `118` flattened indexes changed meaning at `02f4ce0`
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
3. Verify checklist index `23` in the live app, since its meaning changed at `02f4ce0`.
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
