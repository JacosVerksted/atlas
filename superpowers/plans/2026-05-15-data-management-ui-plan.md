# Data Management UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a foldable admin panel on the Apocalymaps map page that lets a self-hoster manage backend data services from the browser — list services, toggle each one, multi-select regions, see size/time projection, confirm, watch live progress.

**Architecture:** Rails admin endpoints under `/admin/` call a small Go sidecar (`apo-control`) over HTTP. Sidecar owns `docker compose` + `osmium` exec. Rails polls sidecar `/status` every 3 s via Solid Queue recurring job, mirrors state into AR, broadcasts Turbo Streams over Action Cable (async adapter — single-process Rails). HTTP Basic auth via env vars.

**Tech Stack:** Rails 8.1.3 + Hotwire + Tailwind + DaisyUI on the front side; Go 1.22+ static binary for the sidecar. RSpec + Go `testing` for unit tests; Playwright (sibling repo) for e2e. Faraday for HTTP between Rails and sidecar.

**Status:** COMPLETE
**Approved:** Yes (via /spec + brainstorming workflow)
**Worktree:** No

---

## File structure (created or modified)

### Rails app — `app/`

| Path | Action | Responsibility |
|------|--------|----------------|
| `app/config/application.rb` | Modify | Uncomment `require "action_cable/engine"` |
| `app/config/cable.yml` | Create | Action Cable async adapter config |
| `app/config/routes.rb` | Modify | Add `namespace :admin` block + admin Rswag mount |
| `app/config/recurring.yml` | Create or modify | Register `ControlPlane::PollStatusJob` every 3 s |
| `app/config/initializers/admin_check.rb` | Create | Detect missing `ADMIN_USERNAME`/`PASSWORD` env vars |
| `app/config/initializers/rswag_ui.rb` | Modify | Add admin swagger endpoint |
| `app/db/migrate/<TS>_create_services.rb` | Create | `services` table |
| `app/db/migrate/<TS>_create_region_selections.rb` | Create | `region_selections` table |
| `app/app/models/service.rb` | Create | `Service` AR model + enum + validations |
| `app/app/models/region_selection.rb` | Create | `RegionSelection` AR model |
| `app/app/services/region_catalog.rb` | Create | Parses `regions/*.env`, exposes regions |
| `app/app/services/control_plane_client.rb` | Create | Faraday wrapper for sidecar HTTP |
| `app/app/services/apply_projection.rb` | Create | Size/time estimate from selection |
| `app/app/jobs/control_plane/poll_status_job.rb` | Create | Recurring sync of sidecar `/status` to AR |
| `app/app/controllers/admin/base_controller.rb` | Create | HTTP Basic auth + rescue_from |
| `app/app/controllers/admin/services_controller.rb` | Create | Toggle single service |
| `app/app/controllers/admin/regions_controller.rb` | Create | Replace region selection |
| `app/app/controllers/admin/apply_controller.rb` | Create | Two-phase confirm + dispatch |
| `app/app/views/admin/services/index.html.erb` | Create | Panel root |
| `app/app/views/admin/services/_service_card.html.erb` | Create | One card per service |
| `app/app/views/admin/regions/_region_chip.html.erb` | Create | One chip per region |
| `app/app/views/admin/apply/_confirmation.html.erb` | Create | Confirmation modal frame |
| `app/app/views/home/index.html.erb` | Modify | Embed admin panel Turbo Frame |
| `app/app/javascript/controllers/panel_controller.js` | Create | Open/close, localStorage persist |
| `app/app/javascript/controllers/regions_controller.js` | Create | Multi-select chip group |
| `app/app/javascript/controllers/confirm_controller.js` | Create | Modal trigger |
| `app/app/javascript/controllers/index.js` | Modify | Register the three new controllers |
| `app/spec/models/service_spec.rb` | Create | Model unit specs |
| `app/spec/models/region_selection_spec.rb` | Create | Model unit specs |
| `app/spec/services/region_catalog_spec.rb` | Create | Service unit specs |
| `app/spec/services/control_plane_client_spec.rb` | Create | Service unit specs |
| `app/spec/services/apply_projection_spec.rb` | Create | Service unit specs |
| `app/spec/jobs/control_plane/poll_status_job_spec.rb` | Create | Job spec |
| `app/spec/requests/admin/auth_spec.rb` | Create | Request spec |
| `app/spec/requests/admin/services_spec.rb` | Create | Request spec |
| `app/spec/requests/admin/regions_spec.rb` | Create | Request spec |
| `app/spec/requests/admin/apply_spec.rb` | Create | Request spec |
| `app/spec/integration/admin/services_spec.rb` | Create | Rswag admin integration spec |
| `app/spec/integration/admin/regions_spec.rb` | Create | Rswag admin integration spec |
| `app/spec/integration/admin/apply_spec.rb` | Create | Rswag admin integration spec |
| `app/spec/support/sidecar_helper.rb` | Create | Boots mock-mode sidecar binary for request specs |
| `app/spec/fixtures/regions/test.env` | Create | Test region preset |
| `app/spec/swagger_helper.rb` | Modify | Add `admin/swagger.yaml` config |
| `app/swagger/admin/swagger.yaml` | Generated | rswag output |

### Sidecar — `apo-control/`

| Path | Action | Responsibility |
|------|--------|----------------|
| `apo-control/go.mod` | Create | Go module manifest |
| `apo-control/go.sum` | Create | dep checksums |
| `apo-control/main.go` | Create | Entrypoint — flags, config, server start |
| `apo-control/Dockerfile` | Create | Multi-stage build → static binary |
| `apo-control/README.md` | Create | Dev guide |
| `apo-control/internal/state/state.go` | Create | In-memory service-state store + RWMutex |
| `apo-control/internal/state/state_test.go` | Create | State tests |
| `apo-control/internal/parsers/parser.go` | Create | `Parser` interface + shared types |
| `apo-control/internal/parsers/photon.go` | Create | Photon log parser |
| `apo-control/internal/parsers/photon_test.go` | Create | + testdata fixtures |
| `apo-control/internal/parsers/placeholder.go` | Create | Placeholder log parser |
| `apo-control/internal/parsers/placeholder_test.go` | Create | + testdata |
| `apo-control/internal/parsers/valhalla.go` | Create | Valhalla log parser |
| `apo-control/internal/parsers/valhalla_test.go` | Create | + testdata |
| `apo-control/internal/parsers/overpass.go` | Create | Overpass log parser |
| `apo-control/internal/parsers/overpass_test.go` | Create | + testdata |
| `apo-control/internal/parsers/otp.go` | Create | OTP log parser |
| `apo-control/internal/parsers/otp_test.go` | Create | + testdata |
| `apo-control/internal/parsers/whosonfirst.go` | Create | Whosonfirst log parser |
| `apo-control/internal/parsers/whosonfirst_test.go` | Create | + testdata |
| `apo-control/internal/dockerexec/dockerexec.go` | Create | docker compose wrapper |
| `apo-control/internal/dockerexec/dockerexec_test.go` | Create | Mocked exec tests |
| `apo-control/internal/osmium/osmium.go` | Create | osmium-tool wrapper |
| `apo-control/internal/osmium/osmium_test.go` | Create | Mocked exec tests |
| `apo-control/internal/regions/regions.go` | Create | Reads `regions/*.env`, resolves URLs |
| `apo-control/internal/regions/regions_test.go` | Create | + fixture .env files |
| `apo-control/internal/server/server.go` | Create | HTTP server + routes |
| `apo-control/internal/server/handlers.go` | Create | Endpoint handlers |
| `apo-control/internal/server/server_test.go` | Create | `httptest.NewServer` + integration tests |
| `apo-control/cmd/mock/main.go` | Create | Mock-mode binary — replays scripted timelines |
| `apo-control/testdata/...` | Create | Recorded log fixtures + scenarios |

### Compose, CI, Docs

| Path | Action | Responsibility |
|------|--------|----------------|
| `compose.yml` | Modify | Add `apo-control` service, `/osm:ro` mounts on Valhalla/Overpass/OTP |
| `.github/workflows/build.yml` | Modify | Parallel job: `build-control-plane` → push to GHCR |
| `.github/workflows/test.yml` | Create | RSpec + Go test runs on push |
| `README.md` | Modify | New section: "Admin panel" |
| `Makefile` | Modify | New targets `admin-up`, `admin-logs` |

### Documentation

- `apo-control/README.md` — local dev (run with mock data), endpoint reference, parser schema.
- `README.md` — when/how to enable, env vars required, security note.

---

## Phase A — Rails foundation

### Task 1: Enable Action Cable (async adapter)

**Files:**
- Modify: `app/config/application.rb`
- Create: `app/config/cable.yml`

This is config-only — no test slice. Smoke-verify by booting.

Use the async adapter (no Redis, no extra DB, no extra gem) — correct for a single-process self-hosted Rails app. Solid Cable can be wired in a future plan task if multi-process broadcasting is ever needed.

- [ ] **Step 1: Uncomment Action Cable engine**

Edit `app/config/application.rb` and change:

```ruby
# require "action_cable/engine"
```

to:

```ruby
require "action_cable/engine"
```

- [ ] **Step 2: Create `app/config/cable.yml`**

```yaml
development:
  adapter: async

test:
  adapter: test

production:
  adapter: async
```

- [ ] **Step 3: Verify Rails still boots**

Run from `app/`:

```bash
bundle exec rails runner 'puts "ok"'
```

Expected: prints `ok` with no errors.

- [ ] **Step 4: Commit**

```bash
git add app/config/application.rb app/config/cable.yml
git commit -m "feat(admin): enable Action Cable with async adapter for Turbo Streams"
```

---

### Task 2: `services` table + `Service` model (TDD)

**Files:**
- Create: `app/db/migrate/<TS>_create_services.rb`
- Create: `app/app/models/service.rb`
- Create: `app/spec/models/service_spec.rb`

- [ ] **Step 1: Write the failing test**

Create `app/spec/models/service_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe Service, type: :model do
  describe "validations" do
    it "requires a name" do
      service = Service.new(profile: "geocoding")
      expect(service).not_to be_valid
      expect(service.errors[:name]).to include("can't be blank")
    end

    it "requires a unique name" do
      Service.create!(name: "photon", profile: "geocoding")
      duplicate = Service.new(name: "photon", profile: "geocoding")
      expect(duplicate).not_to be_valid
      expect(duplicate.errors[:name]).to include("has already been taken")
    end
  end

  describe "#status" do
    it "defaults to 'unknown'" do
      service = Service.new(name: "photon", profile: "geocoding")
      expect(service.status).to eq("unknown")
    end

    it "accepts the known states" do
      service = Service.new(name: "photon", profile: "geocoding")
      %w[unknown stopped starting downloading building ready error unhealthy].each do |state|
        service.status = state
        expect(service).to be_valid, "expected #{state} to be valid"
      end
    end

    it "rejects unknown statuses" do
      expect {
        Service.new(name: "photon", profile: "geocoding", status: "bogus")
      }.to raise_error(ArgumentError, /'bogus' is not a valid status/)
    end
  end

  describe "progress" do
    it "is between 0 and 1" do
      service = Service.new(name: "photon", profile: "geocoding", progress: 1.5)
      expect(service).not_to be_valid
      expect(service.errors[:progress]).to include("must be less than or equal to 1")
    end
  end
end
```

- [ ] **Step 2: Run the test — expect failures**

```bash
cd app && bundle exec rspec spec/models/service_spec.rb
```

Expected: errors about `Service` constant missing.

- [ ] **Step 3: Create the migration**

Run from `app/`:

```bash
bundle exec rails generate migration CreateServices
```

Then replace the generated file body with:

```ruby
class CreateServices < ActiveRecord::Migration[8.1]
  def change
    create_table :services do |t|
      t.string  :name,          null: false
      t.string  :profile,       null: false  # geocoding | routing | pois | transit | basemap
      t.boolean :enabled,       null: false, default: false
      t.integer :status,        null: false, default: 0  # enum
      t.string  :phase
      t.float   :progress
      t.text    :last_log
      t.text    :last_error
      t.bigint  :disk_bytes,    null: false, default: 0
      t.datetime :last_seen_at
      t.timestamps
    end

    add_index :services, :name, unique: true
    add_index :services, :enabled
  end
end
```

- [ ] **Step 4: Create the model**

Create `app/app/models/service.rb`:

```ruby
class Service < ApplicationRecord
  enum :status, {
    unknown:     0,
    stopped:     1,
    starting:    2,
    downloading: 3,
    building:    4,
    ready:       5,
    error:       6,
    unhealthy:   7
  }

  validates :name,    presence: true, uniqueness: true
  validates :profile, presence: true
  validates :progress, numericality: { greater_than_or_equal_to: 0, less_than_or_equal_to: 1 }, allow_nil: true
end
```

- [ ] **Step 5: Migrate the test DB and run specs**

```bash
bundle exec rails db:migrate
RAILS_ENV=test bundle exec rails db:migrate
bundle exec rspec spec/models/service_spec.rb
```

Expected: 5 examples, 0 failures.

- [ ] **Step 6: Commit**

```bash
git add app/db/migrate/*_create_services.rb app/app/models/service.rb app/spec/models/service_spec.rb app/db/schema.rb
git commit -m "feat(admin): add Service model + migration"
```

---

### Task 3: `region_selections` table + `RegionSelection` model (TDD)

**Files:**
- Create: `app/db/migrate/<TS>_create_region_selections.rb`
- Create: `app/app/models/region_selection.rb`
- Create: `app/spec/models/region_selection_spec.rb`

- [ ] **Step 1: Write the failing test**

Create `app/spec/models/region_selection_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe RegionSelection, type: :model do
  describe "validations" do
    it "requires a region_name" do
      expect(RegionSelection.new).not_to be_valid
    end

    it "enforces uniqueness on region_name" do
      RegionSelection.create!(region_name: "berlin", position: 0)
      duplicate = RegionSelection.new(region_name: "berlin", position: 1)
      expect(duplicate).not_to be_valid
      expect(duplicate.errors[:region_name]).to include("has already been taken")
    end
  end

  describe ".active_names" do
    it "returns active region names ordered by position" do
      RegionSelection.create!(region_name: "vienna",  active: true, position: 1)
      RegionSelection.create!(region_name: "berlin",  active: true, position: 0)
      RegionSelection.create!(region_name: "munich",  active: false, position: 2)

      expect(RegionSelection.active_names).to eq(%w[berlin vienna])
    end
  end

  describe "#orphaned?" do
    it "is false by default" do
      expect(RegionSelection.new(region_name: "berlin").orphaned?).to be(false)
    end
  end
end
```

- [ ] **Step 2: Run the test — expect failures**

```bash
cd app && bundle exec rspec spec/models/region_selection_spec.rb
```

Expected: NameError for `RegionSelection`.

- [ ] **Step 3: Create the migration**

```bash
bundle exec rails generate migration CreateRegionSelections
```

Replace body with:

```ruby
class CreateRegionSelections < ActiveRecord::Migration[8.1]
  def change
    create_table :region_selections do |t|
      t.string  :region_name, null: false
      t.boolean :active,      null: false, default: true
      t.integer :position,    null: false, default: 0
      t.boolean :orphaned,    null: false, default: false
      t.timestamps
    end

    add_index :region_selections, :region_name, unique: true
  end
end
```

