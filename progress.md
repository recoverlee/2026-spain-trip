# 2026 Spain Trip App Progress

Last updated: 2026-08-24 (PWA cache bumped to v4)

## Project Overview

This repository powers the `2026 Spain Trip` web app connected to GitHub repository `recoverlee/2026-spain-trip`.

The app started as a single-page Firebase-backed checklist and is now being expanded into a mobile-first travel management web app with three tabs:

1. Schedule
2. Expenses
3. Checklist

The app is intentionally still kept mostly inside `index.html` to avoid a large structural migration while the feature set is stabilizing.

## Current Git State

Latest pushed commit on `main`:

- `cb5dade chore: bump PWA cache version to v4 to force refresh of schedule tab booking card ordering`

⚠️ **Action needed (still open from `8c36e0d`):** the custom checklist item feature added a new Firestore path (`tripData/checklistCustom/items`) and updated `firestore.rules` to allow it, but rules are still not deployed to production (see Firestore Security Rules section). Until `firebase deploy --only firestore:rules` is run, whether add/delete/check on custom items works depends on whatever rules are currently live on the `spain-trip-3006a` project — if production is still on permissive/test-mode rules it will work; if a stricter rule set without this path is live, custom item writes will fail with a permission error. Deploy the rules to be sure.

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
- `hiddenKeys`: object keyed by checklist item index, value `true` for a static item the user has hidden (soft-delete; see "Static item hide/restore" below)
- `updatedAt`: Firestore `serverTimestamp()`
- `updatedBy`: display name

Important compatibility note:

- Checklist item count is now `119` (was `118` through `d0c2a4d`).
- Checklist item count was previously preserved to protect existing Firestore checkbox keys. As of `425b7ee`, a new leaf item (`바르셀로나 공항 택시/버스 이용법`, a link-only reference item) was appended at the end of the `🗺️ 8. 이동` section, which is not the end of the whole checklist, so it shifts flattened indexes for all later sections.
- When checklist text is intentionally corrected, keep the flattened item count stable unless a key migration is planned, or accept and document the drift as done here.

Booking info is display-only:

