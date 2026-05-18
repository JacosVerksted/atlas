# UI Slice 1 — Implementation Plan

**Spec:** `superpowers/specs/2026-05-15-ui-slice-1-design.md`
**Status:** PENDING
**Approved:** No
**Worktree:** No

## Task list

| # | Task | Files |
|---|---|---|
| 1 | Sidecar: add `DiskBytes` field to `state.Update`/`state.Snapshot` | `apo-control/internal/state/*` |
| 2 | Sidecar: `LogFollower.refreshDisk` helper + 30s throttle | `apo-control/internal/server/follower.go` + test |
| 3 | Sidecar: wire `refreshDisk` into the follower goroutine | `follower.go` |
| 4 | Rails: `Admin::RegionsController#update` broadcasts to `region_channel` | controller + spec |
| 5 | Rails: `HomeController#show` populates `@active_regions` + `@degraded` | controller + spec |
| 6 | Rails: service_card partial renders `last_log`, `last_error`, `disk_bytes` | partial + spec |
| 7 | Rails: admin index groups services by profile with section headers | index.html.erb + spec |
| 8 | Rails: `home/_region_pill.html.erb` + Turbo subscription | partial + view |
| 9 | Rails: `home/_degradation_banner.html.erb` + Turbo subscription | partial + view |
| 10 | JS: map controller reacts to hidden region-meta element changes (recenter) | `map_controller.js` |
| 11 | End-to-end smoke via Playwright | manual or sibling repo |

Tasks 1–3 are sidecar Go. Tasks 4–10 are Rails/JS. They're independent.

---

## Task 1 — DiskBytes on state.Update/Snapshot

**Files:**
- `apo-control/internal/state/state.go` (add field)
- existing state tests if any