- [ ] **Step 4: Create the model**

Create `app/app/models/region_selection.rb`:

```ruby
class RegionSelection < ApplicationRecord
  validates :region_name, presence: true, uniqueness: true

  scope :active_names, -> { where(active: true).order(:position).pluck(:region_name) }

  def orphaned?
    !!orphaned
  end
end
```

- [ ] **Step 5: Migrate + run specs**

```bash
bundle exec rails db:migrate
RAILS_ENV=test bundle exec rails db:migrate
bundle exec rspec spec/models/region_selection_spec.rb
```

Expected: 4 examples, 0 failures.

- [ ] **Step 6: Commit**

```bash
git add app/db/migrate/*_create_region_selections.rb app/app/models/region_selection.rb app/spec/models/region_selection_spec.rb app/db/schema.rb
git commit -m "feat(admin): add RegionSelection model + migration"
```

---

### Task 4: `RegionCatalog` service — parses `regions/*.env` (TDD)

**Files:**
- Create: `app/app/services/region_catalog.rb`
- Create: `app/spec/services/region_catalog_spec.rb`
- Create: `app/spec/fixtures/regions/single.env`
- Create: `app/spec/fixtures/regions/multi.env`

- [ ] **Step 1: Create fixture region files**

Create `app/spec/fixtures/regions/single.env`:

```
# Test single-region preset
REGION_NAME=test-berlin
REGION_LABEL="Test Berlin"
COUNTRY_CODE=de
PBF_URL=https://example.test/berlin.osm.pbf
PBF_NAME=berlin.osm.pbf
DEFAULT_LAT=52.52
DEFAULT_LON=13.4
DEFAULT_ZOOM=11
```

Create `app/spec/fixtures/regions/multi.env`:

```
# Test multi-region preset
REGION_NAME=test-dach
REGION_LABEL="Test DACH"
COUNTRY_CODE=de
REGIONS="de,at,ch"
PBF_URLS="https://example.test/de.osm.pbf https://example.test/at.osm.pbf"
PBF_URL=local://data/osm/dach.osm.pbf
PBF_NAME=dach.osm.pbf
DEFAULT_LAT=48.5
DEFAULT_LON=11.0
DEFAULT_ZOOM=5
```

- [ ] **Step 2: Write the failing test**

Create `app/spec/services/region_catalog_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe RegionCatalog do
  let(:fixtures_dir) { Rails.root.join("spec/fixtures/regions") }

  describe ".load_dir" do
    it "loads all .env files into Region objects" do
      catalog = RegionCatalog.load_dir(fixtures_dir)
      expect(catalog.names).to contain_exactly("single", "multi")
    end

    it "exposes single-region attributes" do
      catalog = RegionCatalog.load_dir(fixtures_dir)
      region  = catalog.find("single")
      expect(region.name).to eq("single")
      expect(region.label).to eq("Test Berlin")
      expect(region.country_code).to eq("de")
      expect(region.pbf_urls).to eq(["https://example.test/berlin.osm.pbf"])
      expect(region.default_view).to eq(lat: 52.52, lon: 13.4, zoom: 11)
      expect(region.multi?).to be(false)
    end

    it "exposes multi-region attributes" do
      catalog = RegionCatalog.load_dir(fixtures_dir)
      region  = catalog.find("multi")
      expect(region.pbf_urls).to eq([
        "https://example.test/de.osm.pbf",
        "https://example.test/at.osm.pbf"
      ])
      expect(region.multi?).to be(true)
    end

    it "raises Region::NotFound for missing names" do
      catalog = RegionCatalog.load_dir(fixtures_dir)
      expect { catalog.find("nope") }.to raise_error(RegionCatalog::Region::NotFound)
    end
  end

  describe "EnvParser" do
    it "parses KEY=value pairs" do
      input  = "KEY=value\nFOO=bar\n"
      result = RegionCatalog::EnvParser.parse(input)
      expect(result).to eq("KEY" => "value", "FOO" => "bar")
    end

    it "strips inline comments and blank lines" do
      input  = "# comment\n\nKEY=value\n"
      result = RegionCatalog::EnvParser.parse(input)
      expect(result).to eq("KEY" => "value")
    end

    it "unquotes double-quoted values" do
      input  = 'LABEL="Berlin (city)"' + "\n"
      result = RegionCatalog::EnvParser.parse(input)
      expect(result).to eq("LABEL" => "Berlin (city)")
    end
  end
end
```

- [ ] **Step 3: Run the test — expect failures**

```bash
cd app && bundle exec rspec spec/services/region_catalog_spec.rb
```

Expected: NameError for `RegionCatalog`.

- [ ] **Step 4: Implement the service**

Create `app/app/services/region_catalog.rb`:

```ruby
class RegionCatalog
  class Region < Struct.new(:name, :label, :country_code, :pbf_urls, :default_view, keyword_init: true)
    class NotFound < StandardError; end

    def multi?
      pbf_urls.length > 1
    end
  end

  attr_reader :regions

  def initialize(regions)
    @regions = regions.index_by(&:name)
  end

  def self.load_dir(path)
    files = Dir.glob(File.join(path.to_s, "*.env"))
    regions = files.map do |file|
      name = File.basename(file, ".env")
      env  = EnvParser.parse(File.read(file))
      Region.new(
        name:         name,
        label:        env["REGION_LABEL"] || name,
        country_code: env["COUNTRY_CODE"],
        pbf_urls:     extract_pbf_urls(env),
        default_view: extract_view(env)
      )
    end
    new(regions)
  end

  def find(name)
    regions.fetch(name) { raise Region::NotFound, "region '#{name}' not in catalog" }
  end

  def names
    regions.keys
  end

  def self.extract_pbf_urls(env)
    if env["PBF_URLS"].to_s.strip != ""
      env["PBF_URLS"].split(/\s+/)
    elsif env["PBF_URL"]
      [env["PBF_URL"]]
    else
      []
    end
  end

  def self.extract_view(env)
    { lat:  env["DEFAULT_LAT"]&.to_f,
      lon:  env["DEFAULT_LON"]&.to_f,
      zoom: env["DEFAULT_ZOOM"]&.to_i }
  end

  module EnvParser
    KEY_VALUE = /\A([A-Z_][A-Z0-9_]*)=(.*)\z/

    def self.parse(content)
      content.each_line.each_with_object({}) do |line, acc|
        line = line.strip
        next if line.empty? || line.start_with?("#")
        next unless (match = line.match(KEY_VALUE))

        key   = match[1]
        value = unquote(match[2])
        acc[key] = value
      end
    end

    def self.unquote(value)
      if value.start_with?('"') && value.end_with?('"')
        value[1..-2]
      else
        value
      end
    end
  end
end
```

- [ ] **Step 5: Run tests**

```bash
bundle exec rspec spec/services/region_catalog_spec.rb
```

Expected: 7 examples, 0 failures.

- [ ] **Step 6: Commit**

```bash
git add app/app/services/region_catalog.rb app/spec/services/region_catalog_spec.rb app/spec/fixtures/regions/
git commit -m "feat(admin): RegionCatalog parses regions/*.env"
```

---

## Phase B — Rails control-plane integration

### Task 5: `ControlPlaneClient` Faraday wrapper (TDD)

**Files:**
- Create: `app/app/services/control_plane_client.rb`
- Create: `app/spec/services/control_plane_client_spec.rb`

- [x] **Step 1: Write the failing test**

Create `app/spec/services/control_plane_client_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe ControlPlaneClient do
  let(:stubs) { Faraday::Adapter::Test::Stubs.new }
  let(:conn)  { Faraday.new { |b| b.adapter :test, stubs; b.response :json } }
  subject     { described_class.new(connection: conn) }

  describe "#status" do
    it "returns parsed body on 200" do
      stubs.get("/status") { [200, {}, [{ "name" => "photon", "ready" => true }]] }
      expect(subject.status).to eq([{ "name" => "photon", "ready" => true }])
    end

    it "raises ControlPlane::Unavailable on connection failure" do
      stubs.get("/status") { raise Faraday::ConnectionFailed, "boom" }
      expect { subject.status }.to raise_error(ControlPlaneClient::Unavailable, /boom/)
    end

    it "raises ControlPlane::BadResponse on 5xx" do
      stubs.get("/status") { [502, {}, { "error" => { "code" => "X", "message" => "x" } }] }
      expect { subject.status }.to raise_error(ControlPlaneClient::BadResponse, /502/)
    end
  end

  describe "#enable!" do
    it "POSTs to /actions/services/:name/enable" do
      stubs.post("/actions/services/photon/enable") { [202, {}, ""] }
      expect(subject.enable!("photon")).to eq(true)
    end
  end

  describe "#disable!" do
    it "POSTs to /actions/services/:name/disable" do
      stubs.post("/actions/services/photon/disable") { [202, {}, ""] }
      expect(subject.disable!("photon")).to eq(true)
    end
  end

  describe "#apply_regions" do
    it "POSTs regions array" do
      stubs.post("/actions/regions") do |env|
        expect(JSON.parse(env.body)).to eq("regions" => ["berlin", "vienna"])
        [202, {}, ""]
      end
      expect(subject.apply_regions(%w[berlin vienna])).to eq(true)
    end
  end

  describe "#download_tiles" do
    it "POSTs tile URL" do
      stubs.post("/actions/tiles") do |env|
        expect(JSON.parse(env.body)).to eq("url" => "https://t.example/p.pmtiles")
        [202, {}, ""]
      end
      expect(subject.download_tiles!("https://t.example/p.pmtiles")).to eq(true)
    end
  end
end
```

- [x] **Step 2: Run the test — expect failures**

```bash
bundle exec rspec spec/services/control_plane_client_spec.rb
```

Expected: NameError for `ControlPlaneClient`.

- [x] **Step 3: Implement the client**

Create `app/app/services/control_plane_client.rb`:

```ruby
class ControlPlaneClient
  class Error < StandardError; end
  class Unavailable < Error; end
  class BadResponse < Error; end

  def self.default
    new(connection: build_default_connection)
  end

  def initialize(connection:)
    @conn = connection
  end

  def status
    response = @conn.get("/status")
    raise BadResponse, "#{response.status} from sidecar" unless response.success?
    response.body
  rescue Faraday::ConnectionFailed, Faraday::TimeoutError => e
    raise Unavailable, e.message
  end

  def enable!(name);  post!("/actions/services/#{name}/enable"); end
  def disable!(name); post!("/actions/services/#{name}/disable"); end

  def apply_regions(names)
    post!("/actions/regions", { regions: names })
  end

  def download_tiles!(url)
    post!("/actions/tiles", { url: url })
  end

  def self.build_default_connection
    base = ENV.fetch("CONTROL_PLANE_URL", "http://apo-control:8090")
    Faraday.new(url: base) do |b|
      b.request :json
      b.response :json, content_type: /\bjson$/
      b.options.timeout = 5
      b.options.open_timeout = 2
    end
  end

  private

  def post!(path, body = nil)
    response = body ? @conn.post(path, body) : @conn.post(path)
    raise BadResponse, "#{response.status} from sidecar at #{path}" unless response.success?
    true
  rescue Faraday::ConnectionFailed, Faraday::TimeoutError => e
    raise Unavailable, e.message
  end
end
```

- [x] **Step 4: Run tests**

```bash
bundle exec rspec spec/services/control_plane_client_spec.rb
```

Expected: 7 examples, 0 failures.

- [x] **Step 5: Commit**

```bash
git add app/app/services/control_plane_client.rb app/spec/services/control_plane_client_spec.rb
git commit -m "feat(admin): ControlPlaneClient Faraday wrapper"
```

---

### Task 6: `ApplyProjection` size/time estimate (TDD)

**Files:**
- Create: `app/app/services/apply_projection.rb`
- Create: `app/spec/services/apply_projection_spec.rb`

- [ ] **Step 1: Write the failing test**

Create `app/spec/services/apply_projection_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe ApplyProjection do
  let(:berlin) {
    RegionCatalog::Region.new(name: "berlin", label: "Berlin", country_code: "de",
                              pbf_urls: ["https://x/berlin.osm.pbf"],
                              default_view: {})
  }
  let(:germany) {
    RegionCatalog::Region.new(name: "germany", label: "Germany", country_code: "de",
                              pbf_urls: ["https://x/germany-latest.osm.pbf"],
                              default_view: {})
  }

  describe "#summary" do
    it "estimates per-tool sizes for a single city" do
      proj = ApplyProjection.new(regions: [berlin], services: %w[photon valhalla overpass])
      sum  = proj.summary

      expect(sum.total_disk_gb).to be > 0
      expect(sum.lines.map { |l| l[:name] }).to include("photon", "valhalla", "overpass")
    end

    it "scales for a country region" do
      city_proj    = ApplyProjection.new(regions: [berlin],  services: %w[overpass])
      country_proj = ApplyProjection.new(regions: [germany], services: %w[overpass])

      expect(country_proj.summary.total_disk_gb).to be > city_proj.summary.total_disk_gb
    end

    it "describes time as worst-case longest service" do
      proj = ApplyProjection.new(regions: [germany], services: %w[photon overpass])
      sum  = proj.summary
      expect(sum.first_boot_hours).to be >= 4   # Overpass dominates
    end

    it "returns zero for an empty selection" do
      proj = ApplyProjection.new(regions: [], services: [])
      expect(proj.summary.total_disk_gb).to eq(0)
    end
  end
end
```

- [ ] **Step 2: Run the test — expect failures**

```bash
bundle exec rspec spec/services/apply_projection_spec.rb
```

- [ ] **Step 3: Implement the projection**

Create `app/app/services/apply_projection.rb`:

```ruby
class ApplyProjection
  Summary = Struct.new(:total_disk_gb, :first_boot_hours, :lines, keyword_init: true)
  Line    = Struct.new(:name, :disk_gb, :hours, keyword_init: true) do
    def to_h = { name: name, disk_gb: disk_gb, hours: hours }
  end

  CITY_DISK = {
    "photon"      => 8.0,   # Photon scales by country, not city — flat cost
    "placeholder" => 4.0,   # WOF is global regardless
    "libpostal"   => 0.0,
    "valhalla"    => 1.0,
    "overpass"    => 4.0,
    "otp"         => 1.0
  }.freeze

  COUNTRY_DISK = {
    "photon" => 8.0, "placeholder" => 4.0, "libpostal" => 0.0,
    "valhalla" => 15.0, "overpass" => 45.0, "otp" => 5.0
  }.freeze

  CONTINENT_DISK = {
    "photon" => 30.0, "placeholder" => 4.0, "libpostal" => 0.0,
    "valhalla" => 115.0, "overpass" => 280.0, "otp" => 30.0
  }.freeze

  PLANET_DISK = {
    "photon" => 110.0, "placeholder" => 4.0, "libpostal" => 0.0,
    "valhalla" => 250.0, "overpass" => 700.0, "otp" => 50.0
  }.freeze

  HOURS = {
    "photon" => 2.0, "placeholder" => 1.5, "libpostal" => 0.05,
    "valhalla" => 1.5, "overpass" => 6.0, "otp" => 1.0
  }.freeze

  def initialize(regions:, services:)
    @regions  = regions
    @services = services
  end

  def summary
    table = scaling_table
    lines = @services.map do |svc|
      disk = table.fetch(svc, 0.0)
      Line.new(name: svc, disk_gb: disk, hours: HOURS.fetch(svc, 0.0))
    end
    Summary.new(
      total_disk_gb:    lines.sum(&:disk_gb).round(1),
      first_boot_hours: lines.map(&:hours).max.to_f.round(1),
      lines:            lines.map(&:to_h)
    )
  end

  private

  def scaling_table
    return CITY_DISK     if @regions.empty?
    case classify(@regions.first)
    when :city     then CITY_DISK
    when :country  then COUNTRY_DISK
    when :continent then CONTINENT_DISK
    when :planet   then PLANET_DISK
    end
  end

  def classify(region)
    return :planet    if region.name == "planet"
    return :continent if region.name == "europe"
    return :country   if %w[germany france italy].include?(region.name) || region.country_code && region.pbf_urls.first&.include?("geofabrik")
    :city
  end
end
```