- Accommodation booking details live in the client constant `ACCOMMODATION_BOOKINGS` in `index.html`.
- **As of commit `9458e6d`, these cards no longer render in the checklist tab.** They moved to the `일정` (Schedule) tab, rendered under the date matching each booking's `checkin` field (see the `Schedule` section below). The checklist accommodation heading (`h3`) and its Google Maps link are unchanged; only the detailed booking card moved.
- Each `ACCOMMODATION_BOOKINGS` entry now also has `hotelName` (used to build the schedule card's title, `"${hotelName} 예약 정보"`) and `checkin` (`YYYY-MM-DD`, used to place the card on the right date).
- `ACCOMMODATION_BOOKINGS_BY_CHECKIN` is an auto-built lookup (date → booking), derived once from `ACCOMMODATION_BOOKINGS` by its `checkin` field — do not hand-maintain it separately.
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
- accommodation booking card: for a date that matches an `ACCOMMODATION_BOOKINGS` entry's `checkin` field, the full booking card (breakfast badge, rows, note) renders right under that date's header — moved here from the checklist tab in commit `9458e6d`. Display-only, driven by `ACCOMMODATION_BOOKINGS_BY_CHECKIN`.
- `SCHEDULE_DAY_NOTES` (client constant, display-only): keyed by `YYYY-MM-DD`, each value is an **array** of reference cards rendered under the booking card (if any), above that date's Firestore-backed items. Currently used for `2026-08-28`: the airport departure plan, the airport-to-hotel transit card, and the hotel review link. Not read from or written to Firestore.
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

### Custom Checklist Items

Path:

```text
tripData/checklistCustom/items/{id}
```

Fields:

- `category`: string, must match one of the static checklist section titles exactly (e.g. `🩹 9. 개인용품`), so the item renders in the right section
- `text`: item label
- `checked`: boolean
- `checkedBy`: display name or `null`
- `createdAt`: Firestore `serverTimestamp()`
- `createdBy`: display name
- `updatedAt`: Firestore `serverTimestamp()`
- `updatedBy`: display name

Current UI:

- `체크리스트` tab, at the bottom of every category section
- custom items render after the static items and any info cards for that section, each with a checkbox and a `추가됨` badge
- a `삭제` button on each custom item, and an inline `+ 추가` form at the bottom of each section — both **only visible while edit mode is on** (see "Edit mode" below)
- realtime Firestore subscription via `onSnapshot`, same pattern as Schedule/Expenses

This is a separate collection from `trip/checklist` (the static item state) on purpose: static checklist items are identified by a flattened numeric index that must stay stable for existing checkbox data (see the index drift notes above), so they cannot safely support arbitrary insert/delete. Custom items instead get their own Firestore document ID, so they can be freely added and removed without disturbing the static items' indexes.

`전체 완료` / `전체 미완료` (checkAll/uncheckAll) apply to both static (non-hidden) and custom items.

### Edit Mode and Static Item Hide/Restore

The checklist tab has an `✏️ 항목 편집` / `✅ 편집 완료` toggle button (`editModeBtn`) next to `전체 완료`/`전체 미완료`. It is off by default so delete/hide/add controls don't clutter the normal view; toggling it on reveals them for every item in every section, and toggling it off (or logging out) hides them again.

Static (built-in) checklist items cannot be truly deleted without corrupting other items' flattened indexes (see the index drift notes above), so edit mode instead offers a **숨기기 (hide) / 복원 (restore)** toggle per static item:

- In edit mode, every static item gets a `숨기기` button. Clicking it sets `hiddenKeys[key] = true` on `trip/checklist` (merged write) and re-renders.
- A hidden item is skipped entirely (not rendered, not counted toward `totalItems`) whenever edit mode is off — it looks deleted to a normal user.
- While edit mode is on, hidden items are shown again but styled distinctly (dimmed, strikethrough text, checkbox disabled) with a `복원` button instead of `숨기기`, so a hide can be undone.
- Hiding/restoring never changes the flattened index (`key`) of any item, so no other item's `state`/`checkedBy` is ever affected.

Implementation notes:

- `totalStaticSlots` counts every static leaf item regardless of hidden state (i.e. the true flattened count, currently `119`); `totalStaticItems`/`totalItems` count only currently-visible (non-hidden) items and drive the progress bar.
- `checkAll()` iterates `0..totalStaticSlots-1` and skips any key present in `hiddenKeys`, so hidden items are never force-checked and the presence of hidden items earlier in the list no longer shifts which keys get checked.
- `uncheckAll()` is unaffected (it clears `state`/`checkedBy` to `{}` entirely).

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
- allow `tripData/checklistCustom/items/{id}` (added in commit `8c36e0d`, for the new custom checklist item feature)
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
- current cache name is `spain-trip-pwa-v4` (bumped again to force-refresh the schedule tab's booking card ordering)

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

### Schedule Tab Day Note Card (8/28 Airport Departure Plan)

Committed and pushed directly to `main`:

- `fbf9bd2 feat: add 8/28 airport departure plan card to schedule tab`

- Added `SCHEDULE_DAY_NOTES`, a client constant keyed by `YYYY-MM-DD`, for a display-only reference card shown under a date's header in the `일정` tab, above that date's Firestore-backed schedule items.
- `renderSchedule()` now builds this card as an inline HTML string (reusing the `.booking-card` / `.booking-rows` / `.booking-note` classes already used elsewhere) and inserts it between the date header and the item timeline, when an entry exists for that date.
- Entered the `2026-08-28` car departure plan from Suwon: 06:30~06:40 departure, 07:50~08:10 parking arrival, 08:10~09:00 terminal/check-in, 09:00~10:00 immigration/duty-free, 10:00~11:00 meal/shopping/gate, 11:50 departure.
- Noted that Terminal 2 parking bus boarding is easiest from zones A, B, E, F, and that the early 06:30 departure accounts for the 10-night Spain trip's extra baggage (bicycle-related gear).
- This card is display-only and unrelated to the Firestore-backed `tripData/schedule/items` collection; it does not use `addDoc`/`updateDoc` and cannot be edited from the UI. Future per-date notes can be added the same way by adding a new date key to `SCHEDULE_DAY_NOTES`.
- `SCHEDULE_DAY_NOTES` entries now optionally support a `link` field. When present, a `🔗 참고 링크` link (opens in a new tab, reusing `.item-link`) renders under the note text. Added `https://m.blog.naver.com/taetae-dairy/224350066891` as the reference link for the `2026-08-28` card (commit `5ff6b9b`).

### Page Title Simplification

Committed and pushed directly to `main`:

- `ca3fb38 fix: simplify page title to remove '출국 전 최종 체크리스트' text`

- Changed the browser `<title>` and the header `<h1>` from `🇪🇸 2026 스페인 여행 출국 전 최종 체크리스트` to `🇪🇸 2026 스페인 여행`.
- The login page's own `<h1>2026 스페인 여행</h1>` was already this shorter form and was not changed.
- No functional or data changes; purely a display text update.

### Custom Checklist Items (Add/Delete per Category)

Committed and pushed directly to `main`:

- `8c36e0d feat: add per-category custom checklist items (add/delete)`

- Added a new Firestore collection `tripData/checklistCustom/items`, separate from `trip/checklist` (see Firebase Data Model above for why).
- `renderChecklist()` now renders, for every section: static items → any `SECTION_INFO_CARDS` / `SECTION_BOOKING_STYLE_CARDS` info cards → custom items for that category → an inline `+ 추가` add-item form.
- Custom items show a `추가됨` badge and a `삭제` button; deleting asks for confirmation via `confirm()`.
- Added `loadCustomChecklistItems()` (realtime `onSnapshot` subscription, ordered by `createdAt`), `addCustomChecklistItem()`, `toggleCustomChecklistItem()`, and `deleteCustomChecklistItem()`.
- Introduced `totalStaticItems` (separate from `totalItems`) so `checkAll()`/`uncheckAll()` only write flattened numeric keys for the static checklist's actual item count; they additionally batch-update all custom items' `checked` field via `Promise.all`.
- `updateProgress()` now adds checked custom items to the `done` count so the progress bar reflects both static and custom items.
- Updated `firestore.rules` to allow `checklistCustom` alongside `schedule`/`expenses` under the existing `tripData/{section}/items/{itemId}` wildcard rule.
- All add/edit/delete controls are disabled when logged out, same as the rest of the app.

### Airport-to-Hotel Transit Card (8/28 Schedule)

Committed and pushed directly to `main`:

- `205868f feat: add airport-to-hotel transit card for 8/28 schedule; support multiple day notes per date`

- Refactored `SCHEDULE_DAY_NOTES[date]` from a single card object to an **array** of card objects, so a date can show more than one reference card. `renderSchedule()` now `.map()`s over the array instead of rendering a single object.
- Added a second card for `2026-08-28`: "바르셀로나 공항 → Alberg Centre Esplai 이동", covering taxi vs. bus options, the PR1/N17 bus routes, the €2.9 per-person cash fare if card payment isn't available, and a note that the N17 route is somewhat faster than PR1.
- Linked `https://m.blog.naver.com/eudemonic005/224333827362` as the reference link for this card — the same guide already linked from the `바르셀로나 공항 택시/버스 이용법` checklist item, now also reachable from the relevant schedule day.
- This is a backward-compatible data shape change: any future `SCHEDULE_DAY_NOTES` entry must be an array (`[{...}]`), not a bare object, or it will fail to render.

### Checklist Edit Mode (Hide/Restore Static Items, Gated Delete/Add)

Committed and pushed directly to `main`:

- `6f6575f feat: add edit mode to hide/restore static items and manage custom items`

- Added an `editModeBtn` toggle (`✏️ 항목 편집` / `✅ 편집 완료`) to the checklist tab's action row, alongside `전체 완료`/`전체 미완료`. Off by default.
- When edit mode is off (the default), no delete/hide/restore/add-item controls render anywhere in the checklist — addressing user feedback that a permanently visible delete button was distracting.
- When edit mode is on: every static item gets a `숨기기` button, every custom item gets its `삭제` button back, and every section's `+ 추가` form appears.
- Added `trip/checklist.hiddenKeys` (object keyed by flattened index, value `true`) as a soft-delete mechanism for static items — hiding an item never removes it from the `data` array or changes any index, so it cannot corrupt other items' stored `state`/`checkedBy`, unlike a real delete would. See the Firebase Data Model section above for the full behavior.
- Added `setStaticItemHidden(key, hidden)` to toggle `hiddenKeys` with a merged `setDoc`.
- Added `updateEditModeBtn()` to sync the button's label/active style with `editMode`; wired into `setDataControlsEnabled()` (turns edit mode off automatically when logged out) and `clearSharedDataViews()`.
- Introduced `totalStaticSlots` (always counts every static leaf item, hidden or not) separately from `totalStaticItems`/`totalItems` (visible-only, drives the progress bar), and fixed `checkAll()` to loop over `totalStaticSlots` while skipping `hiddenKeys` — the previous `totalStaticItems`-bounded loop would have checked the wrong keys once any item was hidden, since hiding shrinks the visible count without changing which numeric keys are in use.

### Breakfast Hours, Hotel Review Link, Empty-Schedule Message Fix

Committed and pushed directly to `main`:

- `5e7baf8 feat: add breakfast hours and hotel review link; fix empty-schedule message when day notes exist`

- Added a `조식 시간: 07:00 ~ 10:00` row to the `Alberg Centre Esplai` entry in `ACCOMMODATION_BOOKINGS`.
- Added a third `SCHEDULE_DAY_NOTES` card for `2026-08-28`, placed after the airport-to-hotel transit card: "Alberg Centre Esplai 호텔 후기 (참고용)", linking to `https://blog.naver.com/eudemonic005/224348941656`. This card has an empty `rows` array (note + link only), confirming `createBookingCard`-style rendering handles a card with no key/value rows.
- Fixed `renderSchedule()`: a date with no Firestore-backed schedule items but at least one `SCHEDULE_DAY_NOTES` card no longer shows the `등록된 일정이 없습니다` empty-state message, since it read as contradictory next to real reference content on the same date (e.g. `2026-08-28`, which has three day-note cards but zero Firestore schedule items). The empty message still shows for a date with neither Firestore items nor day notes.

### Accommodation Booking Cards Moved to Schedule Tab

Committed and pushed directly to `main`:

- `9458e6d feat: move accommodation booking cards from checklist to schedule tab by check-in date`

- Added `hotelName` and `checkin` (`YYYY-MM-DD`) fields to every `ACCOMMODATION_BOOKINGS` entry.
- Added `ACCOMMODATION_BOOKINGS_BY_CHECKIN`, built once via `Object.values(ACCOMMODATION_BOOKINGS).reduce(...)`, keyed by `checkin` date.
- `renderChecklist()` no longer renders a booking card under the accommodation `h3` heading; the heading text and its Google Maps link are unchanged, only the detailed card was removed.
- `renderSchedule()` now renders that date's booking card (breakfast badge, all rows including the `code`-styled reservation number, and the note) directly under the date header, above any `SCHEDULE_DAY_NOTES` cards and the Firestore-backed item list. The markup is inlined as an HTML string (reusing `.booking-card`/`.booking-rows`/`.booking-badge` styling) since `renderSchedule()` builds strings, unlike the checklist's DOM-based `createBookingCard()`.
- The empty-schedule message condition was extended again to also suppress `등록된 일정이 없습니다` when a date has a check-in booking card, even with no day notes and no Firestore items.
- `Casp 74 Apartments` has no `ACCOMMODATION_BOOKINGS` entry yet (no confirmation received), so its check-in date (`2026-09-03`) currently shows no booking card — same limitation as before this change, just relocated.

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
- schedule day note render check: PASS, `2026-08-28` departure plan card renders under the date header, above the Firestore-backed item list, using existing `.booking-card` styling
- schedule day note isolation check: PASS, `SCHEDULE_DAY_NOTES` card has no edit/delete controls and does not call `addDoc`/`updateDoc`/`deleteDoc`
- booking key match check: PASS, every `ACCOMMODATION_BOOKINGS` key matches an existing checklist heading
- booking card render check: PASS, `2` cards render at `430px` and `900px` viewports via headless Chromium
- payer removal check: PASS, no `payer` or `결제자` reference remains in `index.html`
- schedule accommodation link check: PASS, daily accommodation mapping is present for hotel nights
- accommodation range fix check: PASS, `Colon Hotel Barcelona` is removed and handoff days are mapped
- service worker cache name check: PASS, `spain-trip-pwa-v4`
- schedule tab card order check: PASS, `renderSchedule()` already places `bookingCardHtml` before `dayNoteHtml` in the template, so the check-in booking card renders above all `SCHEDULE_DAY_NOTES` cards (including the hotel review card) for a date with both — no ordering change was needed, cache was bumped in case a stale service worker was showing an older layout
- Firebase init code presence: PASS
- Google Auth flow code presence: PASS
- checklist Firestore sync code presence: PASS
- Schedule CRUD code presence: PASS
- Expense CRUD code presence: PASS
- Firestore realtime sync code presence: PASS
- Firestore Rules static review: PASS
- Firebase CLI rules compile/deploy: NOT RUN, Firebase CLI not installed
- Live Firestore CRUD write/delete test: NOT RUN, to avoid touching production data during review
- Custom checklist item code presence check: PASS, `checklistCustomCollection`, `loadCustomChecklistItems()`, `addCustomChecklistItem()`, `toggleCustomChecklistItem()`, `deleteCustomChecklistItem()` all present
- Custom checklist item live write test: NOT RUN, to avoid touching production data; also blocked until `firestore.rules` is deployed (see Firestore Security Rules section)
- `checkAll()`/`uncheckAll()` regression check: PASS, static item loop now uses `totalStaticItems` instead of `totalItems`, so no phantom numeric keys are written beyond the real static item count
- `SCHEDULE_DAY_NOTES` array refactor regression check: PASS, `2026-08-28` still renders its original departure-plan card plus the new transit card, both using the existing `.booking-card` styling
- schedule day note isolation check (2nd card): PASS, the new transit card also has no edit/delete controls and does not call `addDoc`/`updateDoc`/`deleteDoc`
- edit mode default-hidden check: PASS, no `삭제`/`숨기기`/`복원`/`+ 추가` controls render anywhere in the checklist tab when `editMode` is `false`
- edit mode toggle check: PASS, clicking `editModeBtn` flips `editMode`, re-renders, and shows the controls; clicking again hides them
- static item hide/restore check: PASS, hiding an item removes it from the default view and totals; restoring brings it back with its original `state`/`checkedBy` intact
- `checkAll()` with a hidden item check: PASS, after fixing the `totalStaticSlots` bug, `checkAll()` checks every non-hidden static item by its correct key and does not touch the hidden item's key
- empty `rows` day-note card render check: PASS, the hotel review card (no rows) renders its title/note/link without errors
- empty-schedule message check: PASS, `2026-08-28` (3 day-note cards, 0 Firestore items) no longer shows `등록된 일정이 없습니다`; a date with neither still shows it
- booking card relocation check: PASS, checklist accommodation headings no longer render a booking card; `일정` tab shows the correct booking card under `2026-08-28`, `2026-08-29`, and `2026-08-31` (the three confirmed check-in dates)
- booking card content parity check: PASS, breakfast badge tone/text, all rows (including the `code`-styled reservation number for Gran Hotel Sóller and Meliá Palma Marina), and notes all carried over unchanged to the schedule card
- `ACCOMMODATION_BOOKINGS_BY_CHECKIN` build check: PASS, lookup has exactly 3 entries (`2026-08-28`, `2026-08-29`, `2026-08-31`), matching the 3 confirmed bookings; `Casp 74 Apartments` correctly absent (no `checkin` field, since it has no `ACCOMMODATION_BOOKINGS` entry yet)

## Future Work Notes

Recommended next steps:

1. **Test login page UI** — Verify login page displays before authentication and app shows after login with both desktop and mobile viewports.
2. Verify booking info UI rendering with the two allowed accounts in the production app — note booking cards now live in the `일정` tab under each check-in date, not the checklist tab.
3. Verify checklist index `23` in the live app, since its meaning changed at `02f4ce0`, and indexes `97`+ in `🩹 9. 개인용품` / `🔐 10. 출국 직전` since they shifted at `425b7ee` — re-check any previously checked items in those two sections.
4. Add booking info for `Casp 74 Apartments` once the confirmation is available — remember to include `hotelName` and `checkin: "2026-09-03"` so it also appears on the schedule tab automatically.
5. Confirm the breakfast policy for `Meliá Palma Marina` and update its badge from `조식 미확인`.
6. Test Schedule and Expense CRUD with the two allowed Google accounts to ensure realtime sync works correctly.
7. Confirm whether seed schedule items should be added automatically or entered manually in the app.
8. **Deploy Firestore Rules after review.** They are still not deployed, so production Firestore is not yet protected by them, and the new custom checklist item feature (`tripData/checklistCustom/items`) may not work until this is done. Use `firebase deploy --only firestore:rules` with Firebase CLI access.
9. Test adding, checking, and deleting a custom checklist item with both allowed accounts in the production app; if it fails with a permission error, that confirms rules need deploying (see item 8).
10. Test the checklist edit mode toggle, hiding a static item, and restoring it, in the production app with both allowed accounts, and confirm the progress bar total updates correctly when items are hidden.

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
