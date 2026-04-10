# Code Conventions

## Convention 1: Minimize Dependencies

- Push Rails to its limits before adding new gems
- Strong technical or business reason required for any new dependency
- Favor old and reliable over new and flashy

## Convention 2: Skinny Controllers, Fat Models

- Business logic belongs in `app/models/`, NOT `app/services/`
- Use Rails concerns and POROs (Plain Old Ruby Objects) for organization
- Concerns can be "one-off" for a single model — used for organizing traits, not just shared behavior
- Models should answer questions about themselves:

```ruby
# GOOD
account.balance_series

# BAD
AccountSeries.new(account).call
```

## Convention 3: Hotwire-First Frontend

- Native HTML preferred over JS components:
  - `<dialog>` for modals, not custom components
  - `<details><summary>` for disclosures
- Turbo frames for page sections over client-side solutions
- Query params for URL state over localStorage/sessions
- Server-side formatting for currencies, numbers, dates — pass to Stimulus for display only
- Turbo streams to enhance, not depend on
- Client-side JS only where server-side would degrade UX (e.g., bulk selection)
- Always use the `icon` helper in `application_helper.rb` — NEVER use `lucide_icon` directly

## Convention 4: Optimize for Simplicity

- Prioritize good OOP domain design over performance
- Only focus on performance for critical/global areas:
  - Be mindful of large data payloads in global layouts
  - Avoid N+1 queries
  - Use `includes`/`joins` appropriately

## Convention 5: Database vs ActiveRecord Validations

- Simple validations (null checks, unique indexes) → database level
- ActiveRecord validations mirror DB ones for form convenience — prefer client-side form validation when possible
- Complex validations and business logic → ActiveRecord

## Code Organization

```
app/models/              # Domain logic, concerns, POROs
app/models/concerns/     # Shared model concerns (accountable, syncable, monetizable, enrichable)
app/models/account/      # Account-namespaced models (balance, holding calculators)
app/models/provider/     # Data provider system
app/controllers/         # Thin controllers
app/controllers/concerns/ # Controller concerns (authentication, auto_sync, breadcrumbable, etc.)
app/components/DS/       # Design system components (Button, Dialog, Menu, Tabs, etc.)
app/components/UI/       # Domain-specific UI components
app/helpers/             # View helpers (icon helper, form builder, formatters)
app/jobs/                # Sidekiq background jobs
app/views/               # ERB templates
app/javascript/controllers/ # Global Stimulus controllers
```

## Ruby Style

- Rubocop with `rubocop-rails-omakase` base config
- 2-space indentation, spaces (not tabs)
- Double quotes for strings (enforced in ERB lint too)
- UTF-8 charset, LF line endings

## JavaScript Style

- Biome for linting and formatting
- Stimulus controllers: declarative actions, lightweight, <7 targets
- No inline JavaScript — pass data via `data-*-value` attributes

## Naming Conventions

- Components: `ComponentName` suffix (e.g., `ButtonComponent`, `DialogComponent`)
- Partials: underscore prefix (e.g., `_trend_change.html.erb`)
- Shared partials: `app/views/shared/`
- Context-specific partials: in relevant controller view directory
