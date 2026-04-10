---
inclusion: fileMatch
fileMatchPattern: "app/models/**,app/controllers/**"
---

# Models & Controllers

## Model Organization

All business logic lives in `app/models/`. No `app/services/` pattern.

### Concerns

Model concerns in `app/models/concerns/`:

- `Accountable` — shared behavior for account delegated types
- `Syncable` — sync infrastructure for Account, PlaidItem, Family
- `Monetizable` — money/currency handling
- `Enrichable` — data enrichment behavior

Concerns can be one-off (single model) for organizing traits — not just for shared behavior.

### Namespaced Models

Complex models use namespaced directories:

```
app/models/account.rb
app/models/account/
  balance.rb
  balance/base_calculator.rb
  holding/base_calculator.rb
```

### Provider Pattern

```
app/models/provider.rb              # Base class
app/models/provider/registry.rb     # Provider registry
app/models/provider/concepts/       # Concept interfaces
app/models/provider/synth.rb        # Concrete implementation
app/models/exchange_rate/provided.rb # "Provided" concern
```

Domain models access providers through `Provided` concerns, not `Registry` directly.

## Controller Organization

### Base Controller

`ApplicationController` includes these concerns:

- `Authentication` — session-based auth
- `AutoSync` — daily family sync trigger
- `Onboardable` — onboarding flow
- `Localize` — locale handling
- `Invitable` — invitation system
- `SelfHostable` — self-hosting guards
- `StoreLocation` — redirect after login
- `Impersonatable` — admin impersonation
- `Breadcrumbable` — breadcrumb navigation
- `FeatureGuardable` — feature flags
- `Notifiable` — flash notifications
- `RestoreLayoutPreferences` — layout state
- `Pagy::Backend` — pagination

### Controller Concerns

Located in `app/controllers/concerns/`:

- `AccountableResource` — shared account CRUD patterns
- `EntryableResource` — shared entry CRUD patterns
- `Periodable` — date range filtering
- `StreamExtensions` — Turbo stream helpers

### Thin Controllers

Controllers should be thin. Delegate to models:

```ruby
# GOOD
def show
  @account = Current.family.accounts.find(params[:id])
  # Account model handles its own data
end

# BAD
def show
  @account = Current.family.accounts.find(params[:id])
  @balance_series = BalanceService.new(@account).calculate  # No services
end
```

### API Controllers

External API lives in `app/controllers/api/v1/`:

- Auth endpoints (signup, login, refresh)
- Resource endpoints (accounts, transactions, chats, usage)
- Doorkeeper OAuth + API key authentication
- Jbuilder templates for JSON responses

## Database Conventions

- UUIDs as primary keys
- Simple validations (null, unique) at DB level
- Complex validations in ActiveRecord
- Business logic never in the database
- Prefer client-side form validation when possible
