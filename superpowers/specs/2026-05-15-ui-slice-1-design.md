# UI Slice 1 — Map awareness + admin card details + profile grouping

**Status:** PENDING
**Approved:** No
**Worktree:** No

## Goal

Make the existing admin panel honest about errors and disk usage, group services by what they do, and give the map screen enough awareness that an end-user knows what data is loaded and when something has broken.

Three independent UI changes shipped as one slice because they share the same Turbo Stream + service model wiring.

## In scope

1. **Map: active region pill + auto re-center + degradation banner**
2. **Admin: per-card `last_error`, `last_log`, `disk_bytes` rendering**
3. **Admin: profile-based section grouping**

## Out of scope

- WOF data-setup trigger (slice 2)
- Basemap section (slice 2)
- Active apply progress ribbon (slice 2)
- Per-service detail/expand with log viewer (slice 3)
- Custom region creation, apply history (later)

---

## Part 1 — Map screen

### Active region pill

A small pill in the bottom-left corner of the map page, e.g. `Berlin · change`.

Source: `RegionSelection.where(active: true)`. If multiple, show `N regions · change` with the names in a tooltip. If none, hide the pill.

The `change` link points at `/admin/services?open=true`.

Implementation:
- `HomeController#show` already exists. Add `@active_regions = RegionSelection.where(active: true).order(:position).pluck(:region_name)`.
- New partial `home/_region_pill.html.erb` rendered inside the map root div.
- Subscribes to a new Turbo Stream channel `region_channel` so the pill updates without reload when admin changes the selection.

### Auto re-center

When the active region changes, the map's `data-map-lat-value` / `lon-value` / `zoom-value` need to update.

Approach: the map Stimulus controller exposes a `recenter(lat, lon, zoom)` method. A new `region` Stimulus controller listens for the `region:changed` custom event (dispatched from `Admin::RegionsController#update` via a Turbo Stream `append` that includes a `<script>` dispatching the event — or simpler: re-render `<turbo-frame id="region_pill">` with new data attributes and the controller diffs them).

Recommended: emit a tiny `_region_pill_update.turbo_stream.erb` after the region update, replacing both the pill DOM and a hidden `<meta data-region-lat=… data-region-lon=… data-region-zoom=…>` element that the map controller observes via `MutationObserver`. No event plumbing; pure DOM contract.

`RegionCatalog#find(name)` already exposes `default_lat/lon/zoom`. If multiple regions are active, use the **first** one's center for re-centering.

### Degradation banner

A thin top banner shown only when at least one **enabled** service is in `status ∈ {error, unhealthy, stopped}`. Hidden otherwise.

Format: `Routing unavailable — open settings` with a "×" to dismiss for the session.

Mapping service → user-facing capability:
- `photon`, `placeholder` → "Search"
- `valhalla` → "Routing"
- `overpass` → "POIs"
- `otp` → "Transit"
- `libpostal` → suppressed (internal, not user-visible)

When multiple capabilities are broken, combine: `Search & Routing unavailable`.

Implementation:
- `HomeController#show` adds `@degraded = Service.where(enabled: true).where(status: %w[error unhealthy stopped]).pluck(:name)`.
- New partial `home/_degradation_banner.html.erb`, rendered inside the map root.
- Subscribes to `services_channel` (already broadcast by `PollStatusJob`). The banner re-renders on any service change. Cheap, fewer than 10 services.
- Dismissal is `sessionStorage`-backed; re-shown next page load if still degraded.

---

## Part 2 — Admin per-service card

Update `admin/services/_service_card.html.erb`. Three additions, all behind `if present?` so the card doesn't grow when there's nothing to say.

```
<%= service.name %>                          [Enable/Disable]
<status> — <phase>                            (existing)
<progress bar if progress > 0 and not ready>  (existing)
<last_log truncated to 1 line, muted>         NEW — small, gray, italic
<last_error in red>                           NEW — only when present, red
<disk: X.X MB / GB>                           NEW — right-aligned next to status
```

### last_log line

Already on the model. Render with `truncate(80)`, monospace, `text-base-content/50`. Hidden when blank.

### last_error line

New behavior. Two sources:
1. If `Service.last_error` is set, use it directly.
2. Otherwise, if `service.status` ∈ {error, unhealthy} and `service.last_log` looks like an error (contains "error", "fail", "denied", case-insensitive), surface `last_log` as the error fallback.

This keeps things working today — overpass with "Failed to process planet file" surfaces immediately — and gives parsers a future hook to set `last_error` explicitly.

### disk_bytes

Populated by the **sidecar's log follower**, not the parsers. After each parser update, the follower calls a new helper:

```go
func (f *LogFollower) refreshDisk(name string) int64 {
    dir, ok := diskDirFor[name]
    if !ok { return 0 }
    abs := filepath.Join(f.cfg.DataDir, dir)
    // os.Stat walk — bounded by the directory's actual contents.
    var total int64
    filepath.WalkDir(abs, func(_ string, d fs.DirEntry, err error) error {
        if err != nil || d.IsDir() { return nil }
        if info, err := d.Info(); err == nil { total += info.Size() }
        return nil
    })
    return total
}
```