Add `DiskBytes int64` to the `Update` struct and to the `Snapshot` struct (the wire format already has `disk_bytes`; confirm and add if missing). Update `Store.Update` to copy the field if non-zero (so a parser-only update doesn't zero out a previous disk reading).

**Steps:**
1. Write failing test in `state_test.go`: passing `Update{DiskBytes: 42}` then `Snapshot()` returns 42 in that field.
2. Run test → fails (no field).
3. Add field to struct, copy in Update.
4. Run test → passes.
5. Commit.

## Task 2 — refreshDisk helper

**Files:**
- `apo-control/internal/server/follower.go` (add helper + cache field)
- `apo-control/internal/server/follower_test.go` (new)

```go
type LogFollower struct {
    // ...existing fields
    diskCache map[string]diskCacheEntry
    diskMu    sync.Mutex
    dataDir   string
}

type diskCacheEntry struct {
    bytes int64
    at    time.Time
}

var diskDirFor = map[string]string{
    "photon":      "photon",
    "valhalla":    "valhalla",
    "overpass":    "overpass",
    "otp":         "otp",
    "placeholder": "placeholder",
}

func (f *LogFollower) refreshDisk(name string) int64 {
    dir, ok := diskDirFor[name]
    if !ok {
        return 0
    }
    f.diskMu.Lock()
    if e, ok := f.diskCache[name]; ok && time.Since(e.at) < 30*time.Second {
        f.diskMu.Unlock()
        return e.bytes
    }
    f.diskMu.Unlock()

    abs := filepath.Join(f.dataDir, dir)
    var total int64
    _ = filepath.WalkDir(abs, func(_ string, d fs.DirEntry, err error) error {
        if err != nil || d.IsDir() {
            return nil
        }
        if info, err := d.Info(); err == nil {
            total += info.Size()
        }
        return nil
    })

    f.diskMu.Lock()
    if f.diskCache == nil {
        f.diskCache = map[string]diskCacheEntry{}
    }
    f.diskCache[name] = diskCacheEntry{bytes: total, at: time.Now()}
    f.diskMu.Unlock()
    return total
}
```

Pass `cfg.DataDir` into `NewLogFollower`. Update constructor signature accordingly; update the two call sites (`NewWithStore` and any tests).

**Steps:**
1. Test: create temp dir tree with known sizes, assert `refreshDisk` returns total.
2. Test: second call within 1s returns cached value (control via monkeying with `time.Since` not needed — just assert same result without filesystem change).
3. Test: unknown service name returns 0.
4. Implement.
5. Commit.

## Task 3 — Wire into follower goroutine

After each parser `Feed(line)` in `LogFollower.run`, call `f.refreshDisk(name)` and include it in the `state.Update`:

```go
f.store.Update(name, state.Update{
    Phase:       r.Phase,
    Progress:    r.Progress,
    LastLogLine: r.LastLogLine,
    Ready:       r.Ready,
    DiskBytes:   f.refreshDisk(name),
})
```

**Steps:**
1. Test (integration): start follower with a fake log source emitting one line, fake parser, assert `store.Snapshot()` shows a non-zero `DiskBytes` for a service whose data dir has content.
2. Implement.
3. Run all sidecar tests → all pass.
4. Commit.

## Task 4 — Region broadcast

**Files:**
- `app/app/controllers/admin/regions_controller.rb` (add broadcast)
- `app/spec/controllers/admin/regions_controller_spec.rb`

After `RegionSelection.transaction { ... }` block, broadcast:

```ruby
Turbo::StreamsChannel.broadcast_replace_to(
  "region_channel",
  target: "region_meta",
  partial: "home/region_meta"
)
Turbo::StreamsChannel.broadcast_replace_to(
  "region_channel",
  target: "region_pill",
  partial: "home/region_pill"
)
```

Both partials need `@active_regions` + center coords. Move that derivation into a helper or pass locals.

Recommended: a `RegionPresenter` PORO with `.from_db` returning `{names:, label:, lat:, lon:, zoom:}`.

**Steps:**
1. Test: POST to `/admin/regions` with `regions: ["berlin"]` triggers two broadcasts to `region_channel` (stub `Turbo::StreamsChannel.broadcast_replace_to` and assert calls).
2. Implement presenter + broadcasts.
3. Commit.

## Task 5 — Home controller additions

**Files:**
- `app/app/controllers/home_controller.rb`
- `app/spec/controllers/home_controller_spec.rb`

Add:
```ruby
@active_regions = RegionSelection.where(active: true).order(:position).pluck(:region_name)
@degraded = Service.where(enabled: true)
                   .where(status: %w[error unhealthy stopped])
                   .pluck(:name)
```

Override `@default_lat/lon/zoom` from the first active region's catalog entry when present.

**Steps:**
1. Test: with no regions, `@active_regions == []` and `@degraded == []`.
2. Test: with Berlin active, `@active_regions == ["berlin"]` and lat/lon/zoom overridden.
3. Test: with overpass enabled+errored, `@degraded == ["overpass"]`.
4. Implement.
5. Commit.

## Task 6 — service_card partial

**Files:**
- `app/app/views/admin/services/_service_card.html.erb`
- `app/spec/views/admin/services/_service_card.html.erb_spec.rb`

Add three rows (all conditional):

```erb
<% if service.last_log.present? %>
  <div class="text-[10px] text-base-content/50 italic truncate font-mono"><%= service.last_log.truncate(80) %></div>
<% end %>

<% error_msg = service.last_error.presence ||
   (service.status.in?(%w[error unhealthy]) && service.last_log&.match?(/error|fail|denied/i) ? service.last_log : nil) %>
<% if error_msg %>
  <div class="text-xs text-error truncate"><%= error_msg.truncate(120) %></div>
<% end %>

<% if service.disk_bytes&.positive? %>
  <div class="text-xs text-base-content/60"><%= number_to_human_size(service.disk_bytes) %></div>
<% end %>
```

The `disk_bytes` cell goes in a small right-aligned column. Restructure card with `flex-col` content on the left and a small `text-right` column before the Enable/Disable button.

**Steps:**
1. Test: render with `last_log: "boot ok"`, assert "boot ok" present.
2. Test: render with `last_error: "kaput"`, assert "kaput" in red.
3. Test: render with `status: "error", last_log: "Failed to bind"`, assert "Failed to bind" present in error styling (fallback).
4. Test: render with `disk_bytes: 1_500_000`, assert "1.43 MB" or `number_to_human_size` output present.
5. Test: render minimal service (no logs/errors/disk) → no extra rows.
6. Implement.
7. Commit.

## Task 7 — Profile grouping

**Files:**
- `app/app/views/admin/services/index.html.erb`
- `app/spec/views/admin/services/index.html.erb_spec.rb`

Replace flat `@services.each` with grouped iteration:

```erb
<% profile_order = %w[geocoding routing pois transit] %>
<% grouped = @services.group_by(&:profile) %>
<% profile_order.each do |profile| %>
  <% list = grouped[profile] || [] %>
  <% next if list.empty? %>
  <h4 class="text-xs uppercase tracking-wide text-base-content/60 mt-2 first:mt-0"><%= profile.titleize %></h4>
  <div class="flex flex-col gap-2">
    <% list.sort_by(&:name).each do |service| %>
      <%= render "admin/services/service_card", service: service %>
    <% end %>
  </div>
<% end %>
```

**Steps:**
1. Test: with one service per profile, all four section headers render in expected order.
2. Test: empty profile (no services) → header not rendered.
3. Implement.
4. Commit.

## Task 8 — Region pill partial

**Files:**
- `app/app/views/home/_region_pill.html.erb` (new)
- `app/app/views/home/index.html.erb` (add include + meta)

```erb
<%# _region_pill.html.erb %>
<div id="region_pill" class="fixed bottom-3 left-3 z-20">
  <% if (names = (defined?(@active_regions) ? @active_regions : [])).any? %>
    <div class="badge badge-neutral gap-1 shadow">
      <span><%= names.size == 1 ? names.first.titleize : "#{names.size} regions" %></span>
      <a class="link link-hover text-xs" href="/admin/services?open=true">change</a>
    </div>
  <% end %>
</div>
```

In `home/index.html.erb`, before the map div:

```erb
<%= turbo_stream_from "region_channel" %>
<%= render "home/region_pill" %>
<%= render "home/region_meta" %>
```

## Task 9 — Degradation banner

**Files:**
- `app/app/views/home/_degradation_banner.html.erb` (new)
- `app/app/views/home/index.html.erb`

```erb
<%# _degradation_banner.html.erb %>
<%
  capability_for = { "photon" => "Search", "placeholder" => "Search",
                     "valhalla" => "Routing", "overpass" => "POIs", "otp" => "Transit" }
  caps = (defined?(@degraded) ? @degraded : []).filter_map { |n| capability_for[n] }.uniq
%>
<div id="degradation_banner" class="<%= caps.empty? ? "hidden" : "" %> ...alert classes...">
  <% if caps.any? %>
    <%= caps.join(" & ") %> unavailable —
    <a href="/admin/services?open=true" class="link">open settings</a>
  <% end %>
</div>
```

Subscribe via `<%= turbo_stream_from "services_channel" %>` (already wired in admin; safe to wire on home too).

When `PollStatusJob` broadcasts a service change, also broadcast the banner re-render. Simplest: extend `PollStatusJob#broadcast` to also call `broadcast_replace_to("services_channel", target: "degradation_banner", partial: "home/degradation_banner", locals: { degraded: derive_degraded })`. The locals derivation goes in the helper.

## Task 10 — Map controller MutationObserver

**Files:**
- `app/app/javascript/controllers/map_controller.js`
- `app/app/views/home/_region_meta.html.erb` (new)

```erb
<%# _region_meta.html.erb %>
<meta id="region_meta"
      data-lat="<%= defined?(@default_lat) ? @default_lat : 51.1657 %>"
      data-lon="<%= defined?(@default_lon) ? @default_lon : 10.4515 %>"
      data-zoom="<%= defined?(@default_zoom) ? @default_zoom : 2 %>">
```

In `map_controller.js`, in `connect()`:

```js
this.regionMeta = document.getElementById("region_meta")
if (this.regionMeta) {
  this.regionObserver = new MutationObserver(() => this.applyRegionCenter())
  this.regionObserver.observe(this.regionMeta, { attributes: true })
}

applyRegionCenter() {
  const lat = parseFloat(this.regionMeta.dataset.lat)
  const lon = parseFloat(this.regionMeta.dataset.lon)
  const zoom = parseFloat(this.regionMeta.dataset.zoom)
  if (this.map && Number.isFinite(lat) && Number.isFinite(lon)) {
    this.map.flyTo({ center: [lon, lat], zoom })
  }
}

disconnect() {
  this.regionObserver?.disconnect()
}
```

When Turbo replaces `#region_meta` with new `data-` attributes, the MutationObserver fires, the map re-centers.

## Task 11 — End-to-end smoke

Manual via Playwright, after rebuilding the app image:

1. Open `/` → no pill, no banner.
2. Hit `/admin/services?open=true`, click Berlin chip → return to `/` → pill shows "Berlin · change", map re-centered to Berlin.
3. Force overpass into error → banner shows "POIs unavailable".
4. Re-enable + ready → banner gone.
5. Photon card shows ~180 MB disk.

---

## Notes for the implementer

- TDD: red → green → commit per step. Don't bundle.
- Sidecar tests live in same package; Rails uses RSpec.
- No Capybara — view specs render partials directly with `render` helper.
- No `Co-Authored-By` trailers on commits (Eugene's standing rule).
- Plans/specs live under `superpowers/`, not under `app/docs/`.
- Use `~/.asdf/shims/bundle exec rspec ...` outside containers; inside the running app container `bin/rspec` is fine.
- After each Rails task, rebuild via `docker build -t apocalymaps-app:dev -f app/Dockerfile app` and `docker compose up -d --force-recreate app`. After each sidecar task, same with `apocalymaps-apo-control:dev`.
- `.env` already has `APP_IMAGE`/`APO_CONTROL_IMAGE` dev overrides locked in.
