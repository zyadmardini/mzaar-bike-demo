# [RESORT NAME] Trail App — Developer Specification (MVP)

**Audience:** Claude Code (and any human developer joining the build)
**Status:** Build-ready spec for the MVP
**Version:** 1.0

---

## 1. Project Overview

Build a cross-platform mobile app for a ski resort that is expanding into summer bike and hike trail operations. The MVP has three features:

1. Interactive trail map with difficulty levels.
2. Real-time open/closed trail status.
3. QR-code-based location finder.

The architecture must be **extensible** — the post-MVP roadmap (trail preview videos, lift status, push notifications, bookings, activity tracking, etc.) should be addable without restructuring core systems.

---

## 2. Recommended Tech Stack

| Layer | Choice | Rationale |
|---|---|---|
| **Mobile** | React Native (Expo) | Single codebase for iOS + Android. Mature ecosystem. Camera/QR/maps libraries all well-supported. |
| **Map rendering** | Mapbox GL Native (via `@rnmapbox/maps`) | Industry standard. Supports custom tiles, vector overlays, offline downloads. Trail polylines render cleanly. Alternative: MapLibre (open-source fork, free). |
| **Backend** | Supabase (PostgreSQL + Realtime + Auth + Storage) | Fast to set up. Realtime out of the box (critical for live status). Built-in auth for admin dashboard. Postgres gives us proper relational data for future features. |
| **Admin dashboard** | Next.js (React) + Tailwind + shadcn/ui | Web-based, mobile-friendly (crew may update status from phones). Auth piggybacks on Supabase. |
| **QR scanning** | `expo-camera` with built-in barcode scanner | No extra dependency. Works offline. |
| **Offline storage** | Mapbox offline regions + local SQLite (via `expo-sqlite`) for trail metadata cache | Trails work without signal. |
| **State management** | Zustand or React Context | Keep it simple. No Redux needed for MVP scope. |
| **Hosting (admin)** | Vercel | Trivial deploys for Next.js. |

If the client has constraints (e.g., must use Firebase, must self-host), swap the backend layer — the data model below is portable.

---

## 3. High-Level Architecture

```
┌─────────────────────┐      ┌──────────────────────┐
│  Mobile App (RN)    │      │  Admin Dashboard     │
│  iOS + Android      │      │  Next.js (web)       │
└──────────┬──────────┘      └──────────┬───────────┘
           │                            │
           │   HTTPS + Realtime (WSS)   │
           ▼                            ▼
       ┌──────────────────────────────────────┐
       │           Supabase                   │
       │  ┌────────────┐  ┌────────────────┐  │
       │  │ PostgreSQL │  │ Realtime (WSS) │  │
       │  └────────────┘  └────────────────┘  │
       │  ┌────────────┐  ┌────────────────┐  │
       │  │   Auth     │  │    Storage     │  │
       │  └────────────┘  └────────────────┘  │
       └──────────────────────────────────────┘
                          │
                          ▼
                ┌──────────────────┐
                │ Mapbox (tiles +  │
                │ custom style)    │
                └──────────────────┘
```

---

## 4. Data Model (PostgreSQL)

### `trails`
| Column | Type | Notes |
|---|---|---|
| id | uuid (PK) | |
| name | text | "Black Diamond Descent" |
| activity_type | enum | `bike` \| `hike` \| `both` |
| difficulty | enum | `green` \| `blue` \| `black` \| `double_black` |
| length_m | integer | meters |
| elevation_gain_m | integer | |
| elevation_loss_m | integer | |
| avg_duration_min | integer | average time to complete |
| description | text | |
| safety_notes | text | nullable |
| geometry | geography(LineString, 4326) | PostGIS polyline of the trail |
| status | enum | `open` \| `closed` \| `caution` \| `maintenance` |
| status_note | text | optional reason ("muddy after rain") |
| status_updated_at | timestamptz | |
| status_updated_by | uuid (FK → users.id) | |
| created_at | timestamptz | |
| updated_at | timestamptz | |

### `pois` (points of interest)
| Column | Type | Notes |
|---|---|---|
| id | uuid (PK) | |
| name | text | "Mid-Station Lodge" |
| type | enum | `lift_station` \| `lodge` \| `restroom` \| `water` \| `viewpoint` \| `first_aid` \| `parking` |
| location | geography(Point, 4326) | |
| description | text | nullable |
| icon | text | icon identifier for map rendering |

### `qr_locations` (the QR code system)
| Column | Type | Notes |
|---|---|---|
| id | uuid (PK) | the value encoded in the QR — short slug like `q-A12` |
| label | text | "Junction 12 — A-Line / Easy Out" |
| location | geography(Point, 4326) | exact coords of the sign |
| facing_direction | integer | degrees (0–359), nullable — what direction signs face |
| nearby_trail_ids | uuid[] | trails accessible from this point |
| nearby_poi_ids | uuid[] | POIs within ~100m |
| installed_at | timestamptz | |
| active | boolean | so we can disable codes without deleting |

