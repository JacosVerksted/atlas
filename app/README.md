# Apocalymaps — Rails app

The full-stack Rails 8 app for Apocalymaps. Serves the map UI (Hotwire + Stimulus + MapLibre) and exposes the orchestration API (`/api/search`, `/api/whats-here`, `/api/route`).

## Bootstrap

This directory will be populated by running `rails new` once. Until then, only this README exists.

### Prerequisites

- Ruby 3.4.x (`mise install ruby@3.4.4` or rbenv — keep in sync with `app/.ruby-version` and the `RUBY_VERSION` ARG in `app/Dockerfile`)
- Node.js 22.x LTS (for esbuild)
- Rails 8 (`gem install rails`)

### One-shot scaffold

`app/Dockerfile` is pre-written (Ruby 3.4.x, libpq5, jemalloc, no Thruster — Caddy fronts everything). `rails new` will overwrite it; restore from git after.

```bash
cd atlas
rails new app \
  --css=tailwind \
  --javascript=esbuild \
  --database=sqlite3 \
  --skip-test \
  --skip-system-test \
  --skip-kamal \
  --force

# Restore our customized Dockerfile + this README
git checkout app/Dockerfile app/README.md

cd app
echo "3.4.4" > .ruby-version    # match Dockerfile's RUBY_VERSION
bundle add rspec-rails --group "development, test"
bundle add pundit
bundle add omniauth-google-oauth2 omniauth-github omniauth-rails_csrf_protection
bin/rails generate rspec:install
bin/rails generate authentication

# DaisyUI v5 (Tailwind v4-compatible)
npm install -D daisyui@latest

# Add MapLibre + PMTiles to the JS bundle
npm install maplibre-gl pmtiles
```

In `app/assets/tailwind/application.css` (Tailwind v4 CSS-first config):

```css
@import "tailwindcss";
@plugin "daisyui";
@plugin "daisyui/theme" {
  name: "apocalymaps-light";
  default: true;
}
@plugin "daisyui/theme" {
  name: "apocalymaps-dark";
  prefersdark: true;
}
```

## Structure (target)

```
app/
├── Dockerfile                       # Rails 8 default + tweaks for compose
├── Gemfile
├── config/
│   ├── database.yml                 # SQLite default; DATABASE_URL switches
│   ├── routes.rb
│   └── initializers/
├── app/
│   ├── controllers/
│   │   ├── api/
│   │   │   ├── search_controller.rb
│   │   │   ├── whats_here_controller.rb
│   │   │   └── routes_controller.rb
│   │   ├── home_controller.rb
│   │   ├── collections_controller.rb
│   │   ├── places_controller.rb
│   │   └── sessions_controller.rb
│   ├── models/
│   │   ├── user.rb
│   │   ├── session.rb
│   │   ├── collection.rb            # visibility: private/unlisted/link/public
│   │   ├── collection_share.rb
│   │   └── place.rb
│   ├── policies/
│   │   ├── collection_policy.rb
│   │   └── place_policy.rb
│   ├── services/
│   │   ├── photon_client.rb
│   │   ├── placeholder_client.rb
│   │   ├── libpostal_client.rb
│   │   ├── valhalla_client.rb
│   │   ├── overpass_client.rb
│   │   └── search_orchestrator.rb   # fan-out + merge
│   ├── views/
│   │   ├── layouts/application.html.erb
│   │   ├── home/index.html.erb      # the map page
│   │   ├── collections/
│   │   └── places/
│   └── javascript/
│       ├── application.js
│       ├── controllers/
│       │   ├── map_controller.js    # mounts MapLibre + PMTiles
│       │   ├── search_controller.js # debounced /api/search
│       │   └── route_controller.js
│       └── lib/
│           ├── pmtiles_protocol.js
│           └── maplibre_style.js
├── db/
│   ├── schema.rb
│   └── migrate/
└── spec/
```

## Service URLs (read from ENV)

| Env var | Default in compose | Purpose |
|---------|--------------------|---------|
| `PHOTON_URL` | `http://photon:2322` | Geocoding |
| `PLACEHOLDER_URL` | `http://placeholder:3000` | Admin hierarchy |
| `LIBPOSTAL_URL` | `http://libpostal:4400` | Query parsing |
| `VALHALLA_URL` | `http://valhalla:8002` | Routing + elevation |
| `OVERPASS_URL` | `http://overpass:80` | POI queries |
| `OTP_URL` | `http://otp:8080` | Transit (optional) |

## Database switching

```bash
# SQLite (default)
DATABASE_URL=sqlite3:/data/app.sqlite3

# PostgreSQL
DATABASE_URL=postgres://user:pass@host:5432/apocalymaps
```

Rails 8 + ActiveRecord handles the rest. Migrations are DB-agnostic.

## Tests

```bash
bin/rspec                 # Ruby specs
# E2E via Playwright lives in apocalymaps/e2e/ (TBD)
```