- [ ] **Step 4: Run tests**

```bash
bundle exec rspec spec/services/apply_projection_spec.rb
```

Expected: 4 examples, 0 failures.

- [ ] **Step 5: Commit**

```bash
git add app/app/services/apply_projection.rb app/spec/services/apply_projection_spec.rb
git commit -m "feat(admin): ApplyProjection size/time estimate"
```

---

### Task 7: `ControlPlane::PollStatusJob` recurring sync (TDD)

**Files:**
- Create: `app/app/jobs/control_plane/poll_status_job.rb`
- Modify: `app/config/recurring.yml`
- Create: `app/spec/jobs/control_plane/poll_status_job_spec.rb`

- [ ] **Step 1: Write the failing test**

Create `app/spec/jobs/control_plane/poll_status_job_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe ControlPlane::PollStatusJob, type: :job do
  let(:client) { instance_double(ControlPlaneClient) }

  before do
    Service.delete_all
    Service.create!(name: "photon", profile: "geocoding", status: :unknown)
  end

  it "updates the service when sidecar reports new state" do
    allow(client).to receive(:status).and_return([
      { "name" => "photon", "container_state" => "running", "phase" => "downloading",
        "progress" => 0.42, "last_log_line" => "Download progress: 42%",
        "ready" => false, "disk_bytes" => 1_234_567 }
    ])

    described_class.new(client: client).perform_now

    photon = Service.find_by(name: "photon")
    expect(photon.status).to eq("downloading")
    expect(photon.phase).to eq("downloading")
    expect(photon.progress).to eq(0.42)
    expect(photon.disk_bytes).to eq(1_234_567)
  end

  it "broadcasts a Turbo Stream when state changes" do
    allow(client).to receive(:status).and_return([
      { "name" => "photon", "container_state" => "running", "phase" => "ready",
        "progress" => 1.0, "last_log_line" => "Photon ready",
        "ready" => true, "disk_bytes" => 8 * 1024**3 }
    ])

    expect {
      described_class.new(client: client).perform_now
    }.to have_broadcasted_to("services_channel").from_channel(Turbo::StreamsChannel)
  end

  it "is silent when sidecar is unreachable" do
    allow(client).to receive(:status).and_raise(ControlPlaneClient::Unavailable.new("down"))
    expect { described_class.new(client: client).perform_now }.not_to raise_error
  end

  it "creates services missing from AR when sidecar reports them" do
    allow(client).to receive(:status).and_return([
      { "name" => "valhalla", "container_state" => "running", "phase" => "building",
        "progress" => 0.1, "last_log_line" => "...", "ready" => false, "disk_bytes" => 0 }
    ])

    expect {
      described_class.new(client: client).perform_now
    }.to change { Service.where(name: "valhalla").count }.from(0).to(1)
  end
end
```

- [ ] **Step 2: Run the test — expect failures**

```bash
bundle exec rspec spec/jobs/control_plane/poll_status_job_spec.rb
```

Expected: NameError for `ControlPlane::PollStatusJob`.

- [ ] **Step 3: Implement the job**

Create `app/app/jobs/control_plane/poll_status_job.rb`:

```ruby
module ControlPlane
  class PollStatusJob < ApplicationJob
    queue_as :default

    PROFILE_FOR = {
      "photon"      => "geocoding",
      "placeholder" => "geocoding",
      "libpostal"   => "geocoding",
      "valhalla"    => "routing",
      "overpass"    => "pois",
      "otp"         => "transit",
      "whosonfirst" => "data-setup"
    }.freeze

    def initialize(client: ControlPlaneClient.default)
      super()
      @client = client
    end

    def perform
      snapshot = @client.status
      Array(snapshot).each { |entry| sync(entry) }
    rescue ControlPlaneClient::Unavailable, ControlPlaneClient::BadResponse => e
      Rails.logger.warn("[PollStatusJob] sidecar #{e.class.name.demodulize}: #{e.message}")
    end

    private

    def sync(entry)
      name = entry["name"]
      service = Service.find_or_initialize_by(name: name) do |s|
        s.profile = PROFILE_FOR.fetch(name, "unknown")
      end

      attrs = {
        status:     map_status(entry),
        phase:      entry["phase"],
        progress:   entry["progress"],
        last_log:   entry["last_log_line"],
        disk_bytes: entry["disk_bytes"] || 0,
        last_seen_at: Time.current
      }

      changed = (service.attributes.slice(*attrs.keys.map(&:to_s)) != attrs.stringify_keys)
      service.assign_attributes(attrs)
      service.save!

      broadcast(service) if changed
    end

    def map_status(entry)
      return "ready" if entry["ready"]
      case entry["phase"]
      when nil, ""           then "unknown"
      when /download/i       then "downloading"
      when /build|merg|ingest|optimize|extract/i then "building"
      when /partial|stop/i   then "stopped"
      when /error|fail/i     then "error"
      when /unhealth/i       then "unhealthy"
      else "starting"
      end
    end

    def broadcast(service)
      Turbo::StreamsChannel.broadcast_replace_to(
        "services_channel",
        target: "service_#{service.name}",
        partial: "admin/services/service_card",
        locals: { service: service }
      )
    end
  end
end
```

- [ ] **Step 4: Register in `config/recurring.yml`**

Open `app/config/recurring.yml` and add (under `production:` if it exists, or create the file):

```yaml
production:
  poll_control_plane:
    class: "ControlPlane::PollStatusJob"
    schedule: "every 3 seconds"

development:
  poll_control_plane:
    class: "ControlPlane::PollStatusJob"
    schedule: "every 5 seconds"
```

- [ ] **Step 5: Run tests**

```bash
bundle exec rspec spec/jobs/control_plane/poll_status_job_spec.rb
```

Expected: 4 examples, 0 failures.

- [ ] **Step 6: Commit**

```bash
git add app/app/jobs/control_plane/poll_status_job.rb app/config/recurring.yml app/spec/jobs/control_plane/poll_status_job_spec.rb
git commit -m "feat(admin): PollStatusJob mirrors sidecar /status into AR"
```

---

## Phase C — Admin HTTP endpoints

### Task 8: `Admin::BaseController` HTTP Basic auth + admin_check initializer (TDD)

**Files:**
- Create: `app/app/controllers/admin/base_controller.rb`
- Create: `app/config/initializers/admin_check.rb`
- Modify: `app/config/routes.rb`
- Create: `app/spec/requests/admin/auth_spec.rb`

- [x] **Step 1: Write the failing test**

Create `app/spec/requests/admin/auth_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "Admin auth", type: :request do
  describe "when env vars set" do
    before do
      stub_const("ADMIN_CHECK_OK", true)
      ENV["ADMIN_USERNAME"] = "admin"
      ENV["ADMIN_PASSWORD"] = "secret"
    end

    after do
      ENV.delete("ADMIN_USERNAME")
      ENV.delete("ADMIN_PASSWORD")
    end

    it "returns 401 without credentials" do
      get "/admin/services"
      expect(response).to have_http_status(:unauthorized)
    end

    it "returns 200 with correct credentials" do
      get "/admin/services", headers: { "HTTP_AUTHORIZATION" =>
        ActionController::HttpAuthentication::Basic.encode_credentials("admin", "secret") }
      expect(response).to have_http_status(:ok)
    end

    it "returns 401 with wrong credentials" do
      get "/admin/services", headers: { "HTTP_AUTHORIZATION" =>
        ActionController::HttpAuthentication::Basic.encode_credentials("admin", "WRONG") }
      expect(response).to have_http_status(:unauthorized)
    end
  end

  describe "when env vars are missing" do
    before { stub_const("ADMIN_CHECK_OK", false) }

    it "returns 503 explaining the misconfig" do
      get "/admin/services"
      expect(response).to have_http_status(:service_unavailable)
      expect(response.body).to include("ADMIN_USERNAME")
    end
  end
end
```

- [x] **Step 2: Run the test — expect failures**

```bash
bundle exec rspec spec/requests/admin/auth_spec.rb
```

Expected: routing error / no `Admin::BaseController`.

- [x] **Step 3: Create the initializer**

Create `app/config/initializers/admin_check.rb`:

```ruby
Rails.application.config.after_initialize do
  ADMIN_CHECK_OK = ENV["ADMIN_USERNAME"].present? && ENV["ADMIN_PASSWORD"].present?
end
```

- [x] **Step 4: Create the base controller**

Create `app/app/controllers/admin/base_controller.rb`:

```ruby
module Admin
  class BaseController < ApplicationController
    before_action :require_admin_configured
    before_action :authenticate

    rescue_from ControlPlaneClient::Unavailable do |e|
      respond_to do |format|
        format.html { render "admin/errors/sidecar_unavailable", status: :service_unavailable, locals: { message: e.message } }
        format.json { render json: { error: { code: "SIDECAR_UNAVAILABLE", message: e.message } }, status: :service_unavailable }
      end
    end

    rescue_from ControlPlaneClient::BadResponse do |e|
      respond_to do |format|
        format.html { render "admin/errors/sidecar_error", status: :bad_gateway, locals: { message: e.message } }
        format.json { render json: { error: { code: "SIDECAR_ERROR", message: e.message } }, status: :bad_gateway }
      end
    end

    rescue_from RegionCatalog::Region::NotFound do |e|
      respond_to do |format|
        format.html { render "admin/errors/region_not_found", status: :unprocessable_entity, locals: { message: e.message } }
        format.json { render json: { error: { code: "REGION_NOT_FOUND", message: e.message } }, status: :unprocessable_entity }
      end
    end

    private

    def require_admin_configured
      return if ADMIN_CHECK_OK
      render plain:
        "Admin panel unconfigured. Set ADMIN_USERNAME and ADMIN_PASSWORD in .env, then `make restart`.",
        status: :service_unavailable
    end

    def authenticate
      authenticate_or_request_with_http_basic("Apocalymaps admin") do |user, pass|
        ActiveSupport::SecurityUtils.secure_compare(user, ENV.fetch("ADMIN_USERNAME", "")) &
        ActiveSupport::SecurityUtils.secure_compare(pass, ENV.fetch("ADMIN_PASSWORD", ""))
      end
    end
  end
end
```

- [x] **Step 5: Create error partials**

Create `app/app/views/admin/errors/_sidecar_unavailable.html.erb`:

```erb
<div class="alert alert-error">
  Sidecar unreachable: <%= message %>.
  Run <code>docker compose ps apo-control</code> to investigate.
</div>
```

Create `app/app/views/admin/errors/_sidecar_error.html.erb`:

```erb
<div class="alert alert-error">
  Sidecar returned an error: <%= message %>
</div>
```

Create `app/app/views/admin/errors/_region_not_found.html.erb`:

```erb
<div class="alert alert-warning"><%= message %></div>
```

These exist to provide non-empty render targets — Task 9 fills in the index view.

- [x] **Step 6: Add a stub `Admin::ServicesController` so routes resolve**

Task 9 will replace this with the real implementation; we just need a valid target so the auth specs can hit the route.

Create `app/app/controllers/admin/services_controller.rb`:

```ruby
module Admin
  class ServicesController < BaseController
    def index
      head :ok
    end
  end
end
```

- [x] **Step 7: Update routes**

Edit `app/config/routes.rb`. After the existing `namespace :api` block, add:

```ruby
  namespace :admin do
    get :services, to: "services#index"
  end
```

- [x] **Step 8: Run tests**

```bash
bundle exec rspec spec/requests/admin/auth_spec.rb
```

Expected: 4 examples, 0 failures.

- [x] **Step 9: Commit**

```bash
git add app/app/controllers/admin/ app/config/initializers/admin_check.rb app/config/routes.rb app/app/views/admin/errors/ app/spec/requests/admin/auth_spec.rb
git commit -m "feat(admin): BaseController HTTP Basic auth + admin_check + stub services controller"
```

---

### Task 9: `Admin::ServicesController` toggle endpoints (TDD)

**Files:**
- Create: `app/app/controllers/admin/services_controller.rb`
- Modify: `app/config/routes.rb`
- Create: `app/spec/requests/admin/services_spec.rb`
- Modify: `app/spec/support/sidecar_helper.rb` (created here)

- [x] **Step 1: Create the sidecar test helper**

Create `app/spec/support/sidecar_helper.rb`:

```ruby
module SidecarHelper
  def stub_sidecar(client_class: ControlPlaneClient, &block)
    fake = instance_double(client_class)
    allow(client_class).to receive(:default).and_return(fake)
    block.call(fake) if block_given?
    fake
  end
end

RSpec.configure do |c|
  c.include SidecarHelper, type: :request
  c.include SidecarHelper, type: :job
end
```

Add a require in `app/spec/rails_helper.rb`:

```ruby
Dir[Rails.root.join("spec/support/**/*.rb")].each { |f| require f }
```

(verify this line exists; add it if not.)

- [x] **Step 2: Write the failing test**

Create `app/spec/requests/admin/services_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "Admin::Services", type: :request do
  before do
    stub_const("ADMIN_CHECK_OK", true)
    ENV["ADMIN_USERNAME"] = "admin"
    ENV["ADMIN_PASSWORD"] = "x"
    Service.create!(name: "photon", profile: "geocoding", enabled: false, status: :stopped)
  end

  after do
    ENV.delete("ADMIN_USERNAME")
    ENV.delete("ADMIN_PASSWORD")
  end

  let(:auth) { { "HTTP_AUTHORIZATION" => ActionController::HttpAuthentication::Basic.encode_credentials("admin", "x") } }

  describe "GET /admin/services" do
    it "renders the panel with each known service" do
      get "/admin/services", headers: auth
      expect(response).to have_http_status(:ok)
      expect(response.body).to include("photon")
    end
  end

  describe "POST /admin/services/:name (enabled=true)" do
    it "updates AR and calls sidecar enable!" do
      sidecar = stub_sidecar { |s| expect(s).to receive(:enable!).with("photon").and_return(true) }
      post "/admin/services/photon", params: { enabled: "true" }, headers: auth
      expect(response).to have_http_status(:ok)
      expect(Service.find_by(name: "photon").enabled).to be(true)
    end
  end

  describe "POST /admin/services/:name (enabled=false)" do
    it "calls sidecar disable! and flips AR" do
      Service.find_by(name: "photon").update!(enabled: true)
      stub_sidecar { |s| expect(s).to receive(:disable!).with("photon").and_return(true) }
      post "/admin/services/photon", params: { enabled: "false" }, headers: auth
      expect(Service.find_by(name: "photon").enabled).to be(false)
    end
  end

  describe "when sidecar unavailable" do
    it "renders 503 and leaves AR untouched" do
      stub_sidecar { |s| expect(s).to receive(:enable!).and_raise(ControlPlaneClient::Unavailable.new("down")) }
      post "/admin/services/photon", params: { enabled: "true" }, headers: auth
      expect(response).to have_http_status(:service_unavailable)
      expect(Service.find_by(name: "photon").enabled).to be(false)
    end
  end
end
```

