# Maybe: Project Overview

Maybe is an open-source (AGPLv3) personal finance application built with Ruby on Rails. It allows users to track accounts, transactions, investments, and net worth across multiple financial institutions.

## App Modes

The app runs in two modes controlled by `Rails.application.config.app_mode`:

- **managed** — The Maybe team operates servers for users (SaaS)
- **self_hosted** — Users host on their own infrastructure via Docker Compose

## Tech Stack

- **Framework**: Ruby on Rails 7.2.x, Ruby 3.4.4
- **Database**: PostgreSQL (UUIDs as primary keys)
- **Cache/Jobs**: Redis + Sidekiq (with sidekiq-cron for scheduled jobs)
- **Frontend**: Hotwire (Turbo + Stimulus), TailwindCSS v4.x, ViewComponents, Lucide Icons
- **Asset Pipeline**: Propshaft + importmap-rails
- **Testing**: Minitest + fixtures + mocha (NEVER RSpec or FactoryBot)
- **Linting**: Rubocop (rubocop-rails-omakase), Biome (JS), ERB Lint
- **External Services**: Plaid (bank syncing), Stripe (payments), OpenAI (AI chat), Synth API (market data)
- **Monitoring**: Sentry, Skylight, Logtail, Vernier, rack-mini-profiler
- **API Auth**: Doorkeeper (OAuth2) + API keys with JWT tokens, Rack Attack rate limiting

## Development Commands

```sh
bin/setup              # Initial project setup
bin/dev                # Start Rails + Sidekiq + Tailwind watcher
bin/rails server       # Start Rails server only
bin/rails console      # Open Rails console
bin/rails test         # Run all tests
bin/rails test:system  # Run system tests (slow, use sparingly)
bin/rubocop -f github -a           # Ruby linting with auto-correct
bundle exec erb_lint ./app/**/*.erb -a  # ERB linting with auto-correct
npm run lint:fix       # Fix JS/TS issues
bin/brakeman --no-pager            # Security analysis
bin/rails db:prepare   # Create and migrate database
bin/rails db:migrate   # Run pending migrations
bin/rails db:seed      # Load seed data
rake demo_data:default # Load demo data
```

## Pre-PR Checklist

All of these must pass before opening a PR:

1. `bin/rails test` — unit/integration tests
2. `bin/rails test:system` — system tests (when applicable)
3. `bin/rubocop -f github -a` — Ruby linting
4. `bundle exec erb_lint ./app/**/*.erb -a` — ERB linting
5. `bin/brakeman --no-pager` — security scan

## Default Credentials (dev seed)

- Email: `user@maybe.local`
- Password: `password`

## Prohibited Actions

- Do not run `rails server` (use `bin/dev` instead)
- Do not run `touch tmp/restart.txt`
- Do not run `rails credentials`
- Do not automatically run migrations
- Ignore i18n — hardcode strings in English
