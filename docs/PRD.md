# PRD: ADS Planner — Android Dev Summit Companion (Kotlin Multiplatform)

| | |
|---|---|
| **Status** | Draft v0.1 |
| **Owner** | cybo42 |
| **Last updated** | 2026-10-07 |
| **Platforms** | Android, iOS (Compose Multiplatform); Desktop JVM as stretch/dev target |
| **References** | [Kotlin Multiplatform](https://kotlinlang.org/multiplatform/) · [ADS Bayview agenda](https://rsvp.withgoogle.com/events/android-dev-summit-bayview/agenda) |

---

## 1. Overview

### 1.1 Problem
Android Dev Summit runs multiple tracks in parallel. Attendees have to scan a web agenda, mentally work out which talks clash, keep a separate list of what they want to see, and set their own alarms. The official page is not optimised for building a personal plan, flagging clashes, or reminding you to change rooms.

### 1.2 Vision
A small, fast, offline-first mobile app that lets an attendee:
1. Browse the agenda **day by day**, with sessions that run at the same time **grouped and flagged**.
2. **Favorite** sessions to build a personal schedule and see **conflicts** between favorites.
3. Get a **reminder notification** shortly before each favorited session.
4. Optionally **sync** their plan across devices (e.g. phone + tablet, Android + iPhone).

### 1.3 Target user
- **Primary:** an in-person ADS attendee (developer) planning ahead and navigating the venue on the day.
- **Secondary:** a remote viewer following the livestream, using the app as a "what's on next" guide.

### 1.4 Success metrics
| Metric | Target |
|---|---|
| Time to build a full-event plan (first launch → all favorites set) | < 5 minutes |
| Favorited sessions missed because of no/late reminder | 0 |
| Reminder delivery accuracy | within ±30 s of scheduled time (Android exact alarm), ±60 s iOS |
| Crash-free sessions | ≥ 99.5 % |
| Cold start to agenda visible (cached data) | < 2 s on a mid-range device |
| Cross-device sync latency (both devices online) | < 5 s |

---

## 2. Goals and non-goals

### 2.1 Goals (v1)
- G1 — Daily agenda view with clear clash detection.
- G2 — Favorites and a personal "My Schedule" view.
- G3 — Reliable local reminder notifications.
- G4 — Required sign-in (Firebase Auth) giving every user an identity.
- G5 — Optional cross-device sync of favorites and reminder settings.
- G6 — Works fully offline after first sign-in and agenda download.
- G7 — Shared business logic and UI across Android and iOS via KMP + Compose Multiplatform.

### 2.2 Non-goals (v1)
- Ticketing, RSVP, or badge management.
- Social features (sharing schedules, seeing friends' plans, chat).
- In-app video or livestream playback (link out only).
- Editing or submitting to the official agenda.
- Web (Wasm/JS) target.
- Venue maps and indoor navigation (possible v2).
- Supporting conferences other than ADS (the design should not block it, but no v1 work goes to it).

---

## 3. Personas and user stories

**Priya — in-person attendee.** Arrives on Day 1 with a list of topics (Compose, KMP, performance). She wants to star talks the night before, see where she has double-booked, and get a nudge to walk to the next room.

**Sam — multi-device attendee.** Plans on a tablet at home, uses a phone at the venue. Expects the same favorites on both.

| ID | As a… | I want to… | So that… |
|---|---|---|---|
| US-1 | attendee | see each day's sessions grouped by time slot | I can understand what is on at any moment |
| US-2 | attendee | see at a glance which sessions overlap | I can choose between clashing talks |
| US-3 | attendee | filter by track, room, format, or favorites | I can find relevant talks quickly |
| US-4 | attendee | search by title, speaker, or tag | I can jump to a talk I heard about |
| US-5 | attendee | star a session | it is added to my personal schedule |
| US-6 | attendee | see a warning when two favorites clash | I can resolve the conflict ahead of time |
| US-7 | attendee | get a notification N minutes before a favorite starts | I don't miss it and have time to change rooms |
| US-8 | attendee | change the reminder lead time globally or per session | reminders fit my walking time |
| US-9 | attendee | sign in with my Google or Apple account | my plan is tied to me |
| US-10 | multi-device user | turn on sync | my favorites show up on all my devices |
| US-11 | attendee | use the app with no connection | it works on flaky conference Wi‑Fi |
| US-12 | attendee | be told if a favorited session's time or room changes | I'm not caught out by agenda updates |
| US-13 | user | sign out and delete my account | I control my data |

---

## 4. Functional requirements

Priority: **P0** = must ship for v1, **P1** = should ship, **P2** = nice to have.

### 4.1 FR-AG — Agenda (daily view)

| ID | Requirement | Priority |
|---|---|---|
| FR-AG-1 | Show one tab per conference day, labeled with weekday and date. Default to "today" during the event, otherwise Day 1. | P0 |
| FR-AG-2 | Within a day, list sessions in chronological order, **grouped into time-slot sections** by start time. | P0 |
| FR-AG-3 | Session card shows title, start–end time, room, speaker name(s), track/tag chips, format (talk, workshop, keynote, lightning, break), and favorite toggle. | P0 |
| FR-AG-4 | Tapping a card opens Session Detail: full description, speakers (name, title, company, photo if available), room, tags, level, links (livestream/recording if provided), favorite toggle, and reminder override. | P0 |
| FR-AG-5 | Times shown in **venue time zone (America/Los_Angeles)** by default, with a setting to show device-local time. The time zone in use is always labeled. | P0 |
| FR-AG-6 | Filter sheet: track, room, format, level, favorites only. Active filters shown as removable chips; filter state kept per session of use. | P1 |
| FR-AG-7 | Search across title, description, speaker name, and tags; results grouped by day. | P1 |
| FR-AG-8 | "Now" / "Up next" indicator during the event; auto-scroll to the current time slot when opening today's tab. | P1 |
| FR-AG-9 | Pull-to-refresh triggers an agenda refresh (see FR-DATA). | P1 |
| FR-AG-10 | Non-session items (breaks, lunch, registration) shown in a muted style and excluded from conflict detection. | P1 |

### 4.2 FR-OV — Overlap and conflict detection

**Definition:** sessions A and B overlap when `A.start < B.end && B.start < A.end` (half-open intervals, so a session ending at 10:00 does **not** overlap one starting at 10:00).

| ID | Requirement | Priority |
|---|---|---|
| FR-OV-1 | In the daily view, sessions that overlap are visually grouped (shared slot header or a side bracket showing "3 sessions in parallel"). | P0 |
| FR-OV-2 | Sessions that start at different times but still overlap (partial overlaps) are marked with an "overlaps with N" indicator listing those sessions. | P0 |
| FR-OV-3 | When ≥ 2 **favorited** sessions overlap, each shows a **conflict badge**, and the day tab shows a conflict count. | P0 |
| FR-OV-4 | My Schedule view lists favorites chronologically and draws conflicting favorites side by side, with the overlap length shown (e.g. "overlaps 20 min"). | P0 |
| FR-OV-5 | Resolve action on a conflict: "Keep this one" (unfavorites the others) or "Keep both" (stops showing the warning for that pair; stored as an acknowledged conflict). | P1 |
| FR-OV-6 | Overlap computation is pure shared Kotlin, O(n log n) per day (sort + sweep), and recomputed reactively when agenda or favorites change. | P0 |
| FR-OV-7 | Optional "travel buffer": treat sessions in different rooms that are less than X minutes apart as a soft conflict (warning, not error). Default off. | P2 |

### 4.3 FR-FV — Favorites

| ID | Requirement | Priority |
|---|---|---|
| FR-FV-1 | Toggle favorite from card, detail, and search results; the change is visible immediately (optimistic local write). | P0 |
| FR-FV-2 | Favorites persist in the local database, scoped to the signed-in `uid`. | P0 |
| FR-FV-3 | My Schedule tab: favorites by day, with conflicts highlighted (FR-OV-4) and gaps shown as free time. | P0 |
| FR-FV-4 | Favorites are keyed by the stable upstream session ID and kept through agenda refreshes. | P0 |
| FR-FV-5 | If a favorited session is **removed** upstream it is shown as "no longer on the agenda" (not silently deleted). If its **time/room changes**, the user is notified (see FR-DATA-5). | P1 |
| FR-FV-6 | Export My Schedule to the device calendar (.ics / platform calendar intent). | P2 |

### 4.4 FR-RM — Reminders

| ID | Requirement | Priority |
|---|---|---|
| FR-RM-1 | Favoriting a session schedules a local notification at `start − leadTime`; unfavoriting cancels it. | P0 |
| FR-RM-2 | Global lead time default **10 min**, options 5 / 10 / 15 / 30; per-session override on Session Detail; option to turn reminders off globally. | P0 |
| FR-RM-3 | Notification content: session title, room, start time; for a conflict, "also starting: X". Tapping it deep-links to Session Detail. | P0 |
| FR-RM-4 | Reminders are reconciled (re-scheduled or cancelled) whenever the agenda refreshes with time changes, lead time changes, favorites sync in from another device, or the app is reinstalled or restored. | P0 |
| FR-RM-5 | Reminders for sessions already in the past, or whose trigger time has passed, are not scheduled. | P0 |
| FR-RM-6 | Ask for notification permission in context (on first favorite or during onboarding), never at cold launch without explanation. If denied, show a non-blocking banner in My Schedule with a link to system settings. | P0 |
| FR-RM-7 | **Android:** use `AlarmManager.setExactAndAllowWhileIdle` when exact-alarm permission is available (`USE_EXACT_ALARM` / `SCHEDULE_EXACT_ALARM`, checking `canScheduleExactAlarms()`); otherwise fall back to inexact alarms and tell the user. Request `POST_NOTIFICATIONS` (API 33+). Re-schedule on `BOOT_COMPLETED`, `TIME_SET`, `TIMEZONE_CHANGED`, and app update. Dedicated notification channel "Session reminders". | P0 |
| FR-RM-8 | **iOS:** use `UNUserNotificationCenter` with `UNCalendarNotificationTrigger`. Because iOS allows at most **64 pending notifications**, schedule the nearest ≤ 60 and top up on app foreground or background refresh. | P0 |
| FR-RM-9 | Reminders are device-local; sync only carries favorites and preferences. Each device schedules its own reminders. A per-device "reminders on this device" toggle lets a tablet stay quiet. | P1 |
| FR-RM-10 | Optional second "starting now" notification. | P2 |

### 4.5 FR-AU — Authentication (required)

| ID | Requirement | Priority |
|---|---|---|
| FR-AU-1 | On first launch, an onboarding/auth gate blocks app content until the user signs in with **Firebase Auth**. | P0 |
| FR-AU-2 | Providers: **Google Sign-In** (Android via Credential Manager, iOS via GoogleSignIn SDK); **Sign in with Apple** on iOS (needed for App Store compliance when offering third-party login); **email link (passwordless)** as fallback. | P0 (Google, Apple), P1 (email link) |
| FR-AU-3 | Auth state persists across launches; the Firebase SDK refreshes tokens. If the device is offline on a later launch, the cached user is used and the app works fully offline. | P0 |
| FR-AU-4 | All local user data (favorites, prefs, acknowledged conflicts) is scoped by `uid` so different accounts on one device never mix. | P0 |
| FR-AU-5 | Sign out: confirm dialog; cancels this device's reminders; local data for that `uid` is kept (so signing back in is instant) but hidden. | P0 |
| FR-AU-6 | Delete account: deletes Firestore user data, then the Firebase Auth user (re-authenticating if Firebase asks for it), then local data. Required by App Store and Play policy. | P0 |
| FR-AU-7 | Clear errors for network failure, cancelled sign-in, and account-exists-with-different-credential (offer account linking). | P1 |
| FR-AU-8 | Agenda data is public and bundled, so the agenda can be shown **behind** the auth gate as a read-only preview. Decide in design review; default is a strict gate. | P2 |

### 4.6 FR-SY — Cross-device sync (optional)

| ID | Requirement | Priority |
|---|---|---|
| FR-SY-1 | A Settings toggle "Sync across devices" is **off by default** and uses the signed-in Firebase identity (no extra sign-in). | P1 |
| FR-SY-2 | Synced data: favorites (with per-session reminder override), global reminder lead time, acknowledged conflicts. **Not synced:** agenda data, per-device reminder toggle, display time-zone preference. | P1 |
| FR-SY-3 | Firestore layout: `users/{uid}` (prefs doc), `users/{uid}/favorites/{sessionId}` with `{ isFavorite, reminderLeadMinutes?, updatedAt, deleted }`. | P1 |
| FR-SY-4 | Security rules: read/write only when `request.auth.uid == uid`; schema validated in rules; covered by emulator tests. | P1 |
| FR-SY-5 | Offline-first: local DB is the source of truth for the UI; writes go into an outbox and are pushed when online. Firestore offline persistence may be used, but correctness must not depend on it. | P1 |
| FR-SY-6 | Conflict resolution: **last-write-wins per session** using `updatedAt` (server timestamp on write, client timestamp in the local outbox); unfavorites are stored as tombstones (`deleted = true`) so they propagate. | P1 |
| FR-SY-7 | When enabled: one-time merge (local ∪ remote, LWW per key), then a real-time snapshot listener while the app is in the foreground and a pull on each foreground. | P1 |
| FR-SY-8 | When disabled: the listener stops, local data stays, and remote data is left untouched (an option offers to delete remote copy). | P1 |
| FR-SY-9 | Incoming remote changes trigger reminder reconciliation (FR-RM-4). | P1 |
| FR-SY-10 | Sync status indicator in Settings: last synced time, pending changes, error state. | P2 |

### 4.7 FR-DATA — Agenda ingestion

| ID | Requirement | Priority |
|---|---|---|
| FR-DATA-1 | **Source investigation (spike):** work out how the official agenda page loads its data (embedded JSON, XHR/JSON endpoint, or server-rendered HTML) and check terms of use. Output: a short ADR choosing the source adapter. | P0 |
| FR-DATA-2 | Pluggable `AgendaSource` interface with implementations: `BundledAgendaSource` (JSON snapshot shipped in app resources), `RemoteJsonAgendaSource`, and if needed `HtmlScrapeAgendaSource` or a self-hosted mirror (e.g. a JSON file in this repo / GitHub Pages, refreshed by a scheduled GitHub Action). | P0 |
| FR-DATA-3 | Normalizer maps source DTOs into the internal model (§6), parses times as `Instant` using the venue time zone, and builds stable IDs (upstream ID, or a hash of title+start if none). | P0 |
| FR-DATA-4 | Refresh policy: on app start (if last refresh > 1 h), on pull-to-refresh, and on background refresh during event days; uses ETag/`If-Modified-Since` where available. Failures keep the last good data and never wipe it. | P0 |
| FR-DATA-5 | Diff old vs new agenda; for favorited sessions with time/room changes or removals, raise an in-app notice (and a local notification if the change is within 24 h of the session). | P1 |
| FR-DATA-6 | Agenda schema version stored; incompatible remote data is rejected and logged. | P1 |

### 4.8 FR-ST — Settings and account

| ID | Requirement | Priority |
|---|---|---|
| FR-ST-1 | Reminder lead time; reminders on/off on this device. | P0 |
| FR-ST-2 | Time display: venue time / device time. | P0 |
| FR-ST-3 | Notification permission status, with a deep link to system settings. | P0 |
| FR-ST-4 | Account: signed-in identity, sign out, delete account. | P0 |
| FR-ST-5 | Sync toggle and status. | P1 |
| FR-ST-6 | About: version, data source, last agenda refresh, open-source licenses. | P1 |
| FR-ST-7 | Theme: system / light / dark (Material 3, dynamic color on Android 12+). | P2 |

---

## 5. Non-functional requirements

| Area | Requirement |
|---|---|
| **Offline** | Everything except sign-in, sync, and agenda refresh works offline. First launch ships a bundled agenda so it works on bad Wi‑Fi once the user has signed in. |
| **Performance** | Cold start < 2 s to cached agenda; scroll at 60 fps on a day of ~60 sessions; overlap computation < 5 ms per day. |
| **Reliability** | Reminders survive reboot, app update, time-zone change, and process death. |
| **Accessibility** | TalkBack/VoiceOver labels for all controls (including conflict badges: "Conflicts with 2 favorites"); dynamic type up to 200 %; contrast ≥ 4.5:1; conflict state never shown by color alone; touch targets ≥ 48 dp / 44 pt. |
| **Privacy and security** | Collect only uid, auth provider info, and favorites/prefs. No third-party analytics in v1 (Crashlytics opt-in only). Firestore rules deny by default. Privacy policy and data-deletion path published for store listings. |
| **Localization** | All strings in Compose Multiplatform resources; English only in v1. |
| **Time correctness** | All storage in UTC `Instant`; display via `kotlinx-datetime` with explicit `TimeZone`. Unit tests cover DST edges and device-TZ ≠ venue-TZ. |
| **Testability** | ≥ 80 % line coverage on `:shared:domain` and `:shared:data-*`; UI screenshot tests for key screens. |
| **Maintainability** | Detekt + ktlint in CI; no platform code in shared modules except via `expect/actual` or interfaces. |
| **Platform minimums** | Android minSdk 26, targetSdk latest; iOS 16+. |

---

## 6. Data model

```kotlin
// :shared:core-model
data class Session(
    val id: SessionId,
    val title: String,
    val description: String,
    val start: Instant,
    val end: Instant,
    val roomId: RoomId?,
    val trackIds: List<TrackId>,
    val speakerIds: List<SpeakerId>,
    val format: SessionFormat,      // KEYNOTE, TALK, WORKSHOP, LIGHTNING, PANEL, BREAK, OTHER
    val level: SessionLevel?,       // INTRO, INTERMEDIATE, ADVANCED
    val tags: List<String>,
    val links: List<Link>,          // livestream, recording, slides
)

data class Speaker(val id: SpeakerId, val name: String, val title: String?, val company: String?, val photoUrl: String?, val bio: String?)
data class Room(val id: RoomId, val name: String, val floor: String?)
data class Track(val id: TrackId, val name: String, val color: String?)
data class ConferenceDay(val date: LocalDate, val sessions: List<Session>)

// User state (scoped by uid)
data class Favorite(
    val uid: String,
    val sessionId: SessionId,
    val isFavorite: Boolean,
    val reminderLeadMinutes: Int?,  // null = use global
    val updatedAt: Instant,
    val deleted: Boolean,           // tombstone for sync
)

data class UserPrefs(val uid: String, val reminderLeadMinutes: Int = 10, val updatedAt: Instant)
data class DevicePrefs(val remindersEnabled: Boolean, val displayTimeZone: DisplayTz, val syncEnabled: Boolean)

// Domain output
data class Overlap(val a: SessionId, val b: SessionId, val minutes: Int)
data class TimeSlot(val start: Instant, val sessions: List<Session>, val parallelCount: Int)
```

**Local DB (SQLDelight) tables:** `session`, `speaker`, `session_speaker`, `room`, `track`, `session_track`, `agenda_meta` (version, etag, lastRefreshed), `favorite`, `user_prefs`, `acknowledged_conflict`, `sync_outbox`, `scheduled_reminder` (sessionId, triggerAt, platformRequestId — used for reconciliation).

---

## 7. Architecture

### 7.1 Module layout

| Module | Responsibility | Platforms |
|---|---|---|
| `:shared:core-model` | Domain models, IDs, value types | common |
| `:shared:data-agenda` | `AgendaSource` implementations, DTOs, normalizer, refresh/diff | common (Ktor engines per platform) |
| `:shared:data-local` | SQLDelight schema, DAOs, repositories (`AgendaRepository`, `FavoritesRepository`, `PrefsRepository`) | common + driver `actual`s |
| `:shared:domain` | Overlap engine, time-slot grouping, filter/search, use cases (`ToggleFavorite`, `ObserveDay`, `ObserveMySchedule`, `ReconcileReminders`) | common, pure Kotlin |
| `:shared:auth` | `AuthRepository` interface + Firebase implementation (GitLive), fake for tests | common + platform sign-in bridges |
| `:shared:reminders` | `ReminderScheduler` interface; Android (`AlarmManager` + `BroadcastReceiver`) and iOS (`UNUserNotificationCenter`) implementations | expect/actual |
| `:shared:sync` | `SyncRepository` interface, Firestore implementation, outbox processor, merge logic | common |
| `:shared:ui` | Compose Multiplatform screens, ViewModels (`androidx.lifecycle` KMP), navigation, theme, resources | common |
| `:androidApp` | Application, Activity, manifest, receivers, Credential Manager glue | Android |
| `:iosApp` | SwiftUI host for `ComposeUIViewController`, AppDelegate (Firebase init, notification delegate), Sign in with Apple / GoogleSignIn glue | iOS |

### 7.2 Key libraries
Kotlin 2.x · Compose Multiplatform · Coroutines/Flow · kotlinx.serialization · kotlinx-datetime · Ktor client · SQLDelight · Koin (DI) · androidx Lifecycle/ViewModel + Navigation (KMP) · Coil 3 (images) · GitLive `firebase-kotlin-sdk` (auth, firestore) · Kermit (logging) · Turbine, kotlin-test, Kotest assertions (tests).

### 7.3 Data flow

```mermaid
flowchart LR
  subgraph Sources
    B[Bundled JSON]:::src
    R[Remote agenda]:::src
  end
  B --> N[Normalizer]
  R --> N
  N --> DB[(SQLDelight)]
  DB -->|Flow| D[Domain: grouping + overlap]
  F[FavoritesRepository] --> DB
  D --> VM[ViewModels]
  VM --> UI[Compose MP UI]
  UI -->|toggle| F
  F --> RS[ReminderScheduler]
  F <--> S[SyncRepository]
  S <--> FS[(Firestore users/uid)]
  A[AuthRepository / Firebase Auth] --> F
  A --> S
  classDef src fill:#eef,stroke:#88a;
```

### 7.4 Reminder reconciliation
`ReconcileReminders` is the **only** code path that touches the platform scheduler. It computes the desired set `{sessionId → triggerAt}` from favorites + agenda + prefs, diffs it against `scheduled_reminder`, then schedules or cancels the differences. It runs after favorite toggles, agenda refresh, incoming sync, pref changes, boot/time-zone change (Android), and app foreground (iOS top-up). This keeps the behavior deterministic and testable with a fake scheduler.

---

## 8. UX outline

| Screen | Key elements | States |
|---|---|---|
| **Onboarding / Sign-in** | Value prop, Google / Apple / email-link buttons, privacy link | loading, error, offline ("connect once to sign in") |
| **Agenda** (bottom-nav tab 1) | Day tabs with conflict counts, time-slot sections, parallel-session grouping, session cards, filter chip row, search action, "Now" marker | loading, empty filter result, offline banner, stale-data notice |
| **Session Detail** | Title, time (with TZ label), room, speakers, description, tags, favorite FAB, reminder override, "overlaps with" list | removed/changed session notice |
| **My Schedule** (tab 2) | Favorites by day, conflict pairs side by side with resolve actions, free-time gaps, notification-permission banner | empty ("star sessions to build your plan"), conflicts |
| **Filter / Search** | Bottom sheet filters; search field with results by day | no results |
| **Settings** (tab 3) | Reminders, time display, sync toggle and status, account (sign out / delete), about | sync error, permission denied |

Navigation: bottom nav (Agenda · My Schedule · Settings), with deep links `adsplanner://session/{id}` from notifications.

---

## 9. Workstreams

Each workstream can be one epic / GitHub milestone. Sizes are rough (S ≈ 1–3 days, M ≈ 1 week, L ≈ 2 weeks for one engineer).

```mermaid
flowchart TD
  WS0[WS0 Foundation] --> WS1[WS1 Agenda data]
  WS0 --> WS2[WS2 Domain logic]
  WS0 --> WSA[WS-AUTH Authentication]
  WS0 --> WS3[WS3 UI]
  WS1 --> WS4[WS4 Favorites & persistence]
  WS2 --> WS3
  WSA --> WS4
  WS4 --> WS5[WS5 Reminders]
  WS4 --> WS6[WS6 Sync]
  WSA --> WS6
  WS3 --> WS7[WS7 Quality & release]
  WS5 --> WS7
  WS6 --> WS7
```

### WS0 — Foundation (blocks all) · Size M
- **Scope:** KMP project via the Kotlin Multiplatform wizard (Android + iOS, shared Compose UI); module skeleton from §7.1; Gradle version catalog; Koin setup; Kermit logging; detekt/ktlint; GitHub Actions CI (Android assemble + unit tests on Linux; iOS framework build + tests on macOS runner); Firebase projects (dev/prod) created; `README` dev setup.
- **Deliverables:** empty app runs on Android emulator and iOS simulator showing a placeholder screen; CI green.
- **Acceptance:** `./gradlew check` passes locally and in CI; iOS app builds from Xcode and CI.

### WS1 — Agenda data and ingestion · Size M–L
- **Scope:** FR-DATA-1 spike + ADR; `core-model`; DTOs; normalizer; `BundledAgendaSource` with a captured snapshot; remote source; SQLDelight agenda tables; `AgendaRepository` exposing `Flow<List<ConferenceDay>>`; refresh policy; diffing (FR-DATA-5).
- **Depends on:** WS0.
- **Acceptance:** fresh install with no network shows the full agenda from the bundle; a refresh with changed fixture data updates the DB and emits a diff; normalizer tests cover time zone, missing fields, and ID stability.

### WS2 — Domain logic · Size M
- **Scope:** overlap engine (FR-OV-1..3, 6), time-slot grouping, favorite-conflict computation, filters and search, use cases; all pure Kotlin with fakes.
- **Depends on:** WS0 (can start on fake data in parallel with WS1).
- **Acceptance:** property-based and table tests for overlap (touching boundaries, nested, identical, partial, multi-way); ≥ 90 % coverage of the module; benchmark < 5 ms for 100 sessions.

### WS3 — UI (Compose Multiplatform) · Size L
- **Scope:** theme/design system, navigation + deep links, Agenda, Session Detail, My Schedule, Filter/Search, Settings, Onboarding screens; ViewModels on top of WS2 use cases; empty/loading/error states; accessibility semantics.
- **Depends on:** WS0; builds against WS2 interfaces and fake repositories until WS1/WS4/WS-AUTH land.
- **Acceptance:** all screens in §8 render on both platforms with fake data; screenshot tests for Agenda (with conflicts), My Schedule, and Detail; TalkBack/VoiceOver walk-through passes.

### WS-AUTH — Authentication · Size M
- **Scope:** FR-AU-1..7; Firebase config files per platform/flavor; GitLive Auth; Android Credential Manager Google sign-in; iOS GoogleSignIn + Sign in with Apple bridges; `AuthRepository` (+ fake); auth gate in navigation; sign-out; delete account (including Firestore cleanup, coordinated with WS6).
- **Depends on:** WS0.
- **Acceptance:** sign in, kill app, relaunch offline → signed in with data available; sign-out then sign-in as another account shows separate favorites; delete account removes the Auth user and `users/{uid}`.

### WS4 — Favorites and persistence · Size S–M
- **Scope:** FR-FV-1..5; `favorite`, `user_prefs`, `acknowledged_conflict` tables scoped by uid; `FavoritesRepository` with Flow APIs; orphan/changed handling; outbox writes (consumed by WS6).
- **Depends on:** WS1 (schema), WS-AUTH (uid).
- **Acceptance:** toggling is reflected in < 100 ms; favorites survive refresh, restart, and upgrade; removed sessions show as orphaned.

### WS5 — Reminders · Size M
- **Scope:** FR-RM-1..9; `ReminderScheduler` expect/actual; Android alarms + receivers + boot/time-zone handling + exact-alarm permission flow + notification channel; iOS notification center, 64-cap rolling window, notification delegate deep link; `ReconcileReminders` use case; permission UX.
- **Depends on:** WS4.
- **Acceptance:** with fake clock tests, the desired set is correct for toggles, lead-time changes, and agenda time changes; on device, a reminder fires within tolerance with the screen off (Android Doze via `adb shell dumpsys deviceidle force-idle`) and after reboot; iOS reminders keep working with > 64 favorites.

### WS6 — Sync (optional feature) · Size M–L
- **Scope:** FR-SY-1..10; Firestore data layout and security rules; emulator-based rules tests; outbox processor; merge/LWW with tombstones; snapshot listener lifecycle; feature flag; sync status UI hook.
- **Depends on:** WS-AUTH, WS4.
- **Acceptance:** two devices/simulators on the same account converge within 5 s; offline edits on both devices merge correctly by LWW after reconnect; an unfavorite on device A removes it on B and cancels B's reminder; rules tests show cross-user access is denied.

### WS7 — Quality and release · Size M
- **Scope:** test plan execution, accessibility audit, performance profiling, crash reporting (opt-in), privacy policy, store listings, Play internal testing track, TestFlight, release checklist, and an event-week "agenda freeze" plan.
- **Depends on:** WS3, WS5 (and WS6 if shipped).
- **Acceptance:** NFR targets in §5 met; builds available to testers on both stores at least **2 weeks before the event**.

### Milestones

| Milestone | Contents | Exit criteria |
|---|---|---|
| **M1 — Browse** | WS0, WS1, WS2, WS3 (agenda screens), WS-AUTH | Signed-in user can browse every day offline with overlaps shown |
| **M2 — Plan** | WS4, WS5, WS3 (My Schedule, Settings) | Favorites, conflict view, and reliable reminders on both platforms |
| **M3 — Sync & ship** | WS6, WS7 | Optional sync behind a toggle; store builds released to testers |

Parallelism: once WS0 is done, **WS1, WS2, WS3, and WS-AUTH can run at the same time** (4 streams). WS4 → WS5 and WS6 can then split across two people.

---

## 10. Acceptance criteria (selected Given/When/Then)

**Overlap**
- *Given* session A 10:00–10:40 and B 10:40–11:20, *when* the day is shown, *then* A and B are **not** marked as overlapping.
- *Given* A 10:00–11:00 and B 10:30–11:30 both favorited, *then* both show a conflict badge, My Schedule shows "overlaps 30 min", and the day tab conflict count is 1.
- *Given* the user chooses "Keep both" on that conflict, *then* the badge disappears for that pair and comes back if either session's time changes.

**Reminders**
- *Given* lead time 10 min and a favorite starting 14:00 venue time, *then* a notification is scheduled for 13:50 venue time, whatever the device time zone.
- *Given* the agenda refresh moves that session to 14:30, *then* the reminder is moved to 14:20 and the user sees a "session time changed" notice.
- *Given* a favorite is removed, *then* its pending reminder is cancelled within the same transaction.
- *Given* notification permission is denied, *when* the user favorites a session, *then* the favorite is saved and My Schedule shows the permission banner.

**Auth**
- *Given* a signed-in user, *when* the app launches in airplane mode, *then* the agenda and favorites load with no sign-in prompt.
- *Given* user X signs out and user Y signs in on the same device, *then* Y sees none of X's favorites and X's reminders are cancelled.

**Sync**
- *Given* sync is on for devices A and B, both offline, *when* A favorites S at t1 and B unfavorites S at t2 > t1, *then* after both reconnect S is unfavorited on both.
- *Given* sync is off, *then* no Firestore reads or writes happen (verified via emulator request log).

---

## 11. Risks and open questions

| # | Risk / question | Impact | Mitigation / next step |
|---|---|---|---|
| R1 | **Agenda source format and terms of use are unknown** (the page could not be inspected while this PRD was written). It may be JS-rendered, change without notice, or forbid scraping. | High | WS1 spike first; bundled snapshot; optional self-hosted JSON mirror updated by a GitHub Action; manual JSON as a last resort. |
| R2 | Late agenda changes during the event week | Medium | Refresh policy + diff notices; reminders reconciled on every refresh. |
| R3 | Android exact-alarm restrictions (API 31+/34+) | Medium | `USE_EXACT_ALARM` is allowed for calendar/alarm-style apps; otherwise request `SCHEDULE_EXACT_ALARM` with an explanation; inexact fallback. |
| R4 | iOS 64 pending-notification cap | Low | Rolling window + top-up on foreground/background refresh. |
| R5 | Time zone and DST bugs (venue vs device TZ) | Medium | Store UTC; explicit TZ everywhere; dedicated tests. |
| R6 | Required login adds onboarding friction and store-review needs (Sign in with Apple, in-app account deletion) | Medium | Fast one-tap providers; FR-AU-6 delete flow; consider FR-AU-8 preview mode. |
| R7 | Sign-in needs network once; conference Wi‑Fi is unreliable | Medium | Encourage sign-in before the event; clear offline messaging. |
| R8 | GitLive Firebase SDK maturity vs native SDKs | Medium | Keep Firebase behind `AuthRepository`/`SyncRepository`; fall back to platform-native SDKs through expect/actual if needed. |
| R9 | Firebase cost and privacy | Low | Tiny data volume (favorites only); rules deny by default; privacy policy. |
| Q1 | Exact event dates, number of days, and venue time zone to confirm from the official site. | — | Confirm in WS1 spike. |
| Q2 | Is a read-only agenda preview before sign-in acceptable (FR-AU-8)? | — | Decide in design review. |
| Q3 | Should "Keep both" acknowledged conflicts sync? (Current assumption: yes.) | — | Confirm with owner. |
| Q4 | Desktop (JVM) target for development only, or shipped? | — | Default: dev only. |

---

## 12. Traceability: user request → requirements → workstreams

| User-requested feature | Requirements | Workstreams |
|---|---|---|
| Daily view of sessions | FR-AG-1..10, FR-DATA | WS1, WS2, WS3 |
| Identify overlapping sessions | FR-OV-1..7 | WS2, WS3 |
| Mark sessions as favorite | FR-FV-1..6 | WS4, WS3 |
| Reminder alert before a session | FR-RM-1..10 | WS5 |
| Optional cross-device sync | FR-SY-1..10 (+ FR-AU) | WS-AUTH, WS6 |
| Authentication via Firebase Auth | FR-AU-1..8 | WS-AUTH |