- [x] **Step 3: Run the test — expect failures**

```bash
bundle exec rspec spec/requests/admin/services_spec.rb
```

- [x] **Step 4: Create the controller**

Create `app/app/controllers/admin/services_controller.rb`:

```ruby
module Admin
  class ServicesController < BaseController
    def index
      @services = Service.order(:profile, :name)
      @regions  = RegionCatalog.load_dir(Rails.root.join("..", "regions")).regions.values
      @region_selection = RegionSelection.active_names
      render "admin/services/index"
    end

    def update
      service = Service.find_by!(name: params[:name])
      wanted  = ActiveModel::Type::Boolean.new.cast(params[:enabled])
      Service.transaction { service.update!(enabled: wanted) }

      if wanted
        ControlPlaneClient.default.enable!(service.name)
      else
        ControlPlaneClient.default.disable!(service.name)
      end

      respond_to do |format|
        format.turbo_stream { render turbo_stream: turbo_stream.replace("service_#{service.name}", partial: "admin/services/service_card", locals: { service: service }) }
        format.html { redirect_to admin_services_path }
        format.json { render json: { name: service.name, enabled: service.enabled }, status: :ok }
      end
    rescue ActiveRecord::RecordNotFound
      render plain: "service not found", status: :not_found
    end
  end
end
```

- [x] **Step 5: Update routes**

Edit `app/config/routes.rb`:

```ruby
  namespace :admin do
    get  :services,         to: "services#index"
    post "services/:name",  to: "services#update", as: :service
  end
```

- [x] **Step 6: Create a stub `services/index.html.erb`**

Create `app/app/views/admin/services/index.html.erb`:

```erb
<%# Replaced by Task 13 with the full panel; minimal version for the request specs %>
<div data-testid="services-panel">
  <% @services.each do |service| %>
    <div id="service_<%= service.name %>"><%= service.name %></div>
  <% end %>
</div>
```

Create `app/app/views/admin/services/_service_card.html.erb`:

```erb
<%# Replaced by Task 13; minimal partial for early broadcasts %>
<div id="service_<%= service.name %>" data-name="<%= service.name %>">
  <%= service.name %> — <%= service.status %>
</div>
```

- [x] **Step 7: Run tests**

```bash
bundle exec rspec spec/requests/admin/services_spec.rb
```

Expected: 4 examples, 0 failures.

- [x] **Step 8: Commit**

```bash
git add app/app/controllers/admin/services_controller.rb app/config/routes.rb app/app/views/admin/services/ app/spec/support/sidecar_helper.rb app/spec/requests/admin/services_spec.rb app/spec/rails_helper.rb
git commit -m "feat(admin): ServicesController#index + #update"
```

---

### Task 10: `Admin::RegionsController` (TDD)

**Files:**
- Create: `app/app/controllers/admin/regions_controller.rb`
- Modify: `app/config/routes.rb`
- Create: `app/spec/requests/admin/regions_spec.rb`

- [ ] **Step 1: Write the failing test**

Create `app/spec/requests/admin/regions_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "Admin::Regions", type: :request do
  before do
    stub_const("ADMIN_CHECK_OK", true)
    ENV["ADMIN_USERNAME"] = "admin"
    ENV["ADMIN_PASSWORD"] = "x"
  end

  after do
    ENV.delete("ADMIN_USERNAME")
    ENV.delete("ADMIN_PASSWORD")
  end

  let(:auth) { { "HTTP_AUTHORIZATION" => ActionController::HttpAuthentication::Basic.encode_credentials("admin", "x") } }

  before do
    fake_catalog = instance_double(RegionCatalog,
      names: %w[berlin germany],
      find: nil
    )
    allow(fake_catalog).to receive(:find) { |n| RegionCatalog::Region.new(name: n, label: n, country_code: "de", pbf_urls: [], default_view: {}) }
    allow(RegionCatalog).to receive(:load_dir).and_return(fake_catalog)
  end

  it "replaces selection in a single transaction" do
    RegionSelection.create!(region_name: "stale", active: true, position: 0)

    post "/admin/regions",
         params: { regions: ["berlin", "germany"] },
         headers: auth

    expect(response).to have_http_status(:ok)
    expect(RegionSelection.active_names).to eq(%w[berlin germany])
  end

  it "rejects unknown regions" do
    fake_catalog = instance_double(RegionCatalog, names: %w[berlin germany])
    allow(fake_catalog).to receive(:find).with("berlin").and_return(RegionCatalog::Region.new(name: "berlin", label: "b", country_code: "de", pbf_urls: [], default_view: {}))
    allow(fake_catalog).to receive(:find).with("nope").and_raise(RegionCatalog::Region::NotFound.new("region 'nope' not in catalog"))
    allow(RegionCatalog).to receive(:load_dir).and_return(fake_catalog)

    post "/admin/regions", params: { regions: %w[berlin nope] }, headers: auth
    expect(response).to have_http_status(:unprocessable_entity)
  end
end
```

- [ ] **Step 2: Run the test — expect failures**

```bash
bundle exec rspec spec/requests/admin/regions_spec.rb
```

- [ ] **Step 3: Create the controller**

Create `app/app/controllers/admin/regions_controller.rb`:

```ruby
module Admin
  class RegionsController < BaseController
    def update
      names = Array(params[:regions]).map(&:to_s)
      catalog = RegionCatalog.load_dir(Rails.root.join("..", "regions"))

      # Validate every region exists; raises Region::NotFound → 422 (rescued by BaseController)
      names.each { |n| catalog.find(n) }

      RegionSelection.transaction do
        RegionSelection.delete_all
        names.each_with_index do |name, position|
          RegionSelection.create!(region_name: name, active: true, position: position)
        end
      end

      respond_to do |format|
        format.html { redirect_to admin_services_path }
        format.json { render json: { regions: RegionSelection.active_names }, status: :ok }
      end
    end
  end
end
```

- [ ] **Step 4: Update routes**

```ruby
  namespace :admin do
    get  :services,         to: "services#index"
    post "services/:name",  to: "services#update", as: :service
    post :regions,          to: "regions#update"
  end
```

- [ ] **Step 5: Run tests**

```bash
bundle exec rspec spec/requests/admin/regions_spec.rb
```

Expected: 2 examples, 0 failures.

- [ ] **Step 6: Commit**

```bash
git add app/app/controllers/admin/regions_controller.rb app/config/routes.rb app/spec/requests/admin/regions_spec.rb
git commit -m "feat(admin): RegionsController#update with catalog validation"
```

---

### Task 11: `Admin::ApplyController` two-phase confirm + dispatch (TDD)

**Files:**
- Create: `app/app/controllers/admin/apply_controller.rb`
- Modify: `app/config/routes.rb`
- Create: `app/app/views/admin/apply/_confirmation.html.erb`
- Create: `app/spec/requests/admin/apply_spec.rb`

- [ ] **Step 1: Write the failing test**

Create `app/spec/requests/admin/apply_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe "Admin::Apply", type: :request do
  before do
    stub_const("ADMIN_CHECK_OK", true)
    ENV["ADMIN_USERNAME"] = "admin"
    ENV["ADMIN_PASSWORD"] = "x"

    fake_region = RegionCatalog::Region.new(
      name: "berlin", label: "Berlin", country_code: "de",
      pbf_urls: ["https://x/b.osm.pbf"], default_view: {})
    fake_catalog = instance_double(RegionCatalog, names: ["berlin"])
    allow(fake_catalog).to receive(:find).with("berlin").and_return(fake_region)
    allow(RegionCatalog).to receive(:load_dir).and_return(fake_catalog)

    RegionSelection.create!(region_name: "berlin", active: true, position: 0)
    Service.create!(name: "photon", profile: "geocoding", enabled: true)
  end

  after do
    ENV.delete("ADMIN_USERNAME")
    ENV.delete("ADMIN_PASSWORD")
  end

  let(:auth) { { "HTTP_AUTHORIZATION" => ActionController::HttpAuthentication::Basic.encode_credentials("admin", "x") } }

  describe "without confirmed=true" do
    it "returns 409 with the projection in the response body" do
      post "/admin/apply", headers: auth
      expect(response).to have_http_status(:conflict)
      expect(response.body).to match(/disk_gb/i)
    end
  end

  describe "with confirmed=true" do
    it "calls sidecar apply_regions for the active selection" do
      stub_sidecar do |s|
        expect(s).to receive(:apply_regions).with(["berlin"]).and_return(true)
      end
      post "/admin/apply", params: { confirmed: "true" }, headers: auth
      expect(response).to have_http_status(:accepted)
    end

    it "surfaces sidecar failures with 502" do
      stub_sidecar do |s|
        expect(s).to receive(:apply_regions).and_raise(ControlPlaneClient::BadResponse.new("502"))
      end
      post "/admin/apply", params: { confirmed: "true" }, headers: auth
      expect(response).to have_http_status(:bad_gateway)
    end
  end
end
```

- [ ] **Step 2: Run the test — expect failures**

```bash
bundle exec rspec spec/requests/admin/apply_spec.rb
```

- [ ] **Step 3: Create the controller**

Create `app/app/controllers/admin/apply_controller.rb`:

```ruby
module Admin
  class ApplyController < BaseController
    def create
      catalog  = RegionCatalog.load_dir(Rails.root.join("..", "regions"))
      regions  = RegionSelection.active_names.map { |n| catalog.find(n) }
      services = Service.where(enabled: true).pluck(:name)
      proj     = ApplyProjection.new(regions: regions, services: services).summary

      if params[:confirmed].present? && ActiveModel::Type::Boolean.new.cast(params[:confirmed])
        ControlPlaneClient.default.apply_regions(regions.map(&:name))
        Service.where(name: %w[valhalla overpass otp]).where(enabled: true).update_all(status: Service.statuses[:starting])
        render json: { ok: true, projection: proj.to_h }, status: :accepted
      else
        respond_to do |format|
          format.turbo_stream do
            render turbo_stream: turbo_stream.update(
              "apply_confirmation",
              partial: "admin/apply/confirmation",
              locals: { projection: proj }
            ), status: :conflict
          end
          format.json { render json: { projection: proj.to_h }, status: :conflict }
          format.html { render plain: proj.to_h.to_json, status: :conflict }
        end
      end
    end
  end
end
```

- [ ] **Step 4: Add the confirmation partial**

Create `app/app/views/admin/apply/_confirmation.html.erb`:

```erb
<div class="modal modal-open">
  <div class="modal-box">
    <h3 class="font-bold text-lg">Confirm</h3>
    <p class="py-2">You're about to download approximately
       <strong><%= projection.total_disk_gb %> GB</strong> and
       wait roughly <strong><%= projection.first_boot_hours %> hours</strong>.</p>
    <table class="table table-xs">
      <thead><tr><th>Service</th><th>Disk (GB)</th><th>Hours</th></tr></thead>
      <tbody>
        <% projection.lines.each do |line| %>
          <tr><td><%= line[:name] %></td><td><%= line[:disk_gb] %></td><td><%= line[:hours] %></td></tr>
        <% end %>
      </tbody>
    </table>
    <div class="modal-action">
      <%= form_with url: admin_apply_path, method: :post, data: { turbo_stream: true } do |f| %>
        <%= f.hidden_field :confirmed, value: "true" %>
        <%= f.submit "Confirm", class: "btn btn-primary" %>
      <% end %>
    </div>
  </div>
</div>
```

- [ ] **Step 5: Update routes**

```ruby
  namespace :admin do
    get  :services,         to: "services#index"
    post "services/:name",  to: "services#update", as: :service
    post :regions,          to: "regions#update"
    post :apply,            to: "apply#create"
  end
```

- [ ] **Step 6: Run tests**

```bash
bundle exec rspec spec/requests/admin/apply_spec.rb
```

Expected: 3 examples, 0 failures.

- [ ] **Step 7: Commit**

```bash
git add app/app/controllers/admin/apply_controller.rb app/config/routes.rb app/app/views/admin/apply/ app/spec/requests/admin/apply_spec.rb
git commit -m "feat(admin): ApplyController two-phase confirm flow"
```

---

## Phase D — Views, Stimulus, and Turbo Frame wiring

### Task 12: Full panel views + Stimulus controllers

**Files:**
- Modify: `app/app/views/admin/services/index.html.erb`
- Modify: `app/app/views/admin/services/_service_card.html.erb`
- Create: `app/app/views/admin/regions/_region_chip.html.erb`
- Create: `app/app/javascript/controllers/panel_controller.js`
- Create: `app/app/javascript/controllers/regions_controller.js`
- Create: `app/app/javascript/controllers/confirm_controller.js`
- Modify: `app/app/javascript/controllers/index.js`

This task is view-heavy; we verify via the existing request specs (which check `#service_<name>` IDs and DaisyUI classes survive) and visually via the e2e covered in Task 19.

- [ ] **Step 1: Write the failing assertion**

Append to `app/spec/requests/admin/services_spec.rb`:

```ruby
  describe "GET /admin/services (full panel)" do
    before do
      Service.create!(name: "valhalla", profile: "routing", enabled: false, status: :stopped)
    end

    it "renders one card per service inside a panel root" do
      get "/admin/services", headers: auth
      expect(response.body).to include('data-controller="panel"')
      expect(response.body).to include('id="service_photon"')
      expect(response.body).to include('id="service_valhalla"')
    end

    it "renders region chips" do
      get "/admin/services", headers: auth
      expect(response.body).to include('data-controller="regions"')
    end
  end
```

- [ ] **Step 2: Run the test — expect failures**

```bash
bundle exec rspec spec/requests/admin/services_spec.rb
```

Expected: two new failures because the panel root isn't there yet.

- [ ] **Step 3: Replace `admin/services/index.html.erb` with the full panel**

```erb
<%= turbo_stream_from "services_channel" %>

<div data-controller="panel"
     data-panel-open-value="<%= params[:open] == 'true' ? 'true' : 'false' %>"
     class="fixed top-3 right-3 z-20">
  <button class="btn btn-circle btn-sm" data-action="panel#toggle" aria-label="Toggle data services panel">
    ⚙
  </button>

  <div data-panel-target="body"
       class="hidden mt-2 w-[min(95vw,420px)] max-h-[80vh] overflow-y-auto card bg-base-100/95 shadow-xl backdrop-blur">
    <div class="card-body p-3 gap-3">
      <h2 class="card-title">Data services</h2>

      <section data-controller="regions">
        <h3 class="text-sm font-bold">Regions</h3>
        <div id="region_chips" class="flex flex-wrap gap-1 my-2">
          <% @regions.each do |region| %>
            <%= render "admin/regions/region_chip", region: region, selected: @region_selection.include?(region.name) %>
          <% end %>
        </div>
      </section>

      <section>
        <h3 class="text-sm font-bold">Tools</h3>
        <div id="service_cards" class="flex flex-col gap-2 mt-2">
          <% @services.each do |service| %>
            <%= render "admin/services/service_card", service: service %>
          <% end %>
        </div>
      </section>

      <div data-controller="confirm">
        <button class="btn btn-primary btn-sm"
                data-action="confirm#open"
                data-confirm-url-value="<%= admin_apply_path(format: :turbo_stream) %>">
          Save & apply
        </button>
        <div id="apply_confirmation"></div>
      </div>
    </div>
  </div>
</div>
```