`diskDirFor` map: photon→"photon", valhalla→"valhalla", overpass→"overpass", otp→"otp", placeholder→"placeholder". libpostal omitted (in-memory, no dir).

Throttle: walk at most once every 30s per service (cache last walk timestamp in `LogFollower`). Walks for sub-GB directories are cheap; bigger ones (Berlin valhalla tiles ~1 GB) take <100ms but we don't want every log line triggering a walk.

`state.Update` gets a new `DiskBytes` field. Sidecar `/status` already returns `disk_bytes` — confirmed wired through. The Rails `PollStatusJob#sync` already sets `disk_bytes`. Two-side change is just populating it on the sidecar.

Display in card: format with `number_to_human_size`. Hidden when zero.

---

## Part 3 — Profile grouping

Group `@services` in `admin/services/index.html.erb` by `Service#profile`. Render section headers in this order:

| Profile      | Header label    | Services                       |
|--------------|----------------|--------------------------------|
| `geocoding`  | "Geocoding"    | photon, placeholder, libpostal |
| `routing`    | "Routing"      | valhalla                       |
| `pois`       | "POIs"         | overpass                       |
| `transit`    | "Transit"      | otp                            |

(Basemap section comes in slice 2.)

Implementation:
- Replace the existing flat `@services.each` loop with `@services.group_by(&:profile)`, iterating in a fixed profile order.
- Section header: `<h4 class="text-xs uppercase tracking-wide text-base-content/60 mt-2 first:mt-0">Geocoding</h4>`.
- Within a section, sort by service name for stable ordering.

The `services_channel` broadcast doesn't change — `broadcast_replace_to` targets `service_<name>` IDs, which still exist inside their new section parents.

---

## Data model

**No migrations.** All fields used already exist on `Service`:

- `status`, `phase`, `progress`, `last_log`, `last_error`, `disk_bytes`, `profile`, `enabled`, `name`, `last_seen_at`

`RegionSelection` already has `region_name`, `active`, `position`.

## Turbo Streams

| Channel            | Trigger                                             | Target              |
|--------------------|-----------------------------------------------------|---------------------|
| `services_channel` | `PollStatusJob` per-service change (existing)       | `#service_<name>`   |
| `region_channel`   | `Admin::RegionsController#update` after each change | `#region_pill`, hidden region meta |

`region_channel` is new. Broadcasting needs an `after_save` on `RegionSelection` (or an explicit `Turbo::StreamsChannel.broadcast_replace_to` call from the controller). Prefer the controller — keeps callbacks out of the model.

## Sidecar changes

One file: `apo-control/internal/server/follower.go` plus a helper.

- Add `DiskBytes` to `state.Update` and `state.Snapshot`.
- Helper as above; throttled per-service.
- Run after each parser `Feed()`, calling `f.store.Update(name, state.Update{..., DiskBytes: disk})` when changed.

`/status` JSON already includes `disk_bytes` (it's part of `state.Snapshot`'s wire form). Verify nothing's stripping it.

## Tests

RSpec — existing patterns:
- `home_controller_spec.rb` — `@active_regions`, `@degraded` populated correctly given fixtures.
- `service_card_partial_spec.rb` — render the partial with combinations of `last_error`/`last_log`/`disk_bytes` and assert presence/absence of each.
- `regions_controller_spec.rb` — assert Turbo Stream broadcast on update.

Go — `apo-control/internal/server/follower_test.go`:
- `refreshDisk` returns correct total for a known fixture tree.
- Throttling: two calls within 30s reuse the cached value.

Playwright e2e (sibling repo) — out of scope here; Eugene can add a smoke test that loads the map page, asserts the pill renders with "Berlin · change", and disables a service to see the banner appear.

## Risk / open questions

- **Disk walk on big trees** (planet valhalla tiles ~70 GB): the walk could take 1–2 seconds. Mitigation: throttle is already 30s, plus the walk runs in the follower goroutine, not the request hot path.
- **Map controller listening for region changes**: the MutationObserver approach is unusual but avoids an event bus. If it feels fragile during implementation, fall back to a tiny custom event dispatched from the Turbo Stream update.
- **Multiple active regions and re-center**: using only the first region's center may surprise users who selected DACH. Acceptable for v1; later we could compute the bounding box of all active regions.

## Acceptance

After implementation:
1. Load `/` with no region selected → no pill, no banner, map at world view.
2. Select Berlin in admin → pill appears as "Berlin · change", map re-centers to Berlin, no reload.
3. Click "Disable" on overpass while overpass card shows `status=error` → admin card shows red error line; map shows "POIs unavailable" banner.
4. Re-enable overpass, wait for ready → banner disappears, card clears the error.
5. Admin panel renders Geocoding / Routing / POIs / Transit sections with services correctly grouped.
6. Photon card shows disk usage like "180 MB" once the German extract is on disk.