### `users` (admin/staff only for MVP)
Standard Supabase auth. Single role for MVP: `staff`. Multi-role can be added later.

### `status_history` (audit log)
Logs every status change. Required for accountability.

| Column | Type |
|---|---|
| id | uuid (PK) |
| trail_id | uuid (FK) |
| previous_status | enum |
| new_status | enum |
| note | text |
| changed_by | uuid (FK) |
| changed_at | timestamptz |

### Tables reserved for post-MVP (create empty stubs so migrations stay clean)

- `trail_videos` — preview video clips (URL + thumbnail).
- `lifts` — lift status.
- `user_profiles` — guest accounts.
- `ride_logs` — activity tracking.

---

## 5. Feature Specifications

### 5.1 Interactive Trail Map

**Screen:** Main screen of the app — opens here on launch.

**Components:**
- Full-screen Mapbox view.
- Filter chips at top: `All` / `Bike` / `Hike`.
- Difficulty legend (collapsible).
- Floating "Scan QR" button (bottom-right).
- Floating "Recenter" button (bottom-right, above scan).

**Behavior:**
- On load, fetch all trails + POIs in one query. Cache locally.
- Render trails as polylines styled by difficulty:
  - Green: `#2ECC40`, weight 4
  - Blue: `#0074D9`, weight 4
  - Black: `#111111`, weight 5
  - Double black: `#111111`, weight 5, dashed pattern
- Closed trails: same color but with a red overlay or 50% opacity + dashed style.
- POIs rendered as icons (use a single sprite sheet).
- Tap a trail → bottom sheet slides up with trail details.
- Tap a POI → small popup with name and type.
- Pinch-to-zoom and rotation enabled.
- Map style: custom Mapbox style with terrain/contour lines visible — request a topographic base.

**Offline:**
- On first launch (or via a "Download for offline" button in settings), download the resort's map region as a Mapbox offline pack.
- Trail and POI data cached to SQLite, refreshed on app open when online.

**Detail panel (bottom sheet) contents:**
- Trail name (large)
- Difficulty badge
- Activity icons (bike, hike)
- Status badge (Open/Closed/Caution/Maintenance) — live
- Length, elevation gain/loss, average time
- Description
- Safety notes (if any)
- *(Post-MVP placeholder slot for: video preview button)*

---

### 5.2 Real-Time Open/Closed Status

**Mobile side:**
- Subscribe to Supabase Realtime on the `trails` table.
- On status change, update local cache and re-render any affected map elements + open detail sheets.
- A "Today's Status" tab in bottom nav shows a list view:
  - Grouped by status (Open first, then Caution, Closed, Maintenance).
  - Each item shows trail name, difficulty, status, note, and "Updated X min ago."
  - Pull-to-refresh fetches latest.
- Show a small "Last synced HH:MM" timestamp at the top.

**Admin dashboard side:**
- Auth-gated (Supabase Auth, email + password).
- Trails list view with inline status dropdowns — change status without leaving the page.
- Optional note field appears when changing status.
- Bulk select + bulk status change ("Close selected").
- Filter by activity type, difficulty, current status.
- Audit log view (read-only, shows `status_history`).

**Data flow:**
1. Staff changes status in dashboard → `UPDATE trails SET status, status_note, status_updated_at, status_updated_by`.
2. Trigger writes to `status_history`.
3. Supabase Realtime broadcasts the row change.
4. All subscribed mobile clients receive the update over WebSocket and re-render.

---

### 5.3 QR Code Location Finder

**Flow:**
1. User taps the floating "Scan QR" button on the map.
2. Camera view opens (full-screen, `expo-camera` with barcode scanner enabled, type `qr`).
3. On scan, decoded value (e.g., `q-A12`) is looked up:
   - First in local SQLite cache (so it works offline).
   - If not found locally and online, query Supabase.
   - If not found at all, show "Unknown QR code" with retry.
4. On success: dismiss camera, return to map, animate map to center on the QR's coordinates, drop a "You are here" pin.
5. Bottom sheet slides up with:
   - QR location label
   - Direction the sign faces (if known) — arrow or compass indicator
   - List of nearby trails (with status badges) — each tappable
   - List of nearby POIs

**QR code format:**
- Encode a short slug — `q-XXX` where XXX is a 2–4 character identifier (alphanumeric).
- This slug is the primary key in `qr_locations`.
- Signs should also include a human-readable label and the resort logo (handled in print design, not the app).

**Caching strategy:**
- On app launch (when online), fetch full `qr_locations` table to SQLite. It's small (likely <500 rows).
- This makes QR scans instantaneous and fully offline-capable.

