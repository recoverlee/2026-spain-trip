# 2026 Spain Trip App Progress

Last updated: 2026-08-31 (총 지출/오늘 지출 summary cards show a KRW estimate next to EUR amounts; cache v22)

## Project Overview

This repository powers the `2026 Spain Trip` web app connected to GitHub repository `recoverlee/2026-spain-trip`.

The app started as a single-page Firebase-backed checklist and is now being expanded into a mobile-first travel management web app with four tabs:

1. Schedule (일정)
2. Expenses (지출)
3. Useful Links (유용한 링크, formerly named 쇼핑/Shopping) — added 2026-08-24, renamed 2026-08-29
4. Checklist (체크리스트)

The app is intentionally still kept mostly inside `index.html` to avoid a large structural migration while the feature set is stabilizing.

## Current Git State

Latest pushed commit on `main`:

- `e5e8219 chore: bump PWA cache version to v22 for expense summary KRW hint`

⚠️ **Action needed (high priority):** the read-only account feature's actual security boundary is in `firestore.rules` (`isEditUser()` vs `isAllowedUser()`), which — like every other Firestore rules change this session — **has not been deployed to production**. Until `firebase deploy --only firestore:rules` is run, `yeonholee1024@gmail.com` may either (a) be unable to read anything if production is on some other restrictive rule set, or (b) worse, actually be able to write if production is still on permissive/test-mode rules. Deploy the rules before relying on the read-only restriction for anything sensitive.

⚠️ **Possible follow-up needed (from 2026-08-24, still open):** the `2026-08-28` departure card's `수원 출발` time changed from `06:30~06:40` to `07:10`, but the downstream rows (`07:50~08:10 인천공항 주차장 도착` etc.) were intentionally left unchanged since only the departure time and a note phrase were explicitly requested. A ~40–70 minute Suwon-to-Incheon-airport drive is tight with the new departure time; if the user wants the rest of the timeline shifted later to match, update `SCHEDULE_DAY_NOTES["2026-08-28"][0].rows` in `index.html` accordingly.

⚠️ **Action needed (still open from `8c36e0d`):** the custom checklist item feature added a new Firestore path (`tripData/checklistCustom/items`) and updated `firestore.rules` to allow it, but rules are still not deployed to production (see Firestore Security Rules section). Until `firebase deploy --only firestore:rules` is run, whether add/delete/check on custom items works depends on whatever rules are currently live on the `spain-trip-3006a` project — if production is still on permissive/test-mode rules it will work; if a stricter rule set without this path is live, custom item writes will fail with a permission error. Deploy the rules to be sure.

Current change set: working tree clean, all features committed and pushed directly to `main`

Workflow note (as of 2026-08-24):

- Development now happens directly on `main` and is pushed immediately, at the user's explicit request. The previously used feature branch `claude/progress-md-review-23dfmg` is no longer the active development branch; it may be behind `main` going forward.
- Because GitHub Pages serves `main`, this means every push to `main` deploys immediately. Verify changes locally before pushing when possible.

Historical note:

- `02f4ce0` was pushed with the placeholder subject `변경 내용 설명`. It is already on `main`, so it is left as is rather than rewriting published history.

## Authorized Users

Only these Google accounts are intended to use the app, as of commit `595aa40`:

- `jhlee8379@gmail.com` -> 재환 (edit access)
- `sweetylove0116@gmail.com` -> 혜리 (edit access)
- `yeonholee1024@gmail.com` -> 연호 (**read-only** — can sign in and view every tab, but cannot check/hide/add/edit/delete anything)

The client keeps the local display-name mapping in `USERS`, and the read-only subset in `READ_ONLY_USERS` (a `Set` of emails). The client computes a global `isReadOnly` flag in `onAuthStateChanged`, reset on logout via `clearSharedDataViews()`.

Security must not rely on the client only. `firestore.rules` is the actual enforcement boundary: `isAllowedUser()` (all 3 emails) gates `read`, while a separate `isEditUser()` (only the original 2 emails) gates `write`, on both `trip/checklist` and `tripData/{section}/items/{itemId}`. The client-side `isReadOnly` UI gating (disabled buttons, hidden 수정/삭제 controls, early-return guards in every write function) is purely for UX — a modified/malicious client could still attempt a write, and only the Firestore rules actually stop it. **As with every prior rules change this session, these rules are not yet deployed to production** — see Future Work Notes.

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

- Checklist item count is now `114` (was `113` through `681082d`, `119` through `4a6cf71`, and `118` through `d0c2a4d`).
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

Known index drift at `681082d`:

- Removed the entire `🧳 4. 짐 → BCN 공항에 보관` subsection (4 items: `28인치 캐리어 1개`, `8/29(토)~9/3(목) 공항 보관서비스 위치 확인`, `보관비용 확인`, `9/3(목) BCN 도착 후 수령 방법 확인` — old flattened indexes `38`–`41`), since the plan to store luggage at BCN airport was dropped.
- Also removed `BCN T1 수하물 보관 위치 확인` (old index `90`) and `BCN 도착 후 보관짐 수령` (old index `94`) from `🗺️ 8. 이동`.
- Total item count dropped from `119` to `113` (6 items removed). Every flattened index from the old index `38` onward shifted down by up to `6` positions (the exact shift depends on how many of the 6 removed indexes precede a given item).
- Any `state`/`checkedBy`/`hiddenKeys` values already stored for old indexes `38` and above now apply to a **different, unrelated item** at the same numeric key. Recommend both users review their checked/hidden state across the whole checklist after this update, not just a specific range, since the shift is non-uniform (some items shift by 4, others by 6, depending on position relative to both removed blocks).

Known index drift at `c0905de`:

- `🧳 4. 짐 → 마요르카로 가져갈 짐 → 20인치 캐리어 2개` (index `38`) reworded to `20인치 캐리어 3개`, same index, same meaning (label-only change, no drift for this one).
- Added a new item `추가 대형 수하물 1개` immediately after it, as index `39` (pushing `백팩 3개` and everything after down by one).
- Total item count grew from `113` to `114`. Every flattened index from `39` onward shifted by `+1`.

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
- `SCHEDULE_DAY_NOTES` (client constant, display-only): keyed by `YYYY-MM-DD`, each value is an **array** of reference cards rendered under that date's header, above that date's Firestore-backed items. Not read from or written to Firestore. Currently has entries for `2026-08-28` (2 cards), `2026-08-29` (3 cards: the Mallorca transfer plan, the Air Europa UX6007 boarding pass info, then the Record Go rental car reservation + pickup guide), and `2026-09-04` (2 cards: the 16-stop Barcelona sightseeing route, then Sagrada Familia entry tips).
- **Booking card position within the day-note cards** (added in commit `6dc7416`): an `ACCOMMODATION_BOOKINGS` entry can set an optional `scheduleOrder` (integer) — the number of `SCHEDULE_DAY_NOTES` cards to render before inserting the booking card among them. Default (no `scheduleOrder`, or `0`) puts the booking card first, before all day notes. When there is no `checkinBooking` for a date, `SCHEDULE_DAY_NOTES` cards just render in their array order as before.
  - `Alberg Centre Esplai` sets `scheduleOrder: 2`, so on `2026-08-28` the order is: 인천공항 출발 계획 → 바르셀로나 공항 이동 → **[예약 카드]** (last, since there are only 2 day notes on this date).
  - `Gran Hotel Sóller` sets `scheduleOrder: 2` (added in commit `2dc355e`), so on `2026-08-29` the order is: 마요르카 이동 → Record Go 렌터카 → **[예약 카드]** (last, matching the day's actual timeline: airport transfer → rental car pickup → arrival at the hotel).
- **Booking card reference link** (added in commit `4c05f6c`): an `ACCOMMODATION_BOOKINGS` entry can optionally set a `link` field, rendered as a `🔗 참고 링크` link (opens in a new tab) at the bottom of the booking card, below the note. `Alberg Centre Esplai` uses this for its hotel review blog link — previously this lived in its own separate `SCHEDULE_DAY_NOTES` card, which has since been removed in favor of attaching the link directly to the booking card it's about.
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
- `receiptImage`: optional — base64 `data:image/jpeg;...` string of a compressed receipt photo (added 2026-08-31, see Completed Work History; capped at ~700KB encoded, no separate Storage bucket)
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
- EUR-denominated line items show a small gray KRW estimate under the main amount (`formatEurKrwEstimate()`, fixed rate `EUR_TO_KRW_RATE = 1600`, display-only, not stored in Firestore) — added 2026-08-29

### Shopping

Path:

```text
tripData/shopping/items/{id}
```

Fields:

- `title`: string, required
- `link`: string (URL), required
- `note`: string, optional
- `createdAt`: Firestore `serverTimestamp()`
- `createdBy`: display name
- `updatedAt`: Firestore `serverTimestamp()`
- `updatedBy`: display name

Current UI:

- `쇼핑` tab, positioned between `지출` and `체크리스트`
- `+ 쇼핑 추가` opens a small form: 타이틀 (title), 링크 (link, `type="url"`), 메모 (optional note)
- Each saved item renders as a card whose title is a clickable link (`target="_blank"`) that opens the URL directly, with 수정/삭제 buttons below
- realtime Firestore subscription via `onSnapshot`, ordered by `createdAt` descending (newest first) — same CRUD pattern as Schedule/Expenses (`resetShoppingForm`/`openShoppingForm`/`closeShoppingForm`, `saveShoppingItem`, `deleteShoppingItem`, `loadShopping`)
- Wired into `loadTripData()`, `clearSharedDataViews()`, and the `setDataControlsEnabled()` disabled-when-logged-out list, consistent with the other tabs

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

- `totalStaticSlots` counts every static leaf item regardless of hidden state (i.e. the true flattened count, currently `114`); `totalStaticItems`/`totalItems` count only currently-visible (non-hidden) items and drive the progress bar.
- `checkAll()` iterates `0..totalStaticSlots-1` and skips any key present in `hiddenKeys`, so hidden items are never force-checked and the presence of hidden items earlier in the list no longer shifts which keys get checked.
- `uncheckAll()` clears `state`/`checkedBy` to `{}` via `updateDoc()` (not `setDoc(...,{merge:true})` — see the bug fix note immediately below for why that distinction matters).

**Bug fixed in commit `681082d`:** restoring a hidden item (`복원`) previously did not persist — the item would come back after the next Firestore sync. Root cause: `setStaticItemHidden()` deleted the key from the local `hiddenKeys` object, then wrote the whole (now-smaller) object via `setDoc(checklistRef, {hiddenKeys}, {merge:true})`. Firestore's `merge:true` recursively merges nested map fields but never removes a key that is simply absent from the payload — the server-side `hiddenKeys.<key>` value was left untouched, so the next `onSnapshot` update brought the "hidden" flag right back. Fixed by sending `{ hiddenKeys: { [key]: deleteField() } }` instead, which explicitly deletes just that nested field. The same root cause affected two other spots, fixed at the same time:
- `toggleItem()`: unchecking an item deleted `checkedBy[key]` locally but re-sent the whole map via merge, so the stale `checkedBy` entry was never actually removed from Firestore (harmless in the UI, since the badge only displays when `state[key]` is also true, but it left orphaned data). Now uses `{ checkedBy: { [key]: deleteField() } }` when unchecking.
- `uncheckAll()`: sent `state: {}, checkedBy: {}` via `setDoc(...,{merge:true})` — an empty object merges nothing, so this was very likely a **no-op against already-stored data**, meaning `전체 미완료` may not have been actually clearing anyone's checked state before this fix. Switched to `updateDoc(checklistRef, {state:{}, checkedBy:{}, ...})`, which replaces those fields outright (unlike `setDoc`+merge, `updateDoc` does not recursively merge a provided object into an existing map — it replaces the field), while leaving `hiddenKeys` and other fields untouched.
- Added `deleteField` to the `firebase-firestore.js` imports to support all three fixes.

## Firestore Security Rules

Rules file:

```text
firestore.rules
```

Firebase config file:

```text
firebase.json
```

Current rules intent (as of commit `595aa40`):

- signed-in users only
- **read**: `jhlee8379@gmail.com`, `sweetylove0116@gmail.com`, and `yeonholee1024@gmail.com` (`isAllowedUser()`)
- **write**: only `jhlee8379@gmail.com` and `sweetylove0116@gmail.com` (`isEditUser()`) — `yeonholee1024@gmail.com` is read-only at the rules level, not just in the UI
- allow read/write (subject to the above split) on `trip/checklist`
- allow read/write (subject to the above split) on `tripData/schedule/items/{id}`
- allow read/write (subject to the above split) on `tripData/expenses/items/{id}`
- allow read/write (subject to the above split) on `tripData/checklistCustom/items/{id}` (added in commit `8c36e0d`, for the new custom checklist item feature)
- allow read/write (subject to the above split) on `tripData/shopping/items/{id}` (added in commit `fe8a997`, for the new 쇼핑 tab)
- deny all other document paths

Prior to commit `595aa40`, `isAllowedUser()` covered both read and write for the same 2 emails (no read-only tier existed). The split into `isAllowedUser()`/`isEditUser()` was added specifically to support `yeonholee1024@gmail.com`'s read-only access — see Authorized Users above.

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
- current cache name is `spain-trip-pwa-v22` (bumped for the 총 지출/오늘 지출 summary-card KRW hint; `v21` was for the expense receipt-photo attachment feature, `v20` was for the EUR→KRW estimate display on expense amounts, `v19` was for the shopping-tab rename to 유용한 링크/Useful Links, `v18` was for the Air Europa UX6007 8/29 flight delay update, `v17` was for the read-only account feature, `v16` was for the Mallorca luggage plan update, `v15` was for the restore bug fix and BCN storage checklist removal, `v14` was for the Air Europa UX6007 boarding pass card, `v13` was for the Air Europa dangerous goods card, `v12` was for the 9/4 schedule card, `v11` was for the 8/28 departure time update, `v10` was for the new shopping tab, `v9` was for the 8/29 card chronological reorder, `v8` was for the Record Go rental car schedule card, `v7` was for the 8/29 Mallorca transfer schedule card, `v6` was for the hotel review link consolidation, `v5` was for the booking card position fix, `v4` was bumped speculatively and did not by itself change the layout)

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
- Entered complete booking details for `Alberg Centre Esplai` (with payment confirmation), `Gran Hotel Sóller`, `Meliá Palma Marina`, and (as of commit `0d08aca`) `Casp 74 Apartments` from the supplied confirmations. All 4 accommodation bookings are now confirmed and entered — see the new subsection below for the last one.
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

- Added `SECTION_BOOKING_STYLE_CARDS`, a second generic map keyed by checklist section title, rendered with `createBookingCard()` (the booking-card look, reused for non-accommodation reference info). **As of commit `6b34a3f`, each key's value is an array of cards** (originally a single card object; refactored the same way `SCHEDULE_DAY_NOTES` was, to allow more than one reference card per checklist section) — see the "Air Europa Dangerous Goods Card" entry further down.
- Generalized `createBookingCard()` to accept an optional `cardTitle` field; falls back to `예약 정보` when absent, so existing `ACCOMMODATION_BOOKINGS` cards are unaffected.
- `renderChecklist()` now also appends every `SECTION_BOOKING_STYLE_CARDS` card (in array order) after a section's items and after any `SECTION_INFO_CARDS` transit card, if any exist for that section.
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
- `Casp 74 Apartments` had no `ACCOMMODATION_BOOKINGS` entry at the time of this commit; it was added afterward in commit `0d08aca` (see below).

### Casp 74 Apartments Booking Confirmation (Final Accommodation)

Committed and pushed directly to `main`:

- `0d08aca feat: add Casp 74 Apartments booking confirmation`

- Added the `9/3(목)~9/7(월) — Casp 74 Apartments` entry to `ACCOMMODATION_BOOKINGS`, matching the existing checklist heading key exactly.
- Fields: `hotelName: "CASP74 Apartments"` (renamed from `CASP74 아파트` in commit `c42fc1a`), `checkin: "2026-09-03"`, reservation number `260313304620`, supplier reference `9091200694940`, stay `2026-09-03 → 2026-09-07 (4박)`, room type (아파트, 침실 1개), occupancy (성인 2명, 아동 1명 11세), price `₩1,724,556`, guest name, contact, and address.
- Breakfast marked `excluded` (조식 미포함) — the only one of the 4 bookings without breakfast.
- Note field covers: free cancellation deadline (2026-08-25 18:00, already passed as of this update — no longer actionable, informational only), voucher-at-check-in requirement, and the self-parking fee (€28/day) whose payment status is unconfirmed.
- This is the 4th and last of the trip's accommodation bookings — all stays (`Alberg Centre Esplai`, `Gran Hotel Sóller`, `Meliá Palma Marina`, `Casp 74 Apartments`) are now fully confirmed and entered.
- No rendering code changes were needed: because of the `ACCOMMODATION_BOOKINGS_BY_CHECKIN` lookup added in commit `9458e6d`, this booking automatically appears on the `일정` tab under `2026-09-03` (Wed 9/3) as soon as the data was added.

### Meliá Palma Marina Breakfast Confirmed as Included

Committed and pushed directly to `main`:

- `a37482c fix: confirm Meliá Palma Marina breakfast is included, add payment breakdown`

- Changed the breakfast badge from `조식 미확인` (unknown) to `조식 포함 (아동 조식 포함)` (included, child breakfast included), based on the supplied payment details screenshot.
- Added a `룸타입` row: `Classic Room (Flexible Meliárewards)`.
- Expanded the `요금` row from a flat total to a breakdown: room `€667.85` + child (2–11) breakfast `€139.65` = total `€807.50`, with 10% national tax included in that total.
- This was the last of the 4 confirmed bookings with an unresolved breakfast status; all 4 accommodation cards now have a definitive breakfast badge (3 `included`, 1 `excluded` for Casp 74 Apartments).

### CASP74 Apartments Rename and City Tax Detail

Committed and pushed directly to `main`:

- `c42fc1a fix: rename CASP74 아파트 to CASP74 Apartments, add city tax detail`

- Renamed `hotelName` from `CASP74 아파트` to `CASP74 Apartments` (matches the official listing name). This also updates the schedule tab's booking card title (`"${hotelName} 예약 정보"`), which is built from `hotelName` automatically — no separate title field to update.
- Added a `시티 택스` row: Barcelona city tax is `€6.88` per adult per night, **adults only** (children excluded) — roughly `€55.04` total for 2 adults × 4 nights, paid locally in cash at check-out.
- Removed the earlier generic `€0.65~2.5` city tax range from the note field (that figure came from a general Spain-wide terms screenshot); the specific `€6.88` figure confirmed for this property supersedes it.

### Booking Card Position Fix (Between Transit and Hotel-Review Notes on 8/28)

Committed and pushed directly to `main`:

- `6dc7416 fix: place Alberg Centre Esplai booking card between the transit and hotel-review notes on 8/28`
- `5cc10ee chore: bump PWA cache version to v5 for booking card position fix`

Corrects a misread of an earlier request. The user asked for the `Alberg Centre Esplai` booking card to sit **between** the "바르셀로나 공항 → Alberg Centre Esplai 이동" and "Alberg Centre Esplai 호텔 후기 (참고용)" day-note cards on `2026-08-28`. The prior response (which only bumped the PWA cache to `v4`, no code change) instead read "위에 오도록" as "above every card on the date" and concluded the existing order (booking card first, before all day notes — including the airport departure plan card) already satisfied the request. It didn't; the user reported the position still hadn't changed as intended.

- Added an optional `scheduleOrder` field to `ACCOMMODATION_BOOKINGS` entries: the count of `SCHEDULE_DAY_NOTES` cards to render before the booking card is spliced in among them. Undefined/`0` (all other bookings) keeps the previous default: booking card first, before all day notes.
- Set `scheduleOrder: 2` on `Alberg Centre Esplai` only, so `2026-08-28` now renders: 인천공항 출발 계획 → 바르셀로나 공항 이동 → **[예약 카드]** → 호텔 후기.
- `renderSchedule()` now builds `dayNoteCards` as an array of individual card HTML strings (rather than one joined string) so the booking card can be `.splice()`-inserted at the right index; the final combined markup is still a single joined string appended after the date header.
- Bumped `sw.js` cache to `spain-trip-pwa-v5` since this is a visible layout change on an already-cached page.

### Hotel Review Link Folded into Booking Card (Alberg Centre Esplai)

Committed and pushed directly to `main`:

- `4c05f6c feat: fold Alberg Centre Esplai hotel review link into the booking card`
- `22b4834 chore: bump PWA cache version to v6 for hotel review link consolidation`

Per user request: removed the separate "Alberg Centre Esplai 호텔 후기 (참고용)" `SCHEDULE_DAY_NOTES` card for `2026-08-28` and instead attached its link directly to the `Alberg Centre Esplai` `ACCOMMODATION_BOOKINGS` entry.

- Added a `link` field to the `Alberg Centre Esplai` booking entry: `https://blog.naver.com/eudemonic005/224348941656` (same URL the removed card used).
- Added `link` rendering to both booking-card code paths: the inline HTML template in `renderSchedule()` (the one actually shown on the schedule tab) and the DOM-based `createBookingCard()` (used for the baggage info card and available if a booking card is ever rendered elsewhere again) — both show a `🔗 참고 링크` link under the note.
- With the review card removed, `2026-08-28` now has only 2 `SCHEDULE_DAY_NOTES` cards (departure plan, airport transit); `scheduleOrder: 2` on the booking still correctly places it last, right after the transit card — no `scheduleOrder` value change was needed since `2` already meant "after all day notes" once there were only 2.

### 8/29 Mallorca Transfer Schedule Card

Committed and pushed directly to `main`:

- `2144ebb feat: add 8/29 Mallorca transfer schedule card (Alberg Centre Esplai -> BCN T1)`
- `0f709d7 chore: bump PWA cache version to v7 for 8/29 schedule card`

Added a new `SCHEDULE_DAY_NOTES["2026-08-29"]` entry (first card of the day, no `checkinBooking` on this date so no `scheduleOrder` interaction), summarizing the user-supplied plan for the Alberg Centre Esplai → BCN Terminal 1 → Mallorca (PMI) transfer:

- Flight: Air Europa BCN → PMI, 08:40 → 09:25, Terminal 1.
- Recommended hotel departure: `06:10`, called out explicitly per the user's instruction (rather than the 06:00 wake-up time also mentioned in their notes).
- Taxi transfer estimate: ~8–10 minutes, €20–24, targeting T1 arrival by 06:20–06:30.
- Airport timeline: check-in/baggage 06:30–06:40, security from 06:40, gate by ~07:30, boarding/departure 08:40.
- Note covers: taxi recommended over public transit for 3 people + luggage; Air Europa uses T1; despite being a short domestic/Schengen hop, the extra checked baggage justifies the ~06:30 airport-arrival target; and the early departure accounts for this being the morning right after the long-haul 8/28 인천 → 바르셀로나 flight.
- The official Aena Barcelona-El Prat airport info link mentioned in the user's source material was not included, since no concrete URL was visible in the supplied content — can be added once the user provides the exact link.
- Bumped `sw.js` cache to `spain-trip-pwa-v7`.

### Record Go Rental Car Reservation and Pickup Guide (8/29 Schedule)

Committed and pushed directly to `main`:

- `d5325d2 feat: add Record Go rental car reservation and pickup guide to 8/29 schedule`
- `75f4e72 chore: bump PWA cache version to v8 for rental car schedule card`

Added a second `SCHEDULE_DAY_NOTES["2026-08-29"]` card (after the Mallorca transfer card), from the user-supplied Record Go booking confirmation screenshot and a Naver blog pickup guide.

- Reservation details: `예약번호 75/2026-74404`, Seat Arona or similar (`CGAR`), 5 seats / 5 doors / 2 luggage / automatic (`A`), rental period `2026-08-29 10:30 → 2026-09-03 08:00` at Palma de Mallorca, `Total Comfort Coverage`, `Full-Full` fuel policy, total `€198.11` (VAT included, already paid), fuel deposit noted as a separate variable fee not included.
- Note condenses the blog's pickup walkthrough into 6 steps: kiosk/desk reservation lookup (by reservation number or QR), confirming/adding booking details on site, showing international driving permit + passport (+ Korean license), prepaying fuel under the Full-Full policy (refunded if returned full), reviewing and photographing the contract, then walking to the assigned parking garage (5–10 min from the terminal) to collect the keys after another document check — plus a reminder to photograph the car's exterior/interior/fuel gauge before driving off.
- Linked `https://blog.naver.com/kwonsy0707/224323000845` (the source blog post) as the reference.
- This complements, rather than duplicates, the existing checklist section `🚗 3. 마요르카 렌터카` (`index.html` line ~684), which has simple checkbox reminders (예약 확인, 서류 준비, 사진 촬영 등) without the reservation number, pricing, or step-by-step guide — the two are intentionally different: checklist items are actionable checkboxes, this schedule card is reference detail tied to the pickup date.
- Bumped `sw.js` cache to `spain-trip-pwa-v8`.

### Gran Hotel Sóller Card Reordered for Chronological Order (8/29)

Committed and pushed directly to `main`:

- `2dc355e fix: move Gran Hotel Sóller booking card after transfer and rental car cards on 8/29 for chronological order`
- `0865cd9 chore: bump PWA cache version to v9 for 8/29 card order fix`

The `Gran Hotel Sóller` booking (check-in `2026-08-29`) had no `scheduleOrder`, so it defaulted to `0` and rendered **before** both `SCHEDULE_DAY_NOTES` cards for that date — ahead of the 마요르카 이동 and 렌터카 수령 cards, even though checking into the hotel happens after both of those in real life. User asked for the cards to read in chronological order.

- Set `scheduleOrder: 2` on `Gran Hotel Sóller`, so `2026-08-29` now renders: 마요르카 이동 (Alberg Centre Esplai → BCN T1) → Record Go 렌터카 수령 → **[Gran Hotel Sóller 예약 카드]**, matching the day's actual sequence of events.
- Bumped `sw.js` cache to `spain-trip-pwa-v9`.

### New Shopping Tab (Shared Title+Link Bookmarks)

Committed and pushed directly to `main`:

- `fe8a997 feat: add 쇼핑 (shopping) tab for shared title+link bookmarks`
- `b5f8b7c chore: bump PWA cache version to v10 for shopping tab`

Added a fourth tab, `쇼핑`, positioned between `지출` and `체크리스트`. Purpose: let either user quickly save a title + a link (e.g. a product page, a packing-list article) to a shared list the other person can also see and open.

- New Firestore collection `tripData/shopping/items` with fields `title`, `link`, optional `note`, plus the usual `createdAt`/`createdBy`/`updatedAt`/`updatedBy`. See the Firebase Data Model section above for full field docs.
- New form (`shoppingForm`): 타이틀 (required text), 링크 (required `type="url"` input), 메모 (optional textarea).
- Each saved item renders as a `.shopping-item` card whose **title itself is the clickable link** (`target="_blank"`, opens immediately) — no separate "open" button, matching the user's request to make items "누르면 링크도 바로바로 실행되도록" (open on click). 수정/삭제 buttons sit below.
- Implementation mirrors the existing Schedule/Expenses CRUD pattern exactly: `resetShoppingForm()`, `openShoppingForm(item)`, `closeShoppingForm()`, `renderShopping()`, `saveShoppingItem(event)`, `deleteShoppingItem(id)`, `loadShopping()` (realtime `onSnapshot`, ordered by `createdAt` descending so newest links show first).
- Wired into `loadTripData()`, `clearSharedDataViews()` (resets `shoppingItems = []` and re-renders on logout), and the `setDataControlsEnabled()` list (`addShoppingBtn` disabled when logged out, alongside the other add buttons).
- `.tabs` CSS grid changed from `repeat(3,1fr)` to `repeat(4,1fr)` to fit the new tab button.
- Updated `firestore.rules` to add `"shopping"` to the allowed `section` list under the existing `tripData/{section}/items/{itemId}` wildcard rule — same not-yet-deployed caveat as the other recent rule changes (see Firestore Security Rules section and Future Work Notes).
- Bumped `sw.js` cache to `spain-trip-pwa-v10`.

### 8/28 Departure Time Update and Bicycle Mention Removal

Committed and pushed directly to `main`:

- `a1c2ec9 fix: update Suwon departure time to 07:10, remove bicycle luggage mention on 8/28 card`
- `67d50b4 chore: bump PWA cache version to v11 for 8/28 departure time update`

- Changed the `2026-08-28` "인천공항 출발 계획" card's `수원 출발` row from `06:30~06:40` to `07:10` (single time, per how the user phrased the request).
- Removed the `자전거 관련 짐/추가 수하물까지 고려해` phrase from the card's note; also updated the note's departure time reference from `06:30` to `07:10` for consistency.
- **Intentionally left unchanged:** the downstream rows (`07:50~08:10 인천공항 주차장 도착`, and everything after). Only the departure time and the bicycle-luggage phrase were explicitly requested. This does leave a tight ~40–70 minute implied drive window between the new `07:10` departure and the still-`07:50~08:10` arrival row — flagged to the user in chat; will need a follow-up edit to `SCHEDULE_DAY_NOTES["2026-08-28"][0].rows` if they want the rest of the timeline shifted later to match.
- Bumped `sw.js` cache to `spain-trip-pwa-v11`.

### 9/4 Barcelona Sightseeing Route and Sagrada Familia Entry Tips

Committed and pushed directly to `main`:

- `77b46e9 feat: add 9/4 Barcelona sightseeing route and Sagrada Familia entry tips`
- `c6e3754 chore: bump PWA cache version to v12 for 9/4 schedule card`

Added two new `SCHEDULE_DAY_NOTES["2026-09-04"]` cards, from user-supplied route-planning-app and blog screenshots:

1. **"바르셀로나 관광 코스 (가우디 투어 + 고딕 지구)"** — a 16-stop walking route starting from Casp 74 Apartments: Casa Batlló → Casa Milà → Park Güell → Casa Angela (reservation 14:30) → Sagrada Familia (interior entry reservation 16:15) → Gràcia street → Plaça Catalunya → La Rambla → Boqueria market → Gothic Quarter → Barcelona Cathedral → Churrería Laietana → Palau de la Música Catalana → Arc de Triomf → El Glop Plaça Catalunya. Each row shows the stop number, the inter-stop walking distance from the source app (e.g. `2 (849m)`), the place name, category/neighborhood tags, and any notes carried over from the screenshot (e.g. `가우디투어 · 예약가능`, `이베리코항정살, 먹물빠에야 1인분가능, 꿀대구`).
2. **"사그라다 파밀리아 입장 팁 (내부입장 16:15 예약)"** — dress code and entry requirements from the supplied blog tip: bring passport + ticket (screenshot OK), download the official app in advance via the confirmation email's link, no sleeveless/deep-cut/sheer tops, no short shorts/miniskirts above the knee, hats must come off inside, avoid worn-out shoes — with a note that non-compliant dress can block entry.

No `link` field was set on either card since no reference URL was included in the supplied screenshots.

### Air Europa Dangerous Goods Card (항공 Checklist Section)

Committed and pushed directly to `main`:

- `6b34a3f feat: add Air Europa dangerous goods / carry-on guidance card to 항공 checklist section`
- `ad296b0 chore: bump PWA cache version to v13 for Air Europa dangerous goods card`

- Refactored `SECTION_BOOKING_STYLE_CARDS` from `{ sectionTitle: singleCardObject }` to `{ sectionTitle: [cardObject, ...] }`, mirroring the earlier `SCHEDULE_DAY_NOTES` array refactor, so a checklist section can show more than one booking-style reference card. `renderChecklist()` now does `(SECTION_BOOKING_STYLE_CARDS[section[0]] || []).forEach(...)` instead of rendering a single card.
- Added a second card to `✈️ 1. 항공`, from a supplied Air Europa check-in screenshot ("Are your bags safe for take-off?"): "Air Europa 수하물 반입 유의사항 (위험물)".
  - **기내 반입 가능**: e-cigarettes, power banks/spare batteries (under 100Wh only), electronic devices, lighters/matches.
  - **위탁 수하물만 가능** (airline confirmation required, marked `*` in the source): aerosols, camp stoves, dry ice, firearms & ammunition, gasoline-powered equipment, scuba tanks.
  - **반입 전면 금지**: flammable liquids/gases, fireworks, corrosive chemicals/household cleaners, small lithium-battery-operated vehicles, biohazard/infectious materials, radioactive products, explosives.
  - Note covers the 100Wh battery limit and the requirement to declare dangerous goods at check-in per country/international regulations.
- This card sits directly below the existing BCN↔PMI baggage size/weight card (added in commit `48b951a`) within the same `✈️ 1. 항공` section — both are display-only, unrelated to Firestore.

### Air Europa UX6007 Boarding Pass Info (8/29 Schedule)

Committed and pushed directly to `main`:

- `c02bea4 feat: add Air Europa UX6007 boarding pass info card to 8/29 schedule`
- `fd8a95e chore: bump PWA cache version to v14 for boarding pass card`

Added a new `SCHEDULE_DAY_NOTES["2026-08-29"]` card, from 3 user-supplied Air Europa boarding pass screenshots (one per family member), inserted right after the existing "마요르카 이동" card and before "Record Go 렌터카": "Air Europa UX6007 탑승권 정보 (BCN → PMI)".

- Flight `UX6007` (Air Europa), `BCN → PMI` via Terminal 1, `08:40 → 09:25` on 29 Aug, boarding time `07:55`, gate shown as "탑승 전 모니터에서 확인" (not yet assigned in the source), `Economy` class.
- All 3 passengers with their boarding group/seat: `JAEHWAN LEE · GROUP 3 · SEAT 22B`, `YEONHO LEE · GROUP 3 · SEAT 22C`, `HYELEE YI · GROUP 3 · SEAT 22A`.
- **Deliberately did not attempt to reproduce the QR codes from the screenshots.** A QR code photographed from a screen can't be reliably re-encoded into a working scannable image without the underlying boarding-pass data, and a broken/fake QR at the gate would be worse than no QR at all. The note field instead tells the user to open the real digital boarding pass (with its live QR) from the Air Europa app or confirmation email at the airport, and that this card is reference info only.
- Since a 3rd day-note card was added ahead of the existing `Record Go 렌터카` card, `Gran Hotel Sóller`'s `scheduleOrder` was bumped from `2` to `3` (`ACCOMMODATION_BOOKINGS` entry, check-in `2026-08-29`) so its booking card still renders last, after all 3 day notes, preserving the chronological order fixed earlier (see "Gran Hotel Sóller Card Reordered for Chronological Order" above). Verified by simulating the splice logic: `["마요르카 이동","탑승권 정보","렌터카 수령"]` with `scheduleOrder: 3` inserts the booking card at the end, as intended.

### Restore Bug Fix and BCN Airport Storage Removal

Committed and pushed directly to `main`:

- `681082d fix: restore/uncheck-all Firestore bug; remove BCN airport luggage storage from checklist`
- `8902305 chore: bump PWA cache version to v15 for restore bug fix and checklist item removal`

Two changes, reported together by the user and fixed/applied in the same commit:

**Bug fix** — see the "Bug fixed in commit `681082d`" note under "Edit Mode and Static Item Hide/Restore" above for the full root-cause writeup. Summary: `setDoc(...,{merge:true})` cannot remove keys from a nested map field (`hiddenKeys`, `checkedBy`) by simply omitting them from the payload — it only merges what's present. Fixed `setStaticItemHidden()` (restore) and `toggleItem()` (uncheck) to use `deleteField()` for the specific key being removed, and switched `uncheckAll()` from a merge-with-empty-objects `setDoc` (a no-op against existing data — likely meaning `전체 미완료` wasn't actually working before this fix either) to `updateDoc()`, which replaces the specified fields outright.

