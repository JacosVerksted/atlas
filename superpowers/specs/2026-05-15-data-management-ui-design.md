# Data Management UI — design

**Date:** 2026-05-15
**Status:** APPROVED (pending user review of this written spec)
**Owner:** Eugene
**Worktree:** No

## Goal

A self-hoster opens Apocalymaps, expands a foldable panel on the map page, picks one or more regions (e.g. Berlin + Vienna, or Germany + France), toggles which data services to enable (geocoding, routing, POIs, basemap), clicks Save → sees a size/time projection → confirms → watches per-service progress stream live → lands on a working stack scoped to those regions. All from the browser, no shell.

## Non-goals

- User-facing auth flow (HTTP Basic via env var is sufficient for now; full Rails 8 auth is on the roadmap, not in this spec)
- PMTiles self-building via Planetiler (the panel triggers existing `tiles-download` flow only)
- Per-tenant or multi-user concerns (single-operator self-hosted model)
- Replacing the existing CLI workflow — `make region`, `make geocoding`, etc. continue to work; the UI is additive

## Locked design decisions

| # | Decision | Choice |
|---|----------|--------|
| 1 | Control plane | **Sidecar service** with docker-socket access; Rails calls it over HTTP |
| 2 | Sidecar runtime | **Go** (single static binary, multi-arch image) |
| 3 | Progress detection | **Log parsing only** — one regex-based parser per service, no healthcheck polling, no disk-watching |
| 4 | Background model | **Sync sidecar calls + one recurring Solid Queue polling job** every 3 s |
| 5 | State persistence | **ActiveRecord** — `services` + `region_selections` tables |
| 6 | PBF plumbing | **Canonical path + shared mount** — `data/osm/current.osm.pbf`, bind-mounted into Valhalla/Overpass/OTP at `/osm:ro` |
| 7 | Auth gate | **HTTP Basic** via `ADMIN_USERNAME` / `ADMIN_PASSWORD` env vars |

## Architecture

```
Browser (admin)
   │  HTTP Basic
   ▼
Caddy (:8000)
   ▼
Rails app  ──────────────────────────────────────────────────┐
   ├─ /admin/services  (Hotwire panel — Turbo Frame in map page)
   ├─ POST /admin/services/:name/{enable,disable}
   ├─ POST /admin/regions  (replace selection)
   ├─ POST /admin/apply  (commit + start work, with two-phase confirm)
   ├─ Solid Queue recurring job — every 3 s, GET sidecar /status, diff vs AR, broadcast Turbo Stream
   ├─ AR: services, region_selections
   └─ Action Cable engine enabled in application.rb; Solid Cable wired in cable.yml
                                                              │
                                                              ▼
                                          apo-control sidecar (Go static binary)
                                             ├─ POST /actions/services/:name/{enable,disable}
                                             ├─ POST /actions/regions   body: {regions: [...]}
                                             ├─ POST /actions/tiles     body: {url}
                                             ├─ GET  /status            (full snapshot per service)
                                             ├─ GET  /logs/:name?lines=N
                                             ├─ Mounts: /var/run/docker.sock, ./data, ./compose.yml, ./regions/
                                             ├─ Shells out to `docker compose` and `osmium`
                                             └─ Per-service log parsers (Photon/Placeholder/Valhalla/Overpass/OTP/Whosonfirst)
                                                              │
                                                              ▼
                                          existing data services (unchanged)
                                          photon · placeholder · libpostal · valhalla · overpass · otp
```

Two **new containers** in compose: `apo-control` (Go sidecar) and a wider volume-mount story for Valhalla/Overpass/OTP (bind `./data/osm:/osm:ro`, services read `/osm/current.osm.pbf`). The sidecar source code lives at **`apo-control/`** at the repo root (sibling to `app/`).

One **Rails change** outside the panel itself: uncomment `require "action_cable/engine"` in `app/config/application.rb` and add `config/cable.yml` for Solid Cable.

## Components

### Sidecar (`apo-control`, Go)

Image: `ghcr.io/dawarich-app/apocalymaps-control` (built by parallel CI job).

| Endpoint | Behavior |
|----------|----------|
| `GET  /status` | Returns `[{name, container_state, phase, progress, last_log_line, ready, disk_bytes}]` for every known service |
| `GET  /logs/:name?lines=50` | Tail stdout of one service container |
| `POST /actions/services/:name/enable` | `docker compose --profile <profile> up -d <name>` |
| `POST /actions/services/:name/disable` | `docker compose stop <name>` |
| `POST /actions/regions` | Resolve regions → download PBFs into `data/osm/sources/` → merge or symlink to `data/osm/current.osm.pbf` → restart affected data services |
| `POST /actions/tiles` | Stream URL into `data/tiles/basemap.pmtiles` |

