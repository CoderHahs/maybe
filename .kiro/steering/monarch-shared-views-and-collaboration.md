---
inclusion: manual
---

# Shared Views, Collaboration & Review Workflow

Implements Monarch parity features #4 (Shared views), #8 (Transaction review), #16 (Advisor collaboration).

## 1. Shared Views — Yours / Mine / Ours

### Concept

Partners in a Family can label accounts and transactions as "yours", "mine", or "ours" to see finances from different perspectives without fully merging or separating.

### Domain Changes

Add `ownership` enum to `Account`:

- `shared` (default, "ours")
- `personal` (belongs to a specific user)

Add `owner_id` (references User, nullable) to `Account`. When `ownership: :personal`, `owner_id` identifies which user owns it.

### View Filters

Add a persistent view filter (stored as user preference or query param per Convention 3):

- "All" — shows everything (default, current behavior)
- "Mine" — shows only accounts where `owner_id == Current.user.id` or `ownership == :shared`
- "Partner's" — shows only accounts where `owner_id == partner.id` or `ownership == :shared`
- "Shared only" — shows only `ownership == :shared` accounts

This filter applies globally: dashboard, transactions, net worth, budgets, reports.

### Implementation

```ruby
# app/models/concerns/ownership_filterable.rb
module OwnershipFilterable
  extend ActiveSupport::Concern

  scope :for_view, ->(view, user) {
    case view
    when "mine"    then where(owner_id: [user.id, nil]).or(where(ownership: :shared))
    when "partner" then where.not(owner_id: user.id).or(where(ownership: :shared))
    when "shared"  then where(ownership: :shared)
    else all
    end
  }
end
```

Use query params (`?view=mine`) for state per Convention 3.

### UI

- View switcher in the global header/nav (tabs or dropdown)
- Each account shows an ownership badge (icon or label)
- Account settings page allows setting ownership
- Net worth, budget, and reports all respect the active view filter

## 2. Transaction Review Workflow

### Concept

Users can mark transactions as "reviewed" to track which ones they've looked at. Unreviewed transactions surface as needing attention.

### Domain Changes

Add `reviewed_at` (datetime, nullable) and `reviewed_by_id` (references User, nullable) to `Entry`.

### Logic

```ruby
# In Entry model or concern
scope :unreviewed, -> { where(reviewed_at: nil) }
scope :reviewed, -> { where.not(reviewed_at: nil) }

def mark_reviewed!(user = Current.user)
  update!(reviewed_at: Time.current, reviewed_by_id: user.id)
end

def unmark_reviewed!
  update!(reviewed_at: nil, reviewed_by_id: nil)
end
```

### UI

- Checkbox or icon button on each transaction row to toggle reviewed status
- Filter toggle on transactions index: "Show unreviewed only"
- Dashboard widget: "X unreviewed transactions"
- Bulk review action (select multiple → mark all reviewed)
- Use Stimulus for the toggle interaction (good use case for client-side per Convention 3)

## 3. Financial Advisor / CPA Collaboration

### Concept

Family admin can invite a financial advisor or CPA with read-only access. This extends the existing invitation system.

### Domain Changes

Add `role: :advisor` to the User role enum (alongside `admin` and `member`). Advisors get:

- Read-only access to all family financial data
- Cannot create/edit/delete accounts, transactions, or settings
- Can view reports, net worth, budgets, goals
- Can participate in AI chat for advice

### Implementation

- Extend existing `Invitation` model with a role field
- Add authorization checks in controllers: `before_action :require_write_access` for mutating actions
- Advisor users see a read-only UI (edit buttons hidden, forms disabled)
- No extra cost — included in family subscription (matches Monarch)

## Implementation Notes

- Ownership filter uses query params, not sessions (Convention 3)
- Review workflow uses Stimulus for toggle UX, server for persistence
- All new scopes/methods on existing models (Convention 2)
- Advisor role extends existing auth system, minimal new code