**Content change** — removed the BCN airport luggage storage plan from the checklist, since it was dropped in favor of not storing bags there:
- Removed `🧳 4. 짐 → BCN 공항에 보관` entirely (4 items).
- Removed `BCN T1 수하물 보관 위치 확인` and `BCN 도착 후 보관짐 수령` from `🗺️ 8. 이동` (2 items).
- Flattened checklist count: `119` → `113`. See "Known index drift at `681082d`" in the Firebase Data Model section above — the shift is non-uniform (4 or 6 positions depending on where an item sits relative to both removed blocks), so both users should review their checked/hidden state across the whole checklist, not just a specific range.

### Mallorca Luggage Plan Update (3 Carriers + 1 Extra Bag)

Committed and pushed directly to `main`:

- `c0905de feat: update Mallorca luggage plan to 3 carriers + 1 extra large bag`
- `e7d9f49 chore: bump PWA cache version to v16 for Mallorca luggage update`

- `🧳 4. 짐 → 마요르카로 가져갈 짐 → 20인치 캐리어 2개` (index `38`) reworded to `20인치 캐리어 3개` — same index, no drift for this label change alone.
- Added a new item `추가 대형 수하물 1개` right after it, as index `39`.
- Flattened checklist count: `113` → `114`. See "Known index drift at `c0905de`" in the Firebase Data Model section above — every index from `39` onward shifts by `+1`.