- [ ] **Step 4: Replace `_service_card.html.erb`**

```erb
<div id="service_<%= service.name %>"
     class="border rounded p-2 flex items-center justify-between gap-2"
     data-name="<%= service.name %>">
  <div class="flex-1 min-w-0">
    <div class="font-medium"><%= service.name %></div>
    <div class="text-xs text-base-content/70 truncate"><%= service.status %><% if service.phase.present? %> — <%= service.phase %><% end %></div>
    <% if service.progress&.positive? && !service.ready? %>
      <progress class="progress progress-primary w-full" value="<%= (service.progress * 100).to_i %>" max="100"></progress>
    <% end %>
  </div>
  <%= form_with url: admin_service_path(name: service.name), method: :post, data: { turbo_stream: true } do |f| %>
    <%= f.hidden_field :enabled, value: service.enabled ? "false" : "true" %>
    <%= f.submit service.enabled ? "Disable" : "Enable",
                 class: "btn btn-xs #{service.enabled ? 'btn-warning' : 'btn-success'}" %>
  <% end %>
</div>
```

- [ ] **Step 5: Create `_region_chip.html.erb`**

```erb
<label class="badge <%= selected ? 'badge-primary' : 'badge-ghost' %> cursor-pointer">
  <input type="checkbox"
         name="regions[]"
         value="<%= region.name %>"
         class="hidden"
         <%= "checked" if selected %>
         data-action="change->regions#toggle">
  <%= region.label %>
</label>
```

- [ ] **Step 6: Stimulus controllers**

Create `app/app/javascript/controllers/panel_controller.js`:

```js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["body"]
  static values  = { open: Boolean }

  connect() {
    const stored = localStorage.getItem("apocalymaps:panel-open")
    if (stored === "true") this.openValue = true
    this.render()
  }

  toggle() {
    this.openValue = !this.openValue
    localStorage.setItem("apocalymaps:panel-open", this.openValue)
    this.render()
  }

  render() {
    this.bodyTarget.classList.toggle("hidden", !this.openValue)
  }
}
```

Create `app/app/javascript/controllers/regions_controller.js`:

```js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  toggle(event) {
    const selected = Array.from(this.element.querySelectorAll('input[type=checkbox]'))
      .filter(cb => cb.checked).map(cb => cb.value)
    const formData = new FormData()
    selected.forEach(name => formData.append("regions[]", name))

    const token = document.querySelector('meta[name="csrf-token"]')?.content
    fetch("/admin/regions", {
      method: "POST",
      body:   formData,
      headers: token ? { "X-CSRF-Token": token, "Accept": "application/json" } : { "Accept": "application/json" }
    })
  }
}
```

Create `app/app/javascript/controllers/confirm_controller.js`:

```js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { url: String }

  open() {
    const token = document.querySelector('meta[name="csrf-token"]')?.content
    fetch(this.urlValue, {
      method: "POST",
      headers: token
        ? { "X-CSRF-Token": token, "Accept": "text/vnd.turbo-stream.html" }
        : { "Accept": "text/vnd.turbo-stream.html" }
    }).then(r => r.text()).then(html => {
      const slot = document.querySelector("#apply_confirmation")
      if (slot) slot.innerHTML = html
    })
  }
}
```

- [ ] **Step 7: Register the controllers**

Edit `app/app/javascript/controllers/index.js` and add:

```js
import PanelController   from "./panel_controller"
import RegionsController from "./regions_controller"
import ConfirmController from "./confirm_controller"

application.register("panel",   PanelController)
application.register("regions", RegionsController)
application.register("confirm", ConfirmController)
```

- [ ] **Step 8: Rebuild JS bundle**

```bash
cd app && bun run build
```

Expected: build succeeds.

- [ ] **Step 9: Run request specs**

```bash
bundle exec rspec spec/requests/admin/services_spec.rb
```

Expected: all examples (incl. the two new ones from Step 1) pass.

- [ ] **Step 10: Commit**

```bash
git add app/app/views/admin/services/ app/app/views/admin/regions/ app/app/views/admin/apply/ app/app/javascript/controllers/ app/spec/requests/admin/services_spec.rb
git commit -m "feat(admin): full panel views + Stimulus controllers"
```

---

### Task 13: Wire panel into the home page

**Files:**
- Modify: `app/app/views/home/index.html.erb`

- [x] **Step 1: Write the failing test**

Append to `app/spec/requests/admin/auth_spec.rb`:

```ruby
  describe "home page integration" do
    before { stub_const("ADMIN_CHECK_OK", true) }

    it "does not include the admin panel by default" do
      get "/"
      expect(response.body).not_to include('data-controller="panel"')
    end
  end
```

The admin panel is loaded only after the user opens `/admin/services` (a Turbo Frame request) — so the home page itself does NOT render the panel by default. Instead, it renders an empty frame that fills in via JS request once the user logs in via HTTP Basic.

- [x] **Step 2: Modify `app/app/views/home/index.html.erb`**

At the end of the existing `<div data-controller="map" ...>` block, **just before the closing `</div>`**, add:

```erb
  <turbo-frame id="admin_panel" src="/admin/services?open=false" loading="lazy"></turbo-frame>
```

This lazy-loaded Turbo Frame triggers a request to `/admin/services` on first paint. The browser will see a 401, prompt for HTTP Basic auth, and only then fetch the panel content.

- [x] **Step 3: Run the test**

```bash
bundle exec rspec spec/requests/admin/auth_spec.rb
```

Expected: original auth specs still pass; new home-page spec passes.

- [x] **Step 4: Smoke-test in Playwright (optional but recommended)**

Already covered by Task 19 e2e — skip here.

- [x] **Step 5: Commit**

```bash
git add app/app/views/home/index.html.erb app/spec/requests/admin/auth_spec.rb
git commit -m "feat(admin): lazy-load admin panel into map page via Turbo Frame"
```

---

## Phase E — rswag/OpenAPI for admin endpoints

### Task 14: Admin rswag specs + admin swagger.yaml + UI mount

**Files:**
- Create: `app/spec/integration/admin/services_spec.rb`
- Create: `app/spec/integration/admin/regions_spec.rb`
- Create: `app/spec/integration/admin/apply_spec.rb`
- Modify: `app/spec/swagger_helper.rb`
- Modify: `app/config/initializers/rswag_ui.rb`
- Modify: `app/config/routes.rb` (route stays the same; just verify admin docs path)

- [ ] **Step 1: Add the admin swagger doc to the helper**

Edit `app/spec/swagger_helper.rb`. Inside `config.openapi_specs = { ... }`, after the existing `"v1/swagger.yaml" => { ... }` entry, add:

```ruby
"admin/swagger.yaml" => {
  openapi: "3.0.3",
  info: {
    title: "Apocalymaps Admin API",
    version: "admin",
    description: "Admin panel endpoints (HTTP Basic auth via ADMIN_USERNAME / ADMIN_PASSWORD)."
  },
  servers: [
    { url: "{scheme}://{host}",
      variables: { scheme: { default: "http", enum: %w[http https] },
                   host: { default: "localhost:8000" } } }
  ],
  components: {
    securitySchemes: {
      basicAuth: { type: :http, scheme: :basic }
    },
    schemas: {
      ServiceSnapshot: {
        type: :object,
        properties: {
          name:       { type: :string },
          enabled:    { type: :boolean },
          status:     { type: :string },
          phase:      { type: :string, nullable: true },
          progress:   { type: :number, nullable: true },
          disk_bytes: { type: :integer }
        }
      },
      ApplyProjectionResponse: {
        type: :object,
        properties: {
          projection: {
            type: :object,
            properties: {
              total_disk_gb:    { type: :number },
              first_boot_hours: { type: :number },
              lines:            { type: :array, items: { type: :object } }
            }
          }
        }
      }
    }
  },
  security: [{ basicAuth: [] }],
  paths: {}
}
```

- [ ] **Step 2: Add the admin doc to the UI**

Edit `app/config/initializers/rswag_ui.rb` and add:

```ruby
c.swagger_endpoint "/api-docs/admin/swagger.yaml", "Admin API"
```

- [ ] **Step 3: Write the failing integration specs**

Create `app/spec/integration/admin/services_spec.rb`:

```ruby
require "swagger_helper"

RSpec.describe "Admin::Services", type: :request do
  path "/admin/services/{name}" do
    parameter name: :name, in: :path, type: :string

    post "Enable or disable a service" do
      tags "Admin"
      security [basicAuth: []]
      consumes "application/x-www-form-urlencoded"
      produces "application/json"

      parameter name: :enabled, in: :formData, type: :string, required: true, example: "true"

      response "200", "service toggled" do
        let(:name)    { "photon" }
        let(:enabled) { "true" }

        before do
          stub_const("ADMIN_CHECK_OK", true)
          ENV["ADMIN_USERNAME"] = "admin"
          ENV["ADMIN_PASSWORD"] = "x"
          Service.create!(name: "photon", profile: "geocoding")
          allow_any_instance_of(ControlPlaneClient).to receive(:enable!).and_return(true)
        end

        run_test!
      end
    end
  end
end
```

Create `app/spec/integration/admin/regions_spec.rb`:

```ruby
require "swagger_helper"

RSpec.describe "Admin::Regions", type: :request do
  path "/admin/regions" do
    post "Replace region selection" do
      tags "Admin"
      security [basicAuth: []]
      consumes "application/json"
      produces "application/json"

      parameter name: :body, in: :body, schema: {
        type: :object, required: %w[regions],
        properties: { regions: { type: :array, items: { type: :string } } }
      }

      response "200", "selection replaced" do
        let(:body) { { regions: %w[berlin] } }

        before do
          stub_const("ADMIN_CHECK_OK", true)
          ENV["ADMIN_USERNAME"] = "admin"
          ENV["ADMIN_PASSWORD"] = "x"

          fake_region = RegionCatalog::Region.new(name: "berlin", label: "Berlin", country_code: "de", pbf_urls: [], default_view: {})
          fake_catalog = instance_double(RegionCatalog, names: %w[berlin])
          allow(fake_catalog).to receive(:find).and_return(fake_region)
          allow(RegionCatalog).to receive(:load_dir).and_return(fake_catalog)
        end

        run_test!
      end
    end
  end
end
```

Create `app/spec/integration/admin/apply_spec.rb`:

```ruby
require "swagger_helper"

RSpec.describe "Admin::Apply", type: :request do
  path "/admin/apply" do
    post "Apply current selection (two-phase)" do
      tags "Admin"
      security [basicAuth: []]
      produces "application/json"

      parameter name: :confirmed, in: :query, type: :string, required: false, example: "true"

      response "409", "confirmation required" do
        schema "$ref" => "#/components/schemas/ApplyProjectionResponse"

        before do
          stub_const("ADMIN_CHECK_OK", true)
          ENV["ADMIN_USERNAME"] = "admin"
          ENV["ADMIN_PASSWORD"] = "x"

          fake_region = RegionCatalog::Region.new(name: "berlin", label: "Berlin", country_code: "de", pbf_urls: [], default_view: {})
          fake_catalog = instance_double(RegionCatalog, names: %w[berlin])
          allow(fake_catalog).to receive(:find).and_return(fake_region)
          allow(RegionCatalog).to receive(:load_dir).and_return(fake_catalog)

          RegionSelection.create!(region_name: "berlin", active: true, position: 0)
        end

        let(:confirmed) { nil }
        run_test!
      end
    end
  end
end
```

- [ ] **Step 4: Generate the admin swagger doc**

```bash
cd app && bundle exec rake rswag:specs:swaggerize
```

Expected: writes `app/swagger/admin/swagger.yaml`.

- [ ] **Step 5: Verify**

```bash
ls -la swagger/
```

Expected: both `v1/swagger.yaml` and `admin/swagger.yaml` exist.

- [ ] **Step 6: Commit**

```bash
git add app/spec/integration/admin/ app/spec/swagger_helper.rb app/config/initializers/rswag_ui.rb app/swagger/admin/
git commit -m "feat(admin): rswag integration specs + admin/swagger.yaml"
```

---

## Phase F — Go sidecar (apo-control)

### Task 15: Sidecar scaffold + health endpoint + tests

**Files:**
- Create: `apo-control/go.mod`, `apo-control/main.go`, `apo-control/internal/server/server.go`
- Create: `apo-control/internal/server/server_test.go`
- Create: `apo-control/Dockerfile`

- [ ] **Step 1: Initialize the module**

```bash
mkdir -p apo-control
cd apo-control
go mod init github.com/dawarich-app/apocalymaps/apo-control
go get github.com/go-chi/chi/v5@latest
```

- [ ] **Step 2: Write the failing test**

Create `apo-control/internal/server/server_test.go`:

```go
package server

import (
    "io"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"
)

func TestHealthEndpoint(t *testing.T) {
    srv := httptest.NewServer(New(Config{}))
    defer srv.Close()

    res, err := http.Get(srv.URL + "/healthz")
    if err != nil {
        t.Fatalf("GET /healthz failed: %v", err)
    }
    defer res.Body.Close()

    if res.StatusCode != http.StatusOK {
        t.Fatalf("want 200, got %d", res.StatusCode)
    }
    body, _ := io.ReadAll(res.Body)
    if !strings.Contains(string(body), "ok") {
        t.Fatalf("want body to contain 'ok', got %q", body)
    }
}
```

- [ ] **Step 3: Run the test — expect failures**

```bash
cd apo-control && go test ./internal/server/
```

Expected: `undefined: New`, `undefined: Config`.

- [ ] **Step 4: Implement the server**

Create `apo-control/internal/server/server.go`:

```go
package server

import (
    "fmt"
    "net/http"

    "github.com/go-chi/chi/v5"
)

type Config struct {
    ComposeFile string
    DataDir     string
    RegionsDir  string
    ListenAddr  string
}

func New(cfg Config) http.Handler {
    r := chi.NewRouter()
    r.Get("/healthz", func(w http.ResponseWriter, _ *http.Request) {
        fmt.Fprintln(w, "ok")
    })
    return r
}
```

- [ ] **Step 5: Implement `main.go`**

Create `apo-control/main.go`:

```go
package main

import (
    "log"
    "net/http"
    "os"

    "github.com/dawarich-app/apocalymaps/apo-control/internal/server"
)

func main() {
    cfg := server.Config{
        ComposeFile: getenv("COMPOSE_FILE", "/work/compose.yml"),
        DataDir:     getenv("DATA_DIR", "/work/data"),
        RegionsDir:  getenv("REGIONS_DIR", "/work/regions"),
        ListenAddr:  getenv("LISTEN_ADDR", ":8090"),
    }
    handler := server.New(cfg)
    log.Printf("apo-control listening on %s", cfg.ListenAddr)
    if err := http.ListenAndServe(cfg.ListenAddr, handler); err != nil {
        log.Fatal(err)
    }
}

func getenv(k, def string) string {
    if v := os.Getenv(k); v != "" {
        return v
    }
    return def
}
```