**Admin dashboard side:**
- A "QR Codes" section to add/edit/disable QR locations.
- For each: enter slug, label, drop pin on map (or paste coordinates), associate nearby trails/POIs.
- Generate printable PDF of the QR code (use `qrcode` npm package server-side) — sized for the standard sign template, with label and slug visible underneath.

---

## 6. Reserved Architecture Hooks for Post-MVP

To avoid rebuilds when adding the roadmap features, scaffold the following now without implementing them:

- **`trail_videos` table** — already noted above. The trail detail bottom sheet should include a placeholder slot where a "Preview Trail" button will appear when a video exists for that trail.
- **Notifications service abstraction.** Even if we don't send any push notifications in MVP, set up Expo Notifications token registration so we can flip the feature on later without app-store re-approval.
- **Activity log table stub** (`ride_logs`) — no UI, but the schema is in place.
- **User accounts** — MVP is anonymous (no guest login required to use the map). But Supabase Auth is already configured for staff, so adding guest accounts later is a config change, not a rewrite.
- **Internationalization wrapper.** Use `i18next` from day one even if there's only one language. Wrap all UI strings. Cost: near zero. Benefit: huge later.

---

## 7. Project Structure (suggested)

```
/resort-trail-app
├── /apps
│   ├── /mobile          # Expo / React Native app
│   │   ├── /src
│   │   │   ├── /screens
│   │   │   │   ├── MapScreen.tsx
│   │   │   │   ├── TrailStatusScreen.tsx
│   │   │   │   ├── QRScanScreen.tsx
│   │   │   │   └── TrailDetailSheet.tsx
│   │   │   ├── /components
│   │   │   ├── /lib
│   │   │   │   ├── supabase.ts
│   │   │   │   ├── offline-cache.ts
│   │   │   │   └── qr-lookup.ts
│   │   │   ├── /store    # Zustand stores
│   │   │   └── /i18n
│   │   └── app.config.ts
│   └── /admin           # Next.js dashboard
│       ├── /app
│       │   ├── /trails
│       │   ├── /qr-codes
│       │   └── /audit-log
│       └── /lib
├── /packages
│   └── /shared          # shared types, enums, validation
└── /supabase
    ├── /migrations      # SQL migrations
    └── /seed            # seed data for dev
```

---

## 8. Build Order (recommended)

1. **Supabase project setup.** Create schema, enable PostGIS, enable Realtime on `trails`, set up auth.
2. **Admin dashboard skeleton.** Login + trails list + status update. This unblocks data entry.
3. **Seed data.** Load real trail polylines (probably from GPX files the resort provides) + POIs.
4. **Mobile app shell.** Navigation, Supabase client, basic map rendering.
5. **Trail rendering + filtering.** Difficulty colors, activity filter, detail sheet.
6. **Real-time status integration.** Subscribe, render badges, "Today's Status" list view.
7. **Offline support.** Mapbox offline region + SQLite cache.
8. **QR scanning + lookup.** Camera, decode, lookup, location pin.
9. **Admin: QR management.** CRUD + printable PDF.
10. **Polish, branding, app store assets, submission.**

---

## 9. Non-Functional Requirements

- **Performance:** Map must render trails in under 2 seconds on mid-range Android devices (2022+).
- **Offline-first:** Map, trail data, and QR lookup must all work with zero connectivity after first sync.
- **Battery:** No background location tracking. No always-on GPS. The QR-code approach is explicitly chosen to keep battery use minimal.
- **Accessibility:** Difficulty colors must also be distinguishable by shape/pattern (colorblind-safe). Use icons + text labels, not color alone.
- **Privacy:** No guest accounts in MVP = no PII collected. Admin accounts use Supabase RLS to enforce that only authenticated staff can mutate data.

---

## 10. Out of Scope (MVP)

Explicitly NOT building in this phase:

- Guest accounts, login, social features.
- Live GPS tracking, activity logging, leaderboards.
- Lift status, weather, webcams.
- E-commerce (tickets, rentals, lessons).
- Push notifications (scaffold only).
- Video previews (scaffold only).
- Native iPad / tablet layouts (phone-first).
- Multiple languages (i18n scaffold only).

These are deferred to later phases per the client proposal's roadmap.

---

## 11. Open Questions for Client

To resolve before development starts:

1. Final count and naming of trails. GPX or KML files of trail centerlines.
2. Brand assets: logo, colors, typography, app icon.
3. Resort base coordinates and bounding box (for the Mapbox offline region).
4. Number and placement strategy for QR signs (drives admin tooling priorities and print quantity).
5. Languages required at launch (English only? Plus Arabic/French?).
6. App store accounts: does the resort have an Apple Developer + Google Play account, or do we publish under our own?
7. Hosting: Supabase cloud (recommended) or self-hosted?