### Read-Only Account (yeonholee1024@gmail.com)

Committed and pushed directly to `main`:

- `595aa40 feat: add read-only account yeonholee1024@gmail.com`
- `275ab5f chore: bump PWA cache version to v17 for read-only account feature`

Added a third allowed account, `yeonholee1024@gmail.com` (display name `연호`), that can sign in and view every tab but cannot make any change — check/hide/add/edit/delete anything, or use `전체 완료`/`전체 미완료`/`항목 편집`.

**Security boundary (the part that actually matters):** `firestore.rules` now has two functions instead of one — `isAllowedUser()` (all 3 emails, gates `read`) and `isEditUser()` (only the original 2 emails, gates `write`) — applied to both `trip/checklist` and `tripData/{section}/items/{itemId}`. This is enforced server-side regardless of what the client does. **Not yet deployed to production** (see the Current Git State warning above and Future Work Notes).

**Client-side UX (does not by itself enforce anything — a modified client could bypass all of this):**
- Added `yeonholee1024@gmail.com` to `USERS` (so it passes the existing allowed-account gate in `onAuthStateChanged` and can sign in) and to a new `READ_ONLY_USERS` `Set`.
- New global `isReadOnly` flag, computed as `READ_ONLY_USERS.has(user.email)` in `onAuthStateChanged`, reset to `false` in `clearSharedDataViews()` on logout.
- `setDataControlsEnabled(enabled)` now computes `canEdit = enabled && !isReadOnly` and disables `addScheduleBtn`/`addExpenseBtn`/`addShoppingBtn`/`uncheckAllBtn`/`checkAllBtn`/`editModeBtn` accordingly — a read-only user stays logged in (`enabled=true`) but these stay disabled.
- Checklist: static item checkboxes (`cb.disabled`), the edit-mode 숨기기/복원 toggle button, custom item checkboxes and their 삭제 button, and the add-item form's input/button all also check `isReadOnly`. (Largely redundant with `editModeBtn` already being disabled — entering edit mode is unreachable for a read-only user through the UI — but kept as defense in depth.)
- Schedule/Expense/Shopping list item templates now conditionally omit the `.item-actions` block (수정/삭제 buttons) entirely when `isReadOnly`, rather than rendering always-clickable buttons whose handlers silently no-op. Previously these buttons had no `disabled` gating at all — only the click-handler functions checked `currentUser`, so a read-only user (who wasn't a concept yet) would have seen fully interactive-looking buttons.
- Every Firestore-writing function gained `|| isReadOnly` in its early-return guard: `toggleItem`, `setStaticItemHidden`, `checkAll`, `uncheckAll`, `addCustomChecklistItem`, `toggleCustomChecklistItem`, `deleteCustomChecklistItem`, `saveSchedule`, `deleteScheduleItem`, `saveExpense`, `deleteExpenseItem`, `saveShoppingItem`, `deleteShoppingItem`.
- Added a `👀 읽기 전용` badge next to the display name in the header's `authBox` when `isReadOnly` is true.
- Updated copy that previously named only 재환/혜리 to be generic: the blocked-user error message (`"⛔ 이 여행 페이지는 허용된 계정만 사용할 수 있습니다."`), the app footer, the header's pre-login subtitle (now lists all 3 names, marking 연호 read-only), and the `전체 미완료` confirm dialog.

### Air Europa UX6007 Flight Delay Update (8/29 Outbound Leg)

Committed and pushed directly to `main`:

- `22afe63 8/29 Air Europa UX6007 항공편 지연 반영 (08:40→09:25 → 09:20→09:48)`
- `80f0402 chore: bump PWA cache version to v18 for flight delay update`

The user reported (with an Air Europa delay-notification screenshot, reservation `852F4P`) that outbound flight `UX6007` (`BCN → PMI`, 2026-08-29) changed from `08:40 → 09:25` to `09:20 → 09:48`. The screenshot's "이전 여행 일정" section also showed the return leg `UX6156` (`PMI → BCN`, 9/3, `10:15 → 11:05`) but with no new replacement time shown — so **only the outbound 8/29 leg's time was updated**; the 9/3 return leg (`Air Europa PMI → BCN 9/3(목) 10:15 → 11:05`, checklist row 787) was left untouched since no new time was confirmed for it.

Three places updated in `index.html`:

- `SCHEDULE_DAY_NOTES["2026-08-29"][0]` ("마요르카 이동 — Alberg Centre Esplai → BCN T1" card): `출발/도착` row `08:40 → 09:25` → `⚠️ 09:20 → 09:48 (지연 변경)`; `🛫 출발` row `08:40 BCN → PMI` → `09:20 BCN → PMI (지연)`; `🚪 게이트 이동` row `07:30 전후` → `08:10 전후 (변경)` (shifted by the same ~40min delay to keep the pre-flight buffer proportional); `🕐 숙소 출발` row annotated `(추천, 유지)` since the earlier prep timeline still applies as a comfortable buffer, not tightened by the delay; note field prefixed with an explicit delay-change callout including the reservation number.
- `SCHEDULE_DAY_NOTES["2026-08-29"][1]` ("Air Europa UX6007 탑승권 정보" card): `출발/도착` row updated to `⚠️ 09:20 → 09:48 (29 Aug, 지연 변경)`. The `🕐 탑승 시각` row (previously `07:55`) was **not** guessed forward — the delay screenshot did not include a new boarding time — so it now reads `⚠️ 미확정 — 앱/이메일에서 새 탑승권 재확인 필요` (unconfirmed, re-check the app/email for the reissued boarding pass), and the note was rewritten to explain the delay and tell the user to re-verify the actual boarding time/gate before departure day rather than relying on this reference card.
- Checklist leaf item (`✈️ 1. 항공` section, flattened index unchanged — text-only edit): `"Air Europa BCN → PMI 8/29(토) 08:40 → 09:25 확인"` → `"Air Europa BCN → PMI 8/29(토) 09:20 → 09:48 확인 (⚠️ 지연 변경, 기존 08:40 → 09:25)"`. No index drift — same array position, label text only.

No Firestore writes involved (all changes are to static `SCHEDULE_DAY_NOTES`/`data` constants in `index.html`, not to any Firestore-backed document), so this required no live data testing.

### Shopping Tab Renamed to 유용한 링크 (Useful Links)

Committed and pushed directly to `main`:

- `b5a46b3 쇼핑 탭을 '유용한 링크'로 명칭 변경`
- `338d82b chore: bump PWA cache version to v19 for shopping-tab rename`

The user pointed out (with an annotated screenshot) that the 4th tab, previously labeled "쇼핑" (Shopping), should instead read "유용한 링크" (Useful Links) — the tab holds arbitrary reference bookmarks (parking spots, restaurants, etc.), not a shopping list specifically. User-facing text only was changed:

- Tab button label (`data-tab="shopping"`): `쇼핑` → `유용한 링크`
- Panel `<h2>` and add button: `쇼핑` / `+ 쇼핑 추가` → `유용한 링크` / `+ 링크 추가`
- Status/confirm messages in `saveShoppingItem`/`deleteShoppingItem`/`renderShopping`: `쇼핑 링크가 저장되었습니다.` → `링크가 저장되었습니다.`, `쇼핑 링크를 저장하지 못했습니다...` → `링크를 저장하지 못했습니다...`, `이 쇼핑 링크를 삭제할까요?` → `이 링크를 삭제할까요?`, `쇼핑 링크가 삭제되었습니다.` → `링크가 삭제되었습니다.`, `쇼핑 링크를 삭제하지 못했습니다...` → `링크를 삭제하지 못했습니다...`, empty-list message `등록된 쇼핑 링크가 없습니다.` → `등록된 링크가 없습니다.`, load-failure message `쇼핑 목록을 불러오지 못했습니다...` → `링크 목록을 불러오지 못했습니다...`

**Deliberately left unchanged (internal-only, no user-facing effect):**
- All JS identifiers/DOM ids/Firestore collection path (`shoppingItems`, `shoppingCollection`, `shoppingPanel`, `shoppingList`, `shoppingForm`, `addShoppingBtn`, `tripData/shopping/items`, `saveShoppingItem`, etc.) — renaming these would be a pure internal refactor with real risk (Firestore path rename would orphan existing saved links) for zero user-visible benefit.
- The unrelated `EXPENSE_CATEGORIES` dropdown option `"쇼핑"` (지출 tab's expense category, e.g. tagging a purchase as a "Shopping" expense) — a different feature that happens to share the same Korean word; not touched.
- Descriptive prose mentioning "쇼핑" inside `SCHEDULE_DAY_NOTES` cards (e.g. "식사/쇼핑/탑승구 이동", "그라시아 거리 · 쇼핑 · ...") — these describe real shopping streets/activities in the itinerary, unrelated to the app tab's name.

No Firestore schema/index change and no data migration needed — this is a pure UI label change.

### Receipt Photo Attachment for Expenses (Free-Tier Approach)

Committed and pushed directly to `main`:

- `38e2b55 지출 입력에 영수증 사진 첨부 기능 추가 (Firestore base64 저장)`
- `3e03aa2 chore: bump PWA cache version to v21 for receipt photo attachment`

The user asked whether receipt photos could be attached when entering an expense. Before implementing, walked through the storage options with the user via `AskUserQuestion` since this is an infrastructure decision with real trade-offs:

1. **Firebase Storage** — best photo quality/no size cap, but Firebase Storage now requires the paid **Blaze** (pay-as-you-go) plan; new Storage buckets cannot be created on the free Spark plan. Would also need a new `storage.rules` file and console-based deployment (same CLI-blocked situation as `firestore.rules`).
2. **GitHub repo storage with a link** — user's own suggestion, but flagged as a real security problem: uploading from the browser to GitHub requires a write-scoped personal access token, and embedding that token in client-side JS on a publicly-accessible, unauthenticated GitHub Pages site would let anyone extract it via dev tools and gain write access to the repo. Rejected.
3. **Compressed photo stored directly in Firestore as base64** — no new billing plan, no new token/credential, works entirely within the existing free Spark plan and existing `firestore.rules`. Trade-off: lower image quality (client-side compressed) and a de-facto per-photo size ceiling from Firestore's 1MB document limit.

**User chose option 3 (free tier).** Implementation in `index.html`:

- **Form**: added a file input (`#expenseReceiptInput`, `accept="image/*"`) plus a preview thumbnail + "사진 삭제" button (`#expenseReceiptPreviewWrap`) to the 지출 add/edit form, right after the 메모 field.
- **Compression**: `compressImageFile(file, maxDim=1000, quality=0.6)` reads the file via `FileReader`, draws it to an offscreen `<canvas>` resized so its longer side is ≤1000px, and re-encodes as JPEG at quality 0.6 via `canvas.toDataURL(...)`, returning a `data:image/jpeg;base64,...` string.
- **Size guard**: `handleReceiptFileChange()` rejects (with a status-bar error asking the user to retry with a simpler photo) any compressed result whose base64 string exceeds 700KB — leaves comfortable headroom under Firestore's 1MB per-document limit, accounting for the ~33% base64 size inflation over raw bytes.
- **State**: two module-level flags — `pendingReceiptDataUrl` (new/replaced photo pending save) and `removeReceiptFlag` (user explicitly clicked 사진 삭제 while editing an item that had a photo). Both reset in `resetExpenseForm()`.
- **Save logic** in `saveExpense()`: `payload.receiptImage = pendingReceiptDataUrl` if a new photo was chosen; `payload.receiptImage = deleteField()` if the user removed an existing photo without picking a replacement; otherwise the field is omitted entirely from the payload so `updateDoc`'s partial-merge behavior leaves an unrelated field edit (e.g. just changing the amount) from touching any existing photo. (`deleteField()` was already imported for the hide/restore bug fix — see "Restore Bug Fix" above — same correct pattern reused here.)
- **List display**: `renderExpenses()` shows a 52×52px `.receipt-thumb` thumbnail under the note (when `expense.receiptImage` is set), `data-receipt-view="${id}"`. Clicking it opens a full-screen lightbox overlay (`#receiptLightbox`, new markup added just before `<div id="app">`) showing the full compressed image; a close button and a click on the dark backdrop both dismiss it.
- **No `firestore.rules` change needed** — `receiptImage` is just another string field on the existing `tripData/expenses/items/{id}` document, already covered by the existing read/write rules for that path.
- **No Storage/Blaze dependency, no new secrets, no CLI deployment step** — this feature is live as soon as the `main` push reaches GitHub Pages, unlike the still-undeployed Firestore rules work from earlier in the session.

### Expense Summary Cards (총 지출/오늘 지출) Show a KRW Hint Next to EUR Totals

Committed and pushed directly to `main`:

- `d4826eb 지출 요약 카드(총 지출/오늘 지출)에 유로 금액 옆 원화 환산 괄호 표기 추가`
- `e5e8219 chore: bump PWA cache version to v22 for expense summary KRW hint`

Follow-up to the earlier per-item EUR→KRW estimate (see below): the user asked (with a screenshot circling the `총 지출` card's `€546.29 / ₩63,500` line) for the same kind of parenthetical KRW estimate on the `총 지출`/`오늘 지출` summary cards, not just individual expense list rows.

- `formatMoneyGroup(group, showKrwHint = false)` gained a second parameter; when `true`, any `EUR` entry in the group gets `(약 ₩x,xxx)` appended right after its formatted amount, computed via the existing `EUR_TO_KRW_RATE = 1600` constant (reused, not redefined). `KRW`/`USD` entries are untouched.
- `renderExpenses()`: `formatMoneyGroup(totals.total, true)` and `formatMoneyGroup(totals.today, true)` for the 총 지출/오늘 지출 cards. The 카테고리별 지출 rows (`formatMoneyGroup(totals.byCategory[category])`) were deliberately left without the hint (third param omitted → defaults to `false`) since the user's request and screenshot only pointed at the two top summary cards.
- Example result for a mixed-currency total: `€546.29 (약 ₩873,264) / ₩63,500`.
- Purely a display computation on already-derived totals — no Firestore field, no write, no migration.

### EUR Expense Amounts Show a KRW Estimate

Committed and pushed directly to `main`:

- `89174af 지출 목록: 유로 금액 아래에 원화 환산 참고값 표시 (환율 1,600원 가정)`
- `c990523 chore: bump PWA cache version to v20 for EUR→KRW estimate display`

The user asked (with a screenshot circling a `€3.80` amount in the 지출 tab) for a small-text KRW conversion to appear under each EUR expense amount, using a fixed assumed rate of ₩1,600 per €1.

- Added `EUR_TO_KRW_RATE = 1600` (a plain constant, not fetched from any exchange-rate API) and `formatEurKrwEstimate(amount, currency)`, which returns `""` for any non-EUR currency and otherwise formats `amount * 1600` as a KRW currency string prefixed with `약` (approx.) and suffixed with `(환율 1,600원 가정)` (rate assumption disclosed inline so it's never mistaken for a live/authoritative conversion).
- `renderExpenses()`: the `.expense-amount` div is now wrapped together with a new `.expense-amount-krw` div directly beneath it, rendered only when `expense.currency === "EUR"` — `KRW`/`USD` expense rows are unaffected (no extra line).
- New CSS `.expense-amount-krw` (12px, gray `#8a8f98`, right-aligned to match the main amount) added next to the existing `.expense-amount` rule; `.expense-amount` itself gained `text-align:right` so the two lines align consistently now that they sit in a shared column `div` instead of being the sole child of `.expense-top`'s right side.
- This is purely a display computation — no new Firestore field, no write, no migration. Existing expense documents render the estimate automatically since it's derived from already-stored `amount`/`currency` at render time.
- Scope: only the 지출 tab's expense list line items (matching the screenshot). The 총 지출/오늘 지출/카테고리별 summary cards (`formatMoneyGroup`) and the expense add/edit form were intentionally left as-is — not requested, and the summary totals mix currencies (`totals.byCategory` groups are keyed by currency already via `formatMoneyGroup`), so adding a blended KRW estimate there would need separate design thought if wanted later.

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
- schedule tab card order check (superseded, see below): the earlier fix bumped the cache assuming `bookingCardHtml` rendering before all `dayNoteHtml` cards already matched the request; this was a misreading — the user wanted the booking card **between** two specific day-note cards, not merely "above" the last one. Corrected in commit `6dc7416`, see the new subsection below.
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
- empty `rows` day-note card render check (historical): the hotel review card that exercised this case was removed in commit `4c05f6c`; its `link` moved onto the `Alberg Centre Esplai` booking card instead
- booking card link render check: PASS, `🔗 참고 링크` renders under the note on the `Alberg Centre Esplai` schedule-tab booking card, opens in a new tab
- `SCHEDULE_DAY_NOTES["2026-08-28"]` count check: PASS, now has 2 cards (was 3); booking card `scheduleOrder: 2` still renders it last, after both
- empty-schedule message check: PASS, `2026-08-28` (3 day-note cards, 0 Firestore items) no longer shows `등록된 일정이 없습니다`; a date with neither still shows it
- booking card relocation check: PASS, checklist accommodation headings no longer render a booking card; `일정` tab shows the correct booking card under `2026-08-28`, `2026-08-29`, and `2026-08-31` (the three confirmed check-in dates)
- booking card content parity check: PASS, breakfast badge tone/text, all rows (including the `code`-styled reservation number for Gran Hotel Sóller and Meliá Palma Marina), and notes all carried over unchanged to the schedule card
- `ACCOMMODATION_BOOKINGS_BY_CHECKIN` build check: PASS, lookup now has 4 entries (`2026-08-28`, `2026-08-29`, `2026-08-31`, `2026-09-03`), one per confirmed booking
- booking key match check (Casp 74): PASS, `9/3(목)~9/7(월) — Casp 74 Apartments` key matches the existing checklist heading exactly
- Casp 74 schedule card check: PASS, booking card now renders under `2026-09-03` in the `일정` tab with no code changes beyond adding the data
- `scheduleOrder` splice logic check: PASS, simulated with `["A","B","C"]` and `scheduleOrder: 2` produces `["A","B","[booking]","C"]`; `2026-08-28` now renders 인천공항 출발 계획 → 바르셀로나 공항 이동 → 예약 카드 → 호텔 후기
- `scheduleOrder` default/regression check: PASS, dates and bookings without `scheduleOrder` (`Gran Hotel Sóller`, `Meliá Palma Marina`, `Casp 74 Apartments`) still render their booking card before any day notes, unchanged from before this fix
- service worker cache name check: PASS, `spain-trip-pwa-v17`
- read-only code presence check: PASS, `READ_ONLY_USERS`, `isReadOnly`, and all listed `|| isReadOnly` guards present in `index.html`
- read-only Firestore rules check: PASS (static review), `isEditUser()` correctly excludes `yeonholee1024@gmail.com`; `isAllowedUser()` includes all 3 emails
- read-only Firestore rules deploy: NOT RUN, same standing limitation as all prior rules changes (no Firebase CLI in this environment) — **critical**, this is the actual security boundary and is not yet live
- read-only live sign-in/permission test: NOT RUN, to avoid touching production data/auth during this session; needs manual verification by the user (see Future Work Notes)
- Mallorca luggage update check: PASS, flattened count is now `114` (was `113`), confirmed via a Node script parsing the `data` array; `20인치 캐리어 3개` and `추가 대형 수하물 1개` both present at indexes `38`/`39`
- checklist item count check: PASS, flattened count is now `113` (was `119`), confirmed via a Node script parsing the `data` array
- BCN storage removal check: PASS, no remaining `BCN 공항에 보관`/`공항 보관서비스`/`BCN T1 수하물 보관` references in `index.html`; unrelated `카드... 보관` (card storage) items untouched
- `deleteField()` restore fix check: PASS (code review — not live-tested against production Firestore, per the no-live-write policy); `setStaticItemHidden()` now sends `{ hiddenKeys: { [key]: deleteField() } }` on restore instead of the full mutated object
- `uncheckAll()` fix check: PASS (code review), now uses `updateDoc()` instead of `setDoc(...,{merge:true})` with empty objects
- boarding pass card order check: PASS, `2026-08-29` renders 마요르카 이동 → 탑승권 정보 → 렌터카 수령 → Gran Hotel Sóller 예약 카드, matching the simulated splice output
- `SECTION_BOOKING_STYLE_CARDS` array refactor regression check: PASS, `✈️ 1. 항공` renders both cards (baggage size/weight, then dangerous goods) in order; other sections with no entry render nothing extra, same as before
- 9/4 schedule card render check: PASS, both new `2026-09-04` cards (16-stop route, Sagrada Familia tips) render under the date header with all rows and notes
- 8/28 departure time update check: PASS, `수원 출발` row shows `07:10`, bicycle-luggage phrase no longer present in the note
- 8/29 card order check: PASS, `Gran Hotel Sóller` booking card now renders after both day-note cards (마요르카 이동, 렌터카 수령), matching the day's chronological order
- shopping tab code presence check: PASS, `shoppingCollection`, `resetShoppingForm()`, `openShoppingForm()`, `closeShoppingForm()`, `renderShopping()`, `saveShoppingItem()`, `deleteShoppingItem()`, `loadShopping()` all present
- shopping tab duplicate-id check: PASS, no new duplicate ids introduced (only the pre-existing unrelated `googleLoginBtn` duplicate remains, from before the dedicated login page)
- shopping tab wiring check: PASS, `addShoppingBtn` disabled when logged out via `setDataControlsEnabled()`; `shoppingItems` reset and `renderShopping()` called on logout via `clearSharedDataViews()`; `loadShopping()` called from `loadTripData()`
- shopping card click-to-open check: PASS, the item title is itself an `<a target="_blank">`, opens the link directly on click without needing a separate button
- rental car schedule card render check: PASS, second `2026-08-29` card renders reservation rows (with `code`-styled reservation number), the 6-step pickup note, and the reference link
- 8/29 schedule card render check: PASS, new card renders under the `2026-08-29` date header with all 10 rows and the note; no `checkinBooking` on this date so `scheduleOrder` logic does not apply

## Future Work Notes

Recommended next steps:

1. **Test login page UI** — Verify login page displays before authentication and app shows after login with both desktop and mobile viewports.
2. Verify booking info UI rendering with the two allowed accounts in the production app — note booking cards now live in the `일정` tab under each check-in date, not the checklist tab.
3. Verify checklist index `23` in the live app, since its meaning changed at `02f4ce0`, and indexes `97`+ in `🩹 9. 개인용품` / `🔐 10. 출국 직전` since they shifted at `425b7ee` — re-check any previously checked items in those two sections.
4. Confirm whether the `Casp 74 Apartments` self-parking fee (€28/day) was actually paid, and update the note if so — currently marked "지불 여부 미확인" based on the supplied confirmation screenshots.
5. Test Schedule, Expense, and now Shopping CRUD with the two allowed Google accounts to ensure realtime sync works correctly.
6. Confirm whether seed schedule items should be added automatically or entered manually in the app.
7. **Deploy Firestore Rules after review — now the highest-priority item.** They are still not deployed, so production Firestore is not yet protected by them: the custom checklist item and shopping tab features (`tripData/checklistCustom/items`, `tripData/shopping/items`) may not work, and — more importantly — **`yeonholee1024@gmail.com`'s read-only restriction is not actually enforced anywhere until this is deployed.** Use `firebase deploy --only firestore:rules` with Firebase CLI access.
8. Test adding, checking, and deleting a custom checklist item with both allowed accounts in the production app; if it fails with a permission error, that confirms rules need deploying (see item 7).
9. Test the checklist edit mode toggle, hiding a static item, and restoring it, in the production app with both allowed accounts, and confirm the progress bar total updates correctly when items are hidden.
10. Get the exact Aena Barcelona-El Prat airport info URL from the user and add it as a `link` on the `2026-08-29` Mallorca transfer schedule card.
11. Test adding, editing, and deleting a shopping item with both allowed accounts in the production app, and confirm clicking a saved title opens the link in a new tab as intended.
12. Confirm with the user whether the `2026-08-28` departure card's downstream timeline (인천공항 도착, 체크인, 출국심사, etc.) should shift later to match the new `07:10` departure time, or whether the ~40–70 minute drive window is intentional as-is.
13. **High priority:** with both allowed accounts in the production app, test hiding a static checklist item, restoring it, unchecking a checked item, and using `전체 미완료`, to confirm the `deleteField()` fix actually resolves the reported bugs against live Firestore (only code-reviewed so far, not live-tested, per the no-live-write policy during this session).
14. Since the checklist item count changed again (`119` → `113`), both users should open the checklist tab and review their checked/hidden items across the whole list — the index shift from this change is non-uniform, unlike prior single-block shifts.
15. Have `yeonholee1024@gmail.com` sign in once the rules are deployed, and confirm: (a) all 4 tabs load and display data correctly, (b) every add/edit/delete/check control is genuinely disabled or hidden, (c) attempting a write (e.g. via browser dev tools calling the Firestore SDK directly, bypassing the UI) is rejected by the rules, not just the UI.
16. Confirm with the user whether `연호`'s read-only access should ever be upgradable to edit access later (e.g. a simple move from `READ_ONLY_USERS` removal + `isEditUser()` rules update), or whether read-only is permanent for this account for the whole trip.
17. **New (from 8/29 delay update):** the UX6007 boarding pass card's `🕐 탑승 시각` row is now marked unconfirmed (was `07:55`, tied to the old `08:40` departure). Once the user has the reissued boarding pass (new boarding time/gate) for the delayed `09:20` departure, update `SCHEDULE_DAY_NOTES["2026-08-29"][1].rows` in `index.html` with the real value instead of the placeholder warning text.
18. Confirm whether the return leg `UX6156` (`PMI → BCN`, 9/3, currently still `10:15 → 11:05`) is also affected by a schedule change — the delay screenshot listed it under "이전 여행 일정" with no visible new time, so it was left unchanged in this pass; re-check with the user or the Air Europa app closer to the date.
19. Test the receipt photo attachment (add, edit-replace, edit-remove, and the lightbox) live with both allowed accounts, especially on an actual phone camera photo (large original file) to confirm the 700KB post-compression cap doesn't reject normal receipt photos too often — if it does, consider lowering `maxDim`/`quality` further in `compressImageFile()`, or revisit the Firebase Storage + Blaze option now that the user has seen the trade-offs.
20. If the household later decides the free-tier photo quality is too low or 700KB rejections become common, the documented path is: enable Billing → Blaze in the Firebase console, add a `storage.rules` file mirroring the existing `isAllowedUser()`/`isEditUser()` split, deploy it from the console (same as the outstanding `firestore.rules` deploy), and swap `receiptImage` from a base64 string to a Storage download URL.

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