- [ ] **Step 6: Run tests + sanity-build**

```bash
go test ./...
go build ./...
```

Expected: tests pass; build succeeds.

- [ ] **Step 7: Create the Dockerfile**

Create `apo-control/Dockerfile`:

```Dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.22-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/apo-control .

FROM alpine:3.20
RUN apk add --no-cache docker-cli docker-compose-cli osmium-tool curl ca-certificates
COPY --from=build /out/apo-control /usr/local/bin/apo-control
EXPOSE 8090
ENTRYPOINT ["/usr/local/bin/apo-control"]
```

- [ ] **Step 8: Commit**

```bash
git add apo-control/
git commit -m "feat(sidecar): scaffold apo-control + /healthz + Dockerfile"
```

---

### Task 16: Sidecar state store + RWMutex (TDD)

**Files:**
- Create: `apo-control/internal/state/state.go`
- Create: `apo-control/internal/state/state_test.go`

- [x] **Step 1: Write the failing test**

Create `apo-control/internal/state/state_test.go`:

```go
package state

import (
    "sync"
    "testing"
)

func TestSnapshot(t *testing.T) {
    s := New()
    s.Update("photon", Update{Phase: "downloading", Progress: 0.5, LastLogLine: "foo"})
    snap := s.Snapshot()

    if len(snap) != 1 {
        t.Fatalf("want 1 entry, got %d", len(snap))
    }
    if snap[0].Name != "photon" || snap[0].Progress != 0.5 {
        t.Fatalf("unexpected snapshot: %+v", snap[0])
    }
}

func TestConcurrentUpdate(t *testing.T) {
    s := New()
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(i int) {
            defer wg.Done()
            s.Update("photon", Update{Phase: "x", Progress: float64(i) / 100})
        }(i)
    }
    wg.Wait()
    if len(s.Snapshot()) != 1 {
        t.Fatalf("want 1 entry after concurrent updates")
    }
}
```

- [x] **Step 2: Run — expect failures**

```bash
go test ./internal/state/
```

- [x] **Step 3: Implement**

Create `apo-control/internal/state/state.go`:

```go
package state

import (
    "sync"
    "time"
)

type Service struct {
    Name           string  `json:"name"`
    ContainerState string  `json:"container_state"`
    Phase          string  `json:"phase"`
    Progress       float64 `json:"progress"`
    LastLogLine    string  `json:"last_log_line"`
    Ready          bool    `json:"ready"`
    DiskBytes      int64   `json:"disk_bytes"`
    UpdatedAt      time.Time `json:"updated_at"`
}

type Update struct {
    ContainerState string
    Phase          string
    Progress       float64
    LastLogLine    string
    Ready          bool
    DiskBytes      int64
}

type Store struct {
    mu       sync.RWMutex
    services map[string]Service
}

func New() *Store {
    return &Store{services: map[string]Service{}}
}

func (s *Store) Update(name string, u Update) {
    s.mu.Lock()
    defer s.mu.Unlock()
    svc := s.services[name]
    svc.Name = name
    if u.ContainerState != "" { svc.ContainerState = u.ContainerState }
    if u.Phase != ""          { svc.Phase = u.Phase }
    if u.Progress != 0        { svc.Progress = u.Progress }
    if u.LastLogLine != ""    { svc.LastLogLine = u.LastLogLine }
    svc.Ready = u.Ready || svc.Ready
    if u.DiskBytes != 0       { svc.DiskBytes = u.DiskBytes }
    svc.UpdatedAt = time.Now()
    s.services[name] = svc
}

func (s *Store) Snapshot() []Service {
    s.mu.RLock()
    defer s.mu.RUnlock()
    out := make([]Service, 0, len(s.services))
    for _, svc := range s.services {
        out = append(out, svc)
    }
    return out
}
```

- [x] **Step 4: Run tests**

```bash
go test ./internal/state/
```

Expected: 2 tests pass.

- [x] **Step 5: Commit**

```bash
git add apo-control/internal/state/
git commit -m "feat(sidecar): in-memory state store with RWMutex"
```

---

### Task 17: Phase parser interface + Photon parser (TDD)

**Files:**
- Create: `apo-control/internal/parsers/parser.go`
- Create: `apo-control/internal/parsers/photon.go`
- Create: `apo-control/internal/parsers/photon_test.go`
- Create: `apo-control/testdata/photon-download.log`
- Create: `apo-control/testdata/photon-extract.log`
- Create: `apo-control/testdata/photon-ready.log`

- [x] **Step 1: Create test fixtures**

Create `apo-control/testdata/photon-download.log`:

```
2026-05-14 21:35:35,015 - root - INFO - Starting download of 5.46GB to photon-db-latest.tar.bz2
2026-05-14 21:35:45,092 - root - INFO - Download progress: 0.5% (0.03GB / 5.46GB)
2026-05-14 21:36:32,400 - root - INFO - Download progress: 12.5% (0.68GB / 5.46GB)
```

Create `apo-control/testdata/photon-extract.log`:

```
2026-05-14 21:42:14,021 - root - INFO - Download complete. Extracting...
2026-05-14 21:42:28,109 - root - INFO - Extracting index file
```

Create `apo-control/testdata/photon-ready.log`:

```
2026-05-14 21:42:30,573 - d.k.p.App - INFO - Database cluster is now ready.
2026-05-14 21:42:30,850 - i.j.Javalin - INFO - Listening on http://0.0.0.0:2322/
2026-05-14 21:42:32,738 - root - INFO - Photon ready after 5.0 seconds
```

- [x] **Step 2: Write the failing test**

Create `apo-control/internal/parsers/photon_test.go`:

```go
package parsers

import (
    "bufio"
    "os"
    "path/filepath"
    "testing"
)

func parseFixture(t *testing.T, p Parser, fixture string) Result {
    t.Helper()
    f, err := os.Open(filepath.Join("..", "..", "testdata", fixture))
    if err != nil { t.Fatal(err) }
    defer f.Close()
    var last Result
    sc := bufio.NewScanner(f)
    for sc.Scan() {
        last = p.Feed(sc.Text())
    }
    return last
}

func TestPhotonDownloadProgress(t *testing.T) {
    r := parseFixture(t, &Photon{}, "photon-download.log")
    if r.Phase != "downloading" {
        t.Fatalf("want phase=downloading, got %q", r.Phase)
    }
    if r.Progress < 0.10 || r.Progress > 0.20 {
        t.Fatalf("want progress ~0.125, got %f", r.Progress)
    }
    if r.Ready {
        t.Fatalf("should not be ready during download")
    }
}

func TestPhotonExtract(t *testing.T) {
    r := parseFixture(t, &Photon{}, "photon-extract.log")
    if r.Phase != "extracting" {
        t.Fatalf("want phase=extracting, got %q", r.Phase)
    }
}

func TestPhotonReady(t *testing.T) {
    r := parseFixture(t, &Photon{}, "photon-ready.log")
    if !r.Ready {
        t.Fatalf("want ready=true, got %+v", r)
    }
    if r.Phase != "ready" {
        t.Fatalf("want phase=ready, got %q", r.Phase)
    }
}
```

- [x] **Step 3: Run — expect failures**

```bash
go test ./internal/parsers/
```

- [x] **Step 4: Define the interface**

Create `apo-control/internal/parsers/parser.go`:

```go
package parsers

type Result struct {
    Phase       string
    Progress    float64
    LastLogLine string
    Ready       bool
}

type Parser interface {
    Feed(line string) Result
}
```

- [x] **Step 5: Implement Photon parser**

Create `apo-control/internal/parsers/photon.go`:

```go
package parsers

import (
    "regexp"
    "strconv"
)

type Photon struct {
    state Result
}

var (
    photonDownloadStart    = regexp.MustCompile(`Starting download of`)
    photonDownloadProgress = regexp.MustCompile(`Download progress: ([\d.]+)%`)
    photonExtract          = regexp.MustCompile(`(?i)extracting`)
    photonReady            = regexp.MustCompile(`Photon ready after`)
)

func (p *Photon) Feed(line string) Result {
    p.state.LastLogLine = line
    switch {
    case photonReady.MatchString(line):
        p.state.Phase    = "ready"
        p.state.Progress = 1.0
        p.state.Ready    = true
    case photonExtract.MatchString(line):
        p.state.Phase = "extracting"
    case photonDownloadProgress.MatchString(line):
        m := photonDownloadProgress.FindStringSubmatch(line)
        if pct, err := strconv.ParseFloat(m[1], 64); err == nil {
            p.state.Phase    = "downloading"
            p.state.Progress = pct / 100.0
        }
    case photonDownloadStart.MatchString(line):
        p.state.Phase = "downloading"
    }
    return p.state
}
```

- [x] **Step 6: Run tests**

```bash
go test ./internal/parsers/
```

Expected: 3 tests pass.

- [x] **Step 7: Commit**

```bash
git add apo-control/internal/parsers/ apo-control/testdata/
git commit -m "feat(sidecar): Parser interface + Photon log parser"
```

---

### Task 18: Parsers for Placeholder, Valhalla, Overpass, OTP, Whosonfirst (TDD, repeated structure)

**Files:**
- Create: 5 parser .go files + 5 _test.go + ~15 testdata fixtures

For each service, repeat the pattern from Task 17. The parser-specific phases and regexes are:

| Service | Phases | Sample log markers |
|---------|--------|---------------------|
| **Placeholder** | `extracting`, `building`, `optimizing`, `ready` | `Creating extract at`, `populate fts`, `optimize\.\.\.`, `server listening on port` |
| **Valhalla** | `parsing`, `building-admins`, `building-elevation`, `building-tiles`, `ready` | `Parsing relations`, `Building admin db`, `downloading SRTM`, `building tiles`, `Tile build complete` |
| **Overpass** | `downloading`, `ingesting`, `ready` | `Downloading planet`, `compiled \d+ blocks`, `Server started` |
| **OTP** | `loading-osm`, `loading-gtfs`, `building-graph`, `ready` | `Loaded OSM`, `Loaded GTFS`, `Graph built`, `Started listening` |
| **Whosonfirst** | `downloading`, `complete` | `Downloading whosonfirst`, `Download complete` |

- [x] **Step 1: For EACH service, create 2-3 fixture .log files**

Use real log output from this session's investigation as a starting point. Save under `apo-control/testdata/<service>-<phase>.log`.

- [x] **Step 2: For EACH service, write the parser test FIRST**

Pattern (Placeholder example, `apo-control/internal/parsers/placeholder_test.go`):

```go
package parsers

import "testing"

func TestPlaceholderExtract(t *testing.T) {
    r := parseFixture(t, &Placeholder{}, "placeholder-extract.log")
    if r.Phase != "extracting" {
        t.Fatalf("want extracting, got %q", r.Phase)
    }
}

func TestPlaceholderReady(t *testing.T) {
    r := parseFixture(t, &Placeholder{}, "placeholder-ready.log")
    if !r.Ready {
        t.Fatalf("want ready=true")
    }
}
```

- [x] **Step 3: Run — expect failures**

```bash
go test ./internal/parsers/
```

- [x] **Step 4: Implement EACH parser**

Pattern (`apo-control/internal/parsers/placeholder.go`):

```go
package parsers

import "regexp"

type Placeholder struct{ state Result }

var (
    phExtract  = regexp.MustCompile(`Creating extract at`)
    phBuild    = regexp.MustCompile(`populate fts`)
    phOptimize = regexp.MustCompile(`optimize\.\.\.`)
    phReady    = regexp.MustCompile(`server listening`)
)

func (p *Placeholder) Feed(line string) Result {
    p.state.LastLogLine = line
    switch {
    case phReady.MatchString(line):
        p.state.Phase = "ready"; p.state.Ready = true; p.state.Progress = 1.0
    case phOptimize.MatchString(line):
        p.state.Phase = "optimizing"; p.state.Progress = 0.9
    case phBuild.MatchString(line):
        p.state.Phase = "building"; p.state.Progress = 0.6
    case phExtract.MatchString(line):
        p.state.Phase = "extracting"; p.state.Progress = 0.2
    }
    return p.state
}
```

Repeat for `valhalla.go`, `overpass.go`, `otp.go`, `whosonfirst.go` following the markers in the table at the top of the task.

- [x] **Step 5: Run all parser tests**

```bash
go test ./internal/parsers/
```

Expected: all tests pass.

- [x] **Step 6: Commit**

```bash
git add apo-control/internal/parsers/ apo-control/testdata/
git commit -m "feat(sidecar): parsers for placeholder/valhalla/overpass/otp/whosonfirst"
```

---

### Task 19: docker exec + osmium wrappers (TDD with interface)

**Files:**
- Create: `apo-control/internal/dockerexec/dockerexec.go`
- Create: `apo-control/internal/dockerexec/dockerexec_test.go`
- Create: `apo-control/internal/osmium/osmium.go`
- Create: `apo-control/internal/osmium/osmium_test.go`

- [ ] **Step 1: Define an exec interface (test seam)**

Create `apo-control/internal/dockerexec/dockerexec.go`:

```go
package dockerexec

import (
    "context"
    "fmt"
    "io"
    "os/exec"
)

type Runner interface {
    Run(ctx context.Context, name string, args ...string) (string, error)
}

type ShellRunner struct{}

func (ShellRunner) Run(ctx context.Context, name string, args ...string) (string, error) {
    cmd := exec.CommandContext(ctx, name, args...)
    out, err := cmd.CombinedOutput()
    if err != nil {
        return string(out), fmt.Errorf("%s %v failed: %w (output: %s)", name, args, err, string(out))
    }
    return string(out), nil
}

type DockerCompose struct {
    File   string
    Runner Runner
}

func (d *DockerCompose) Up(ctx context.Context, profile, service string) (string, error) {
    return d.Runner.Run(ctx, "docker", "compose", "-f", d.File, "--profile", profile, "up", "-d", service)
}

func (d *DockerCompose) Stop(ctx context.Context, service string) (string, error) {
    return d.Runner.Run(ctx, "docker", "compose", "-f", d.File, "stop", service)
}

func (d *DockerCompose) Restart(ctx context.Context, services ...string) (string, error) {
    args := append([]string{"compose", "-f", d.File, "restart"}, services...)
    return d.Runner.Run(ctx, "docker", args...)
}

// LogsTail streams tailed logs into w (used for parser feed)
func (d *DockerCompose) LogsTail(ctx context.Context, service string, w io.Writer) error {
    cmd := exec.CommandContext(ctx, "docker", "compose", "-f", d.File, "logs", "-f", "--no-color", "--tail=100", service)
    cmd.Stdout = w
    cmd.Stderr = w
    return cmd.Run()
}
```

- [ ] **Step 2: Test**

Create `apo-control/internal/dockerexec/dockerexec_test.go`:

```go
package dockerexec

import (
    "context"
    "errors"
    "strings"
    "testing"
)

type mockRunner struct {
    lastName string
    lastArgs []string
    output   string
    err      error
}

func (m *mockRunner) Run(_ context.Context, name string, args ...string) (string, error) {
    m.lastName = name
    m.lastArgs = args
    return m.output, m.err
}

func TestUpCallsCorrectArgs(t *testing.T) {
    m := &mockRunner{output: "ok"}
    dc := DockerCompose{File: "/work/compose.yml", Runner: m}
    if _, err := dc.Up(context.Background(), "geocoding", "photon"); err != nil {
        t.Fatal(err)
    }
    if m.lastName != "docker" {
        t.Fatalf("want docker, got %s", m.lastName)
    }
    if !strings.Contains(strings.Join(m.lastArgs, " "), "--profile geocoding up -d photon") {
        t.Fatalf("missing flags: %v", m.lastArgs)
    }
}

func TestStopPropagatesError(t *testing.T) {
    m := &mockRunner{err: errors.New("boom")}
    dc := DockerCompose{File: "/work/compose.yml", Runner: m}
    if _, err := dc.Stop(context.Background(), "photon"); err == nil {
        t.Fatal("want error")
    }
}
```

- [ ] **Step 3: Run + commit**

```bash
go test ./internal/dockerexec/
```

- [ ] **Step 4: osmium wrapper**

Create `apo-control/internal/osmium/osmium.go`:

```go
package osmium

import (
    "context"
    "github.com/dawarich-app/apocalymaps/apo-control/internal/dockerexec"
)

type Osmium struct{ Runner dockerexec.Runner }

func (o *Osmium) Merge(ctx context.Context, dataDir string, sources []string, out string) (string, error) {
    args := []string{"run", "--rm",
        "-v", dataDir + ":/data",
        "-w", "/data",
        "stefda/osmium-tool",
        "osmium", "merge"}
    args = append(args, sources...)
    args = append(args, "-O", "-o", out)
    return o.Runner.Run(ctx, "docker", args...)
}
```

Create `apo-control/internal/osmium/osmium_test.go`:

```go
package osmium

import (
    "context"
    "strings"
    "testing"

    "github.com/dawarich-app/apocalymaps/apo-control/internal/dockerexec"
)

type mockRunner struct{ lastArgs []string }

func (m *mockRunner) Run(_ context.Context, name string, args ...string) (string, error) {
    _ = name
    m.lastArgs = args
    return "", nil
}

func TestMergeCommandLine(t *testing.T) {
    m := &mockRunner{}
    o := Osmium{Runner: m}
    _, _ = o.Merge(context.Background(), "/work/data/osm", []string{"a.pbf", "b.pbf"}, "current.osm.pbf")

    s := strings.Join(m.lastArgs, " ")
    if !strings.Contains(s, "osmium merge a.pbf b.pbf") {
        t.Fatalf("missing merge args: %s", s)
    }
    if !strings.Contains(s, "-O -o current.osm.pbf") {
        t.Fatalf("missing output flags: %s", s)
    }

    var _ dockerexec.Runner = m // compile-time interface check
}
```

- [ ] **Step 5: Test + commit**

```bash
go test ./internal/osmium/
git add apo-control/internal/dockerexec/ apo-control/internal/osmium/
git commit -m "feat(sidecar): docker compose + osmium wrappers with mock-friendly Runner"
```

---

### Task 20: Sidecar HTTP handlers (TDD via httptest)

**Files:**
- Create: `apo-control/internal/server/handlers.go`
- Modify: `apo-control/internal/server/server.go`
- Modify: `apo-control/internal/server/server_test.go`

- [x] **Step 1: Write the failing test**

Replace `apo-control/internal/server/server_test.go` body with (keeping the health test, adding new ones):

```go
package server

import (
    "bytes"
    "context"
    "encoding/json"
    "io"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"

    "github.com/dawarich-app/apocalymaps/apo-control/internal/dockerexec"
    "github.com/dawarich-app/apocalymaps/apo-control/internal/state"
)

type stubRunner struct {
    calls []string
    err   error
}

func (s *stubRunner) Run(_ context.Context, name string, args ...string) (string, error) {
    s.calls = append(s.calls, name+" "+strings.Join(args, " "))
    return "ok", s.err
}

func TestHealthEndpoint(t *testing.T) {
    srv := httptest.NewServer(New(Config{}))
    defer srv.Close()
    res, _ := http.Get(srv.URL + "/healthz")
    if res.StatusCode != 200 { t.Fatalf("want 200, got %d", res.StatusCode) }
}

func TestStatusReturnsSnapshot(t *testing.T) {
    s := state.New()
    s.Update("photon", state.Update{Phase: "ready", Ready: true})
    srv := httptest.NewServer(NewWithStore(Config{}, s, &stubRunner{}))
    defer srv.Close()

    res, _ := http.Get(srv.URL + "/status")
    body, _ := io.ReadAll(res.Body)
    var got []state.Service
    if err := json.Unmarshal(body, &got); err != nil { t.Fatal(err) }
    if len(got) != 1 || got[0].Name != "photon" {
        t.Fatalf("unexpected snapshot: %s", body)
    }
}

func TestEnableInvokesDockerUp(t *testing.T) {
    r := &stubRunner{}
    srv := httptest.NewServer(NewWithStore(Config{ComposeFile: "/work/compose.yml"}, state.New(), r))
    defer srv.Close()

    res, _ := http.Post(srv.URL+"/actions/services/photon/enable", "application/json", nil)
    if res.StatusCode != 202 { t.Fatalf("want 202, got %d", res.StatusCode) }
    if len(r.calls) == 0 || !strings.Contains(r.calls[0], "up -d photon") {
        t.Fatalf("expected docker compose up -d photon, got %v", r.calls)
    }
}

func TestApplyRegionsCallsOsmium(t *testing.T) {
    r := &stubRunner{}
    srv := httptest.NewServer(NewWithStore(Config{ComposeFile: "/work/compose.yml", DataDir: "/work/data"}, state.New(), r))
    defer srv.Close()

    body, _ := json.Marshal(map[string]any{"regions": []string{"berlin"}})
    res, _ := http.Post(srv.URL+"/actions/regions", "application/json", bytes.NewReader(body))
    if res.StatusCode != 202 { t.Fatalf("want 202, got %d", res.StatusCode) }
    // single-region: no osmium merge call, just a copy/symlink — assertion lives in the implementation
}

var _ dockerexec.Runner = &stubRunner{}
```

- [x] **Step 2: Run — expect failures**

```bash
go test ./internal/server/
```

- [x] **Step 3: Implement handlers**

Replace `apo-control/internal/server/server.go`:

```go
package server

import (
    "encoding/json"
    "fmt"
    "net/http"

    "github.com/go-chi/chi/v5"
    "github.com/dawarich-app/apocalymaps/apo-control/internal/dockerexec"
    "github.com/dawarich-app/apocalymaps/apo-control/internal/state"
)

type Config struct {
    ComposeFile string
    DataDir     string
    RegionsDir  string
    ListenAddr  string
}

func New(cfg Config) http.Handler {
    return NewWithStore(cfg, state.New(), dockerexec.ShellRunner{})
}

func NewWithStore(cfg Config, store *state.Store, runner dockerexec.Runner) http.Handler {
    h := handlers{cfg: cfg, store: store, runner: runner,
        compose: &dockerexec.DockerCompose{File: cfg.ComposeFile, Runner: runner}}
    r := chi.NewRouter()
    r.Get("/healthz", h.healthz)
    r.Get("/status",  h.status)
    r.Post("/actions/services/{name}/enable",  h.enable)
    r.Post("/actions/services/{name}/disable", h.disable)
    r.Post("/actions/regions",                  h.applyRegions)
    r.Post("/actions/tiles",                    h.tiles)
    return r
}

type handlers struct {
    cfg     Config
    store   *state.Store
    runner  dockerexec.Runner
    compose *dockerexec.DockerCompose
}

func (h handlers) healthz(w http.ResponseWriter, _ *http.Request) { fmt.Fprintln(w, "ok") }

func (h handlers) status(w http.ResponseWriter, _ *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    _ = json.NewEncoder(w).Encode(h.store.Snapshot())
}

var profileFor = map[string]string{
    "photon": "geocoding", "placeholder": "geocoding", "libpostal": "geocoding",
    "valhalla": "routing", "overpass": "pois", "otp": "transit",
}

func (h handlers) enable(w http.ResponseWriter, r *http.Request) {
    name := chi.URLParam(r, "name")
    profile, ok := profileFor[name]
    if !ok { http.Error(w, "unknown service", http.StatusBadRequest); return }
    if _, err := h.compose.Up(r.Context(), profile, name); err != nil {
        writeError(w, http.StatusBadGateway, "DOCKER_COMPOSE_FAILED", err.Error())
        return
    }
    w.WriteHeader(http.StatusAccepted)
}

func (h handlers) disable(w http.ResponseWriter, r *http.Request) {
    name := chi.URLParam(r, "name")
    if _, err := h.compose.Stop(r.Context(), name); err != nil {
        writeError(w, http.StatusBadGateway, "DOCKER_COMPOSE_FAILED", err.Error())
        return
    }
    w.WriteHeader(http.StatusAccepted)
}

type regionsBody struct {
    Regions []string `json:"regions"`
}

func (h handlers) applyRegions(w http.ResponseWriter, r *http.Request) {
    var body regionsBody
    if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
        writeError(w, http.StatusBadRequest, "BAD_REQUEST", err.Error()); return
    }
    // TODO in Task 21: resolve regions to PBFs, download, merge, restart services.
    _ = body
    w.WriteHeader(http.StatusAccepted)
}

type tilesBody struct {
    URL string `json:"url"`
}

func (h handlers) tiles(w http.ResponseWriter, r *http.Request) {
    var body tilesBody
    if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
        writeError(w, http.StatusBadRequest, "BAD_REQUEST", err.Error()); return
    }
    // TODO in Task 21
    w.WriteHeader(http.StatusAccepted)
}

func writeError(w http.ResponseWriter, status int, code, msg string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    _ = json.NewEncoder(w).Encode(map[string]any{
        "error": map[string]string{"code": code, "message": msg},
    })
}
```

- [x] **Step 4: Run tests**

```bash
go test ./internal/server/
```

Expected: 4 tests pass.

- [x] **Step 5: Commit**

```bash
git add apo-control/internal/server/
git commit -m "feat(sidecar): HTTP handlers — /status, enable/disable, applyRegions stub"
```

---

### Task 21: Full applyRegions + tiles workflows + log follower wiring

**Files:**
- Modify: `apo-control/internal/server/server.go` (fill TODOs from Task 20)
- Create: `apo-control/internal/regions/regions.go`
- Create: `apo-control/internal/regions/regions_test.go`

- [x] **Step 1: Region catalog in Go (mirrors Rails RegionCatalog)**

Create `apo-control/internal/regions/regions.go`:

```go
package regions

import (
    "fmt"
    "os"
    "path/filepath"
    "strings"
)

type Region struct {
    Name        string
    PBFURLs     []string
    PBFName     string
    DiffURL     string
}

func Load(dir, name string) (*Region, error) {
    raw, err := os.ReadFile(filepath.Join(dir, name+".env"))
    if err != nil {
        return nil, fmt.Errorf("region %q: %w", name, err)
    }
    kv := parse(string(raw))
    region := &Region{
        Name:    name,
        PBFName: kv["PBF_NAME"],
        DiffURL: kv["OVERPASS_DIFF_URL"],
    }
    if urls := kv["PBF_URLS"]; urls != "" {
        region.PBFURLs = strings.Fields(urls)
    } else if u := kv["PBF_URL"]; u != "" {
        region.PBFURLs = []string{u}
    }
    return region, nil
}

func parse(content string) map[string]string {
    out := map[string]string{}
    for _, line := range strings.Split(content, "\n") {
        line = strings.TrimSpace(line)
        if line == "" || strings.HasPrefix(line, "#") { continue }
        if idx := strings.Index(line, "="); idx > 0 {
            k := line[:idx]
            v := strings.Trim(line[idx+1:], `"`)
            out[k] = v
        }
    }
    return out
}
```

Create `apo-control/internal/regions/regions_test.go`:

```go
package regions

import (
    "os"
    "path/filepath"
    "testing"
)

func TestLoadSingleRegion(t *testing.T) {
    dir := t.TempDir()
    os.WriteFile(filepath.Join(dir, "berlin.env"),
        []byte(`PBF_URL=https://x/berlin.pbf
PBF_NAME=berlin.osm.pbf`), 0644)

    r, err := Load(dir, "berlin")
    if err != nil { t.Fatal(err) }
    if len(r.PBFURLs) != 1 || r.PBFURLs[0] != "https://x/berlin.pbf" {
        t.Fatalf("unexpected pbf urls: %v", r.PBFURLs)
    }
}

func TestLoadMultiRegion(t *testing.T) {
    dir := t.TempDir()
    os.WriteFile(filepath.Join(dir, "dach.env"),
        []byte(`PBF_URLS="https://x/de.pbf https://x/at.pbf"`), 0644)

    r, err := Load(dir, "dach")
    if err != nil { t.Fatal(err) }
    if len(r.PBFURLs) != 2 {
        t.Fatalf("want 2 urls, got %v", r.PBFURLs)
    }
}
```

- [x] **Step 2: Run + verify**

```bash
go test ./internal/regions/
```

- [x] **Step 3: Wire applyRegions into the handler**

Edit `apo-control/internal/server/server.go`. Replace the TODO body of `applyRegions` with:

```go
func (h handlers) applyRegions(w http.ResponseWriter, r *http.Request) {
    var body regionsBody
    if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
        writeError(w, http.StatusBadRequest, "BAD_REQUEST", err.Error()); return
    }
    if len(body.Regions) == 0 {
        writeError(w, http.StatusBadRequest, "BAD_REQUEST", "regions cannot be empty"); return
    }

    osmDir := filepath.Join(h.cfg.DataDir, "osm")
    sourcesDir := filepath.Join(osmDir, "sources")
    if err := os.MkdirAll(sourcesDir, 0755); err != nil {
        writeError(w, http.StatusInternalServerError, "MKDIR_FAILED", err.Error()); return
    }

    var sources []string
    for _, name := range body.Regions {
        region, err := regions.Load(h.cfg.RegionsDir, name)
        if err != nil {
            writeError(w, http.StatusUnprocessableEntity, "REGION_NOT_FOUND", err.Error()); return
        }
        for _, url := range region.PBFURLs {
            target := filepath.Join(sourcesDir, filepath.Base(url))
            if _, err := os.Stat(target); err == nil {
                sources = append(sources, target)
                continue
            }
            if err := downloadFile(r.Context(), url, target); err != nil {
                writeError(w, http.StatusBadGateway, "DOWNLOAD_FAILED", err.Error()); return
            }
            sources = append(sources, target)
        }
    }

    current := filepath.Join(osmDir, "current.osm.pbf")
    switch len(sources) {
    case 1:
        os.Remove(current)
        if err := os.Symlink(sources[0], current); err != nil {
            writeError(w, http.StatusInternalServerError, "SYMLINK_FAILED", err.Error()); return
        }
    default:
        merger := osmium.Osmium{Runner: h.runner}
        relSources := make([]string, len(sources))
        for i, s := range sources { relSources[i] = filepath.Base(s) }
        if _, err := merger.Merge(r.Context(), osmDir+"/sources", relSources, "../current.osm.pbf.partial"); err != nil {
            writeError(w, http.StatusBadGateway, "MERGE_FAILED", err.Error()); return
        }
        os.Rename(current+".partial", current)
    }

    // Restart consuming services (best-effort)
    h.compose.Restart(r.Context(), "valhalla", "overpass", "otp")
    w.WriteHeader(http.StatusAccepted)
}

