# Mzaar Bike App — CLAUDE.md

## What This Is
A mobile trail app for a mountain resort expanding into summer mountain biking and hiking.
Three components:
- `apps/mobile` — Expo React Native (iOS + Android). Guest-facing product.
- `packages/shared` — TypeScript types, enums, and theme tokens.
- `data/` — Static JSON files (trail data, status, POIs, QR locations). Hosted on a CDN/GitHub Pages.

> **No Supabase. No real-time WebSockets. No admin server.**
> Trail status updates are delivered via pull-to-refresh polling against a static `status.json` file.

---

## Tech Stack (Do Not Deviate)
| Layer | Technology |
|---|---|
| Mobile | Expo SDK + React Native |
| Maps | `@rnmapbox/maps` (Mapbox GL Native) |
| Data source | Static JSON files (hosted CDN / GitHub Pages) |
| Status updates | Pull-to-refresh polling — no WebSockets |
| Offline | Mapbox offline packs + `expo-sqlite` |
| QR scanning | `expo-camera` barcode scanner |
| State | Zustand |
| Package manager | pnpm workspaces |

---

## Environment Variables
All in `apps/mobile/.env.local` (gitignored). See `.env.example`.

```
EXPO_PUBLIC_MAPBOX_TOKEN           # Mapbox GL access token
EXPO_PUBLIC_MAP_STYLE_URL          # default: mapbox://styles/mapbox/outdoors-v12
EXPO_PUBLIC_DATA_BASE_URL          # base URL for all JSON files e.g. https://host.com/data
EXPO_PUBLIC_RESORT_LAT             # map center latitude  (approx 33.93)
EXPO_PUBLIC_RESORT_LNG             # map center longitude (approx 35.85)
EXPO_PUBLIC_RESORT_BOUNDS          # JSON: [[swLng,swLat],[neLng,neLat]]
```

---

## Data Files (`/data/`)
Staff **only ever edit `status.json`**. The other files change when trails are added/renamed.

| File | Contents | Who Edits |
|---|---|---|
| `trails.json` | Trail list with geometry, difficulty, length, elevation, description | Developer (when trails change) |
| `status.json` | Trail status map + `updated_at` timestamp | **Staff** (via CMS or direct edit) |
| `pois.json` | Points of interest (lodges, lifts, restrooms, water, etc.) | Developer |
| `qr-locations.json` | QR slug → lat/lng + nearby trails/POIs | Developer (when signs are installed) |

### `status.json` shape
```json
{
  "updated_at": "2026-05-18T09:30:00Z",
  "trails": {
    "trail-001": { "status": "open", "note": "" },
    "trail-002": { "status": "closed", "note": "Muddy after rain" }
  }
}
```
Valid status values: `"open"` | `"closed"` | `"caution"` | `"maintenance"`

---

## Mobile App Architecture
**Entry:** `apps/mobile/App.tsx` — Mapbox token init, expo-font, SQLite init, i18n, initial data fetch

**Screens:**
- `MapScreen` — full-screen Mapbox map, filter chips, floating QR scan button
- `TrailStatusScreen` — SectionList grouped by status, pull-to-refresh, "Last synced" header
- `QRScanScreen` — expo-camera with reticle overlay
- `TrailDetailSheet` — bottom sheet (trail details, status chip)
- `QRResultSheet` — bottom sheet (QR location result, nearby trails/POIs)

**Key lib files:**
- `src/lib/mapbox.ts` — `MAP_STYLE_URL`, `RESORT_CENTER`, `RESORT_BOUNDS`, `downloadOfflineRegion()`
- `src/lib/data-fetcher.ts` — `fetchTrails()`, `fetchStatus()`, `fetchPOIs()`, `fetchQRLocations()` (simple fetch calls)
- `src/lib/offline-cache.ts` — SQLite read/write helpers; all four data types cached here
- `src/lib/qr-lookup.ts` — `lookupQR(slug)`: SQLite first → fallback to `qr-locations.json` if online

**Zustand store (`src/store/trailStore.ts`):**
- `trails[]`, `pois[]`, `statusMap` (keyed by trail ID), `lastSynced`
- `fetchAll()` — fetches all JSON files on launch
- `refreshStatus()` — fetches only `status.json` (pull-to-refresh)
- `activityFilter` — `"all" | "bike" | "hike"`

---

## Map Rendering Rules
- **4 `ShapeSource` + `LineLayer` groups** by difficulty — NOT one layer per trail (performance).
- Difficulty colors: green `#2ECC40`, blue `#0074D9`, black `#111111`, double-black `#111111` dashed.
- Closed trails: same color, 50% opacity, dashed overlay (driven by `statusMap`).
- POI `SymbolLayer` uses `maki` icon names during dev. Switch to custom sprite names with branded Mapbox style.
- `MAP_STYLE_URL` env var switches the entire map appearance — no code change needed to go from dev to branded style.

---

## Offline Strategy
1. On launch: if online → fetch all JSON files → write to SQLite.
2. If offline: read from SQLite for all data (trails, status, POIs, QR locations).
3. Mapbox offline region downloaded manually from Settings screen.
4. `lastSynced` timestamp stored in SQLite, shown in TrailStatusScreen.

## Status Refresh Strategy (No Real-Time)
- `refreshStatus()` called on: app launch, pull-to-refresh, app coming to foreground (`AppState` listener).
- Status freshness shown via `updated_at` field in `status.json` ("Updated X min ago" via `date-fns`).

---

## Staff Update Flow (No Admin Server)
Staff edit `status.json` via one of:
- **Tina CMS / Netlify CMS** — free headless CMS, web form UI, auto-commits to GitHub.
- **Direct GitHub edit** — edit the file in the GitHub UI, commit → auto-deploys via GitHub Pages/Netlify.
- **JSONbin.io** — hosted editable JSON store (zero-config fallback).

---

## Required External Accounts
| Service | Purpose | Cost |
|---|---|---|
| Mapbox | Map tiles + offline | Free (50k loads/month) |
| GitHub | Host `status.json` + source control | Free |
| Apple Developer | iOS submission | $99/year |
| Google Play | Android submission | $25 one-time |

---

## Build Commands
```bash
pnpm dev:mobile       # start Expo dev server
pnpm build:mobile     # production build via EAS
pnpm lint             # ESLint all workspaces
pnpm typecheck        # tsc --noEmit all packages
```

---

## Placeholder Assets (Check If Real Assets Have Arrived)
| Placeholder | What to replace with |
|---|---|
| `MAP_STYLE_URL = outdoors-v12` | Custom Mapbox Studio style URL |
| Straight-line geometry in `trails.json` | Real GeoJSON converted from client GPX/KML files |
| `apps/mobile/assets/icon.png` = Expo default | Real app icon (1024×1024 PNG) |
| `apps/mobile/assets/logo.png` = placeholder SVG | Real logo (SVG preferred) |
| `RESORT_BOUNDS` = approximate | Exact coordinates from on-site survey |
| `maki` POI icon names | Custom Mapbox sprite icon names (when branded style is ready) |

---

## Post-MVP Stubs (Intentionally Incomplete — Do Not Remove)
- Video preview slot in `TrailDetailSheet.tsx`
- Expo Notifications token registration scaffold in `App.tsx`
- i18n wrappers on all strings (even if only English is active — wrap with `t()` always)