Internals:
- Phase parsers: one Go file per service, ~30 lines of regex + state machine, mapping log lines to `(phase, progress 0..1, ready)`.
- In-memory state; persisted only via Rails polling.
- `sync.Mutex` serializes region-apply actions (no concurrent merges).
- Config via env: `COMPOSE_FILE=/work/compose.yml`, `DATA_DIR=/work/data`, `REGIONS_DIR=/work/regions`, `LISTEN_ADDR=:8090`.

### Rails additions

| Layer | Pieces |
|-------|--------|
| Models | `Service` (name, profile, enabled, status, phase, progress, last_log, last_error, disk_bytes, updated_at), `RegionSelection` (region_name, active, position, orphaned) |
| Controllers | `Admin::ServicesController#update`, `Admin::RegionsController#update`, `Admin::ApplyController#create`, `Admin::BaseController` with `http_basic_authenticate_with` |
| Service objects | `ControlPlaneClient` (Faraday wrapper, returns typed exceptions), `RegionCatalog` (parses `regions/*.env` at boot using a small dotenv-style scanner — supports `KEY=value`, `KEY="quoted value"`, comments; no shell-expansion semantics), `ApplyProjection` (computes size/time estimate from selection) |
| Background | `ControlPlane::PollStatusJob` — registered in `config/recurring.yml` (Solid Queue's recurring-tasks config) with `every: "3s"` |
| Views | `admin/services/index.html.erb` rendered into the map page as a Turbo Frame; `_service_card.html.erb` partial broadcast individually for O(1) DOM swaps |
| Stimulus | `panel_controller.js` (open/close, persists state in `localStorage`), `regions_controller.js` (multi-select chips), `confirm_controller.js` (modal) |
| Migrations | `CreateServices`, `CreateRegionSelections` |
| Initializer | `config/initializers/admin_check.rb` — `after_initialize` block that flips a constant if `ADMIN_USERNAME`/`PASSWORD` missing |

### Region catalog

`RegionCatalog` is parsed once at boot (and on filesystem change in development, via a watcher). Each region exposes:

```ruby
{
  name: "berlin",
  label: "Berlin (city)",
  country_code: "de",
  pbf_url: "https://download.bbbike.org/.../Berlin.osm.pbf",  # OR pbf_urls (multi)
  default_view: { lat:, lon:, zoom: },
  expected_pbf_bytes: 31_000_000  # optional, used for ApplyProjection
}
```

New region presets dropped into `regions/` appear after a Rails reload.

### PBF plumbing

Sidecar manages `data/osm/`:

```
data/osm/
├── sources/                  # raw downloads (cached, reused on re-merge)
│   ├── germany-latest.osm.pbf
│   └── austria-latest.osm.pbf
├── current.osm.pbf           # canonical → symlink for single-region, real merged file for multi
└── current.json              # metadata: regions used, hash, timestamp
```

`compose.yml`: Valhalla, Overpass, OTP each gain a read-only mount of `./data/osm:/osm`. Their PBF env vars point at `/osm/current.osm.pbf`. On region change, sidecar atomically updates the canonical path, then `docker compose restart`s the affected services.

## Data flow

### A. Toggle a single service

1. User clicks Enable on Photon's card → Stimulus posts `POST /admin/services/photon { enabled: true }`.
2. `Admin::ServicesController#update` updates AR, calls `ControlPlaneClient.enable!("photon")` (sync).
3. Sidecar runs `docker compose --profile geocoding up -d photon`, starts following `docker logs photon`, returns 202.
4. `PollStatusJob` (next tick, ≤3 s later) fetches sidecar `/status`, sees new phase/progress, updates AR, broadcasts Turbo Stream.
5. Subscribed clients see the card transition in real-time.

### B. Apply region change (multi-region)

1. User checks Berlin + Vienna in the region panel, clicks Apply → Stimulus posts `POST /admin/apply` with selection.
2. `Admin::ApplyController#create` validates against catalog, computes projection, returns **409 + confirmation Turbo Frame** ("you'll download ~X GB, expect ~Y hours, OK?").
3. User confirms (POST with `confirmed=true`).
4. Inside a transaction: `RegionSelection` rows replaced; affected services flipped to "restarting" in AR.
5. `ControlPlaneClient.apply_regions(["berlin","vienna"])` (sync, 202).
6. Sidecar downloads each PBF into `data/osm/sources/` (skips cached); runs `osmium merge` → `data/osm/current.osm.pbf` atomically; writes metadata; restarts Valhalla/Overpass/OTP (only enabled ones).
7. `PollStatusJob` picks up new phases, broadcasts.

### C. Long-running download

- Photon download is ~5 GB; parser emits `phase=downloading, progress=0..1.0`.
- Every 3 s poll, Rails sees new progress, broadcasts Turbo Stream.
- DOM swap updates the progress bar.
- User can close tab; server-side state continues. On re-open, AR renders immediate snapshot + WebSocket resubscribes.

### D. Sidecar / Rails restart

- **Sidecar restart**: in-memory phase trackers lost. On restart, inspects each service's existing log output (`docker logs --tail=500`) to recompute phase. ≤5 s of stale UI.
- **Rails restart**: AR survives; `PollStatusJob` re-scheduled on boot.
- **Both restart**: sidecar reconstitutes from log tail; AR re-syncs within 3 s.

### E. Disable mid-build

- `docker compose stop` sends SIGTERM; service halts; `data/<service>/` has partial artifacts.
- Service AR: `enabled=false, status="stopped", phase="partial-build"`.
- Re-enabling: container restarts; image-level resume logic handles continuation (gisops/docker-valhalla does this; others vary).

## Error handling

| Failure | Behavior |
|---------|----------|
| Sidecar unreachable | `ControlPlane::Unavailable` raised; panel shows "sidecar unreachable" banner; toggles disabled; PollStatusJob logs + skips |
| Sidecar 5xx on action | Error envelope parsed; flashed onto the relevant card; AR `last_error` populated; user can retry |
| `docker compose` fails | Sidecar captures stderr; returns 502 + DOCKER_COMPOSE_FAILED + last 20 lines; rendered in collapsible "details" disclosure |
| PBF download fails | `.partial` file deleted; status=`error`, phase=`download-failed`; previous `current.osm.pbf` untouched |
| osmium merge fails | Same atomic pattern; previous merged file preserved |
| WebSocket disconnect | Panel falls back to 10 s polling of `/admin/services.json` (degraded mode banner); reconnect auto-resubscribes |
| Auth misconfigured | After-initializer detects missing env vars; admin routes return 503 with config instructions; rest of app boots cleanly |
| Concurrent region-apply | Sidecar `sync.Mutex` serializes; subsequent calls queued, return 202 with "queued" status; UI shows "in progress" badge |
| Disk full | Pre-check `df -B1 data/osm/` against `expected_pbf_bytes` from the region catalog (a follow-up populates these — see Open follow-ups); return 507 DISK_FULL if insufficient; during-download overflow surfaced via curl exit 23 |
| Region preset deleted | `Region::NotFound` raised; `Admin::*` returns 422 + banner listing available; orphaned `RegionSelection` rows flagged on next reconcile |
| Container unhealthy after ready | Sidecar polling of `docker compose ps --format json` notices `health: unhealthy`; phase=`unhealthy`; UI flips red |

## Testing

### Rails (RSpec)

- `spec/models/{service,region_selection}_spec.rb`
- `spec/services/{region_catalog,control_plane_client,apply_projection}_spec.rb`
- `spec/requests/admin/{services,regions,apply,auth}_spec.rb`
- `spec/jobs/control_plane/poll_status_job_spec.rb`
- `spec/integration/admin/*_spec.rb` (rswag → `swagger/admin/swagger.yaml` mounted at `/api-docs/admin`)

Faraday test adapter for sidecar HTTP; no real Docker.

### Sidecar (Go `testing`)

- `parser/<service>_test.go` — one per service, loads recorded log fixtures (`testdata/<service>-<phase>.log`), asserts parser output
- `handler/actions_test.go` — `httptest.NewServer`, mutex serialization, error envelopes
- `docker_test.go`, `osmium_test.go` — process invocation through an interface, mocked

### End-to-end (Playwright, in sibling `e2e/`)

- `e2e/admin-panel.spec.ts` — happy path: open panel, login, toggle Photon, watch transitions through mock sidecar
- `e2e/admin-panel-errors.spec.ts` — sidecar-unreachable banner, auth-misconfigured 503, mid-build disable

### Mock sidecar

`apo-control --mock=fixtures/scenario-N.yml` replays scripted timelines. Used by Rails request specs (via `spec/support/sidecar_helper.rb`) and Playwright e2e.

### NOT tested

- Real `docker compose` calls (would require Docker-in-Docker)
- Real osmium merges
- Hot-reload of `regions/*.env` at runtime (boot-time only is acceptable for now)

### CI

- New parallel job in `.github/workflows/build.yml`: `build-control-plane` → multi-arch image at `ghcr.io/dawarich-app/apocalymaps-control`
- New `test.yml` workflow: RSpec + Go tests on every push
- E2E workflow runs nightly (cost-aware)

## Open follow-ups (out of this spec)

- Full Rails 8 auth + Pundit (replaces HTTP Basic eventually)
- Hot-reload of region catalog on filesystem change
- Disk-space pre-check using accurate per-preset sizes (requires populating `expected_pbf_bytes` across all `regions/*.env`)
- Service heartbeat (sidecar emits "I'm alive" pings via WebSocket so Rails can show even sidecar-side staleness)
- Cost-aware download mode (rate-limit downloads when bandwidth is metered)