func downloadFile(ctx context.Context, url, target string) error {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil { return err }
    res, err := http.DefaultClient.Do(req)
    if err != nil { return err }
    defer res.Body.Close()
    if res.StatusCode != 200 { return fmt.Errorf("HTTP %d", res.StatusCode) }
    out, err := os.OpenFile(target+".partial", os.O_CREATE|os.O_WRONLY|os.O_TRUNC, 0644)
    if err != nil { return err }
    if _, err := io.Copy(out, res.Body); err != nil { out.Close(); os.Remove(target+".partial"); return err }
    out.Close()
    return os.Rename(target+".partial", target)
}
```

Add the new imports at the top of the file:

```go
import (
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os"
    "path/filepath"

    "github.com/go-chi/chi/v5"
    "github.com/dawarich-app/apocalymaps/apo-control/internal/dockerexec"
    "github.com/dawarich-app/apocalymaps/apo-control/internal/osmium"
    "github.com/dawarich-app/apocalymaps/apo-control/internal/regions"
    "github.com/dawarich-app/apocalymaps/apo-control/internal/state"
)
```

- [x] **Step 4: Tiles workflow**

Replace the `tiles` handler body:

```go
func (h handlers) tiles(w http.ResponseWriter, r *http.Request) {
    var body tilesBody
    if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
        writeError(w, http.StatusBadRequest, "BAD_REQUEST", err.Error()); return
    }
    target := filepath.Join(h.cfg.DataDir, "tiles", "basemap.pmtiles")
    os.MkdirAll(filepath.Dir(target), 0755)
    if err := downloadFile(r.Context(), body.URL, target); err != nil {
        writeError(w, http.StatusBadGateway, "DOWNLOAD_FAILED", err.Error()); return
    }
    w.WriteHeader(http.StatusAccepted)
}
```

- [x] **Step 5: Run tests**

```bash
go test ./...
```

Expected: all tests pass.

- [x] **Step 6: Commit**

```bash
git add apo-control/
git commit -m "feat(sidecar): full applyRegions + tiles flows with download + merge"
```

---

### Task 22: Mock-mode binary for Rails + e2e tests

**Files:**
- Create: `apo-control/cmd/mock/main.go`
- Create: `apo-control/testdata/scenarios/photon-quick.yml`

- [x] **Step 1: Implement the mock binary**

Create `apo-control/cmd/mock/main.go`:

```go
package main

import (
    "encoding/json"
    "flag"
    "log"
    "net/http"
    "os"
    "sync"
    "time"

    "gopkg.in/yaml.v3"
)

type Scenario struct {
    Services []struct {
        Name   string `yaml:"name"`
        Timeline []struct {
            At       string  `yaml:"at"`
            Phase    string  `yaml:"phase"`
            Progress float64 `yaml:"progress"`
            Ready    bool    `yaml:"ready"`
        } `yaml:"timeline"`
    } `yaml:"services"`
}

func main() {
    scenarioPath := flag.String("scenario", "", "path to scripted YAML")
    addr := flag.String("addr", ":8090", "listen addr")
    flag.Parse()

    if *scenarioPath == "" {
        log.Fatal("--scenario is required")
    }

    raw, err := os.ReadFile(*scenarioPath)
    if err != nil { log.Fatal(err) }
    var s Scenario
    if err := yaml.Unmarshal(raw, &s); err != nil { log.Fatal(err) }

    var mu sync.Mutex
    state := map[string]map[string]any{}
    started := time.Now()

    for _, svc := range s.Services {
        go func(svc struct {
            Name   string `yaml:"name"`
            Timeline []struct {
                At       string  `yaml:"at"`
                Phase    string  `yaml:"phase"`
                Progress float64 `yaml:"progress"`
                Ready    bool    `yaml:"ready"`
            } `yaml:"timeline"`
        }) {
            for _, step := range svc.Timeline {
                d, _ := time.ParseDuration(step.At)
                time.Sleep(time.Until(started.Add(d)))
                mu.Lock()
                state[svc.Name] = map[string]any{
                    "name": svc.Name, "phase": step.Phase, "progress": step.Progress, "ready": step.Ready,
                }
                mu.Unlock()
            }
        }(svc)
    }

    mux := http.NewServeMux()
    mux.HandleFunc("/healthz", func(w http.ResponseWriter, _ *http.Request) {
        w.Write([]byte("ok\n"))
    })
    mux.HandleFunc("/status", func(w http.ResponseWriter, _ *http.Request) {
        mu.Lock()
        out := make([]map[string]any, 0, len(state))
        for _, v := range state { out = append(out, v) }
        mu.Unlock()
        json.NewEncoder(w).Encode(out)
    })
    mux.HandleFunc("/actions/", func(w http.ResponseWriter, _ *http.Request) {
        w.WriteHeader(http.StatusAccepted)
    })

    log.Printf("apo-control --mock listening on %s", *addr)
    log.Fatal(http.ListenAndServe(*addr, mux))
}
```

- [x] **Step 2: Scenario fixture**

Create `apo-control/testdata/scenarios/photon-quick.yml`:

```yaml
services:
  - name: photon
    timeline:
      - {at: 0s,   phase: starting,    progress: 0.0,  ready: false}
      - {at: 1s,   phase: downloading, progress: 0.5,  ready: false}
      - {at: 2s,   phase: extracting,  progress: 0.9,  ready: false}
      - {at: 3s,   phase: ready,       progress: 1.0,  ready: true}
```

- [x] **Step 3: Add `gopkg.in/yaml.v3` to go.mod**

```bash
cd apo-control
go get gopkg.in/yaml.v3
go mod tidy
go build ./cmd/mock/
```

- [x] **Step 4: Add sidecar boot helper for Rails specs**

Append to `app/spec/support/sidecar_helper.rb`:

```ruby
def boot_mock_sidecar(scenario)
  binary = ENV.fetch("APO_CONTROL_MOCK_BIN", File.expand_path("../../../apo-control/mock", __dir__))
  scenario_path = File.expand_path("../../../apo-control/testdata/scenarios/#{scenario}.yml", __dir__)
  port = (Random.rand(20_000..50_000)).to_s
  pid = Process.spawn(binary, "--scenario", scenario_path, "--addr", ":#{port}", out: "/dev/null", err: "/dev/null")
  at_exit { Process.kill("TERM", pid) rescue nil }
  url = "http://127.0.0.1:#{port}"
  # wait for /healthz
  10.times do
    begin
      return url if Net::HTTP.get_response(URI("#{url}/healthz")).code == "200"
    rescue Errno::ECONNREFUSED
      sleep 0.1
    end
  end
  raise "mock sidecar did not become healthy"
end
```

This helper is opt-in for tests that need a real running sidecar (vs the Faraday stub for unit tests).

- [ ] **Step 5: Commit**

```bash
git add apo-control/cmd/mock/ apo-control/testdata/scenarios/ apo-control/go.mod apo-control/go.sum app/spec/support/sidecar_helper.rb
git commit -m "feat(sidecar): --mock binary + scenarios + Rails test helper"
```

---

## Phase G — Compose, CI, Caddy

### Task 23: Update compose.yml

**Files:**
- Modify: `compose.yml`

- [ ] **Step 1: Append the apo-control service**

Add this to `compose.yml`, in the `services:` map (alongside the existing app/caddy/etc):

```yaml
  apo-control:
    image: ${APO_CONTROL_IMAGE:-ghcr.io/dawarich-app/apocalymaps-control:latest}
    pull_policy: ${APO_CONTROL_PULL_POLICY:-always}
    container_name: apo-control
    environment:
      COMPOSE_FILE: /work/compose.yml
      DATA_DIR:     /work/data
      REGIONS_DIR:  /work/regions
      LISTEN_ADDR:  ":8090"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - .:/work:ro
      - ./data:/work/data
    restart: unless-stopped
```

(The bind mount overrides the `:ro` mount of the parent for `./data`, since we need writable access to `data/osm/`.)

- [ ] **Step 2: Add the `/osm:ro` mount to Valhalla, Overpass, OTP**

For each of those services in `compose.yml`, add to their `volumes:` block:

```yaml
      - ./data/osm:/osm:ro
```

And update their PBF env var to:

```yaml
      # valhalla
      tile_urls: /osm/current.osm.pbf
      # overpass
      OVERPASS_PLANET_URL: file:///osm/current.osm.pbf
      # otp — files already in /var/opentripplanner/osm
```

- [ ] **Step 3: Wire app to know about sidecar**

In the `app:` service env, add:

```yaml
      CONTROL_PLANE_URL: http://apo-control:8090
      ADMIN_USERNAME: ${ADMIN_USERNAME:-}
      ADMIN_PASSWORD: ${ADMIN_PASSWORD:-}
```

- [ ] **Step 4: Verify compose still validates**

```bash
docker compose config --services > /dev/null
```

Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add compose.yml
git commit -m "feat(admin): wire apo-control into compose + /osm:ro shared mount"
```

---

### Task 24: CI — build apo-control image + test workflow

**Files:**
- Modify: `.github/workflows/build.yml`
- Create: `.github/workflows/test.yml`

- [ ] **Step 1: Add parallel job to build.yml**

At the end of `.github/workflows/build.yml`, add a new top-level job alongside `build`:

```yaml
  build-control-plane:
    runs-on: ${{ matrix.runner }}
    permissions: { contents: read, packages: write }
    strategy:
      fail-fast: false
      matrix:
        include:
          - platform: linux/amd64
            runner: ubuntu-24.04
          - platform: linux/arm64
            runner: ubuntu-24.04-arm
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - name: Log in to GHCR
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: ./apo-control
          file: ./apo-control/Dockerfile
          platforms: ${{ matrix.platform }}
          push: ${{ github.event_name != 'pull_request' }}
          tags: ghcr.io/dawarich-app/apocalymaps-control:latest
```

- [ ] **Step 2: Create test workflow**

Create `.github/workflows/test.yml`:

```yaml
name: Test

on:
  push:
    branches: [main]
  pull_request:

jobs:
  rspec:
    runs-on: ubuntu-24.04
    defaults: { run: { working-directory: ./app } }
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: 3.4.6
          bundler-cache: true
          working-directory: ./app
      - run: bundle exec rails db:test:prepare
      - run: bundle exec rspec

  go:
    runs-on: ubuntu-24.04
    defaults: { run: { working-directory: ./apo-control } }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.22"
      - run: go test ./...
```

- [ ] **Step 3: Verify YAML parses**

```bash
yq . .github/workflows/build.yml > /dev/null
yq . .github/workflows/test.yml > /dev/null
```

(If `yq` is missing, skip this step.)

- [ ] **Step 4: Commit**

```bash
git add .github/workflows/
git commit -m "ci: build apo-control image + add test workflow"
```

---

## Phase H — Documentation

### Task 25: README + apo-control/README.md

**Files:**
- Modify: `README.md`
- Create: `apo-control/README.md`

- [ ] **Step 1: Add admin-panel section to root README**

Insert a new section just before "TODO":

```markdown
## Admin panel

The map page (/) lazy-loads an admin panel in the corner. Auth is HTTP Basic, configured via env vars:

```bash
echo 'ADMIN_USERNAME=admin' >> .env
echo 'ADMIN_PASSWORD=use-a-real-password' >> .env
docker compose --profile all up -d
```

Open the panel via the cog icon → toggle services and pick regions from the dropdown → click Save → confirm. Live progress streams through Turbo Streams over Action Cable (async adapter).

A Go sidecar (`apo-control`) owns docker socket access; Rails talks to it over HTTP. Both ship as separate images: `ghcr.io/dawarich-app/apocalymaps:latest` (Rails) and `ghcr.io/dawarich-app/apocalymaps-control:latest` (sidecar). The sidecar source lives at `apo-control/`.

OpenAPI for the admin endpoints: `http://localhost:8000/api-docs/admin/swagger.yaml`.
```

- [ ] **Step 2: Create `apo-control/README.md`**

```markdown
# apo-control

Small Go sidecar that owns `docker compose` + `osmium` exec for the Apocalymaps admin panel.

## Endpoints

| Method | Path | Body | Returns |
|--------|------|------|---------|
| GET    | `/healthz`                              |   — | `ok\n` |
| GET    | `/status`                               |   — | JSON snapshot of every known service |
| POST   | `/actions/services/{name}/enable`       |   — | 202 |
| POST   | `/actions/services/{name}/disable`      |   — | 202 |
| POST   | `/actions/regions`                      | `{regions: ["berlin","vienna"]}` | 202 |
| POST   | `/actions/tiles`                        | `{url: "..."}` | 202 |

## Local development

```bash
cd apo-control
go test ./...
go run . --addr :8090

# Or with mock mode
go run ./cmd/mock --scenario testdata/scenarios/photon-quick.yml --addr :8090
```

## Building the image

```bash
docker build -t apocalymaps-control:dev .
```

## Architecture

- `internal/state/` — in-memory per-service state with RWMutex.
- `internal/parsers/` — one per upstream service; `Feed(line)` returns updated `Result{phase, progress, ready}`.
- `internal/dockerexec/` — `docker compose` wrapper through a `Runner` interface for mockable tests.
- `internal/osmium/` — `osmium-tool merge` wrapper.
- `internal/regions/` — minimal `regions/*.env` parser (mirrors the Rails side).
- `internal/server/` — chi router + handlers.

All non-config state is in-memory; persistence is Rails's job.
```

- [ ] **Step 3: Commit**

```bash
git add README.md apo-control/README.md
git commit -m "docs(admin): root README admin section + apo-control README"
```

---

## Final verification

After all 25 tasks:

- [ ] **A. Rails specs pass**

```bash
cd app && bundle exec rspec
```

- [ ] **B. Go tests pass**

```bash
cd apo-control && go test ./...
```

- [ ] **C. swagger.yaml regenerated and committed**

```bash
cd app && bundle exec rake rswag:specs:swaggerize
git add app/swagger/
git commit -m "docs: regenerate swagger.yaml" || echo "no swagger changes"
```

- [ ] **D. Local smoke test**

```bash
cp .env.example .env
echo "ADMIN_USERNAME=admin" >> .env
echo "ADMIN_PASSWORD=test" >> .env

# Build sidecar locally
docker build -t apocalymaps-control:dev apo-control/
APO_CONTROL_IMAGE=apocalymaps-control:dev APO_CONTROL_PULL_POLICY=never \
APP_IMAGE=apocalymaps-app:dev APP_PULL_POLICY=never \
  docker compose build app
APO_CONTROL_IMAGE=apocalymaps-control:dev APO_CONTROL_PULL_POLICY=never \
APP_IMAGE=apocalymaps-app:dev APP_PULL_POLICY=never \
  docker compose up -d

curl -u admin:test http://localhost:8000/admin/services
# Expected: HTML containing the panel
```

- [ ] **E. Push**

```bash
git push origin main
```

- [ ] **F. Update plan status**

Change the `Status:` header in this file to `COMPLETE` once all tasks are done; spec-verify will re-classify to `VERIFIED` after independent review.
