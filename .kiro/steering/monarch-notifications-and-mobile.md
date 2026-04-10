---
inclusion: manual
---

# Notifications, Mobile & Remaining Features

Implements Monarch parity features #18 (Mobile), #20 (Notifications), #21 (PWA), #23 (Tax reporting), #24 (Property value sync), #25 (Crypto connectivity).

## 1. Notification System

### Domain Model

```
Notification (new model, belongs_to :user)
  - notification_type: enum (bill_due, goal_milestone, budget_exceeded,
      unusual_spending, large_transaction, sync_error, weekly_summary)
  - title: string
  - body: text
  - read_at: datetime (nullable)
  - action_url: string (nullable — deep link to relevant page)
  - delivered_via: enum (in_app, email, push)
  - metadata: jsonb
```

### Notification Preferences

```
NotificationPreference (new model, belongs_to :user)
  - notification_type: string (matches Notification types)
  - in_app_enabled: boolean (default true)
  - email_enabled: boolean (default false)
  - push_enabled: boolean (default false)
```

### Notification Triggers

Create `NotificationDispatcher` PORO in `app/models/notification/dispatcher.rb`:

- **bill_due**: Triggered by `RecurringTransaction` when `next_expected_on` is within 3 days. Run daily via Sidekiq cron.
- **goal_milestone**: When goal reaches 25%, 50%, 75%, 100% of target. Checked after each sync.
- **budget_exceeded**: When category spending exceeds budget target. Checked after transaction sync.
- **unusual_spending**: When a single transaction exceeds 2x the average for that merchant/category. Checked during transaction enrichment.
- **large_transaction**: When transaction amount exceeds user-configured threshold. Checked on new transaction creation.
- **sync_error**: When a Sync record fails. Triggered in sync callbacks.
- **weekly_summary**: Triggered by `WeeklySummaryJob`.

### Delivery Channels

- **In-app**: Bell icon in nav bar with unread count badge. Dropdown shows recent notifications. Full notifications page at `/notifications`.
- **Email**: Use existing mailer infrastructure. New `NotificationMailer` with templates per type. Only if SMTP configured and user opted in.
- **Push**: Future — requires mobile app or PWA service worker registration.

### UI

- Bell icon in global nav (use `icon("bell")` helper)
- Unread count badge (Turbo stream updates when new notifications arrive)
- Notification dropdown: last 10 notifications with mark-as-read
- Settings > Notifications page: toggle each type × channel
- Click notification → navigate to `action_url`

## 2. PWA Enhancements

### Current State

Maybe already has a service worker and manifest route (`/service-worker`, `/manifest`).

### Enhancements

- **Offline support**: Cache critical pages (dashboard, transactions list) via service worker. Show cached data when offline with "offline" banner.
- **Home screen install**: Ensure manifest.json has proper `display: standalone`, icons, theme colors, and `start_url`.
- **Push notifications**: Register push subscription via service worker. Store `PushSubscription` on User model. Use Web Push API (web-push gem) to deliver notifications.

### Implementation

```ruby
# Gemfile addition (only if push notifications needed)
gem "web-push"

# app/models/push_subscription.rb
class PushSubscription < ApplicationRecord
  belongs_to :user
  # endpoint: string, p256dh: string, auth: string
end
```

Service worker updates in `app/views/pwa/service_worker.js.erb` for caching strategy.

## 3. Tax-Related Reporting

### Concept

Help users categorize and report tax-relevant transactions.

### Implementation

Add `tax_deductible` boolean to `Category` model. Users can flag categories as tax-relevant (charitable donations, business expenses, medical, etc.).

Add a "Tax Report" saved report type that:

- Filters to tax-deductible categories only
- Groups by category
- Shows totals per category for the tax year
- Exportable as CSV for accountant/tax software import

### Tax Categories (Seed Data)

Provide default tax-relevant categories that users can opt into:

- Charitable Donations
- Medical Expenses
- Business Expenses
- Home Office
- Education
- State/Local Taxes Paid
- Mortgage Interest (auto-detected from Loan accounts)

## 4. Property Value Sync

### Concept

Auto-update property account values from external data sources (similar to Zillow Zestimate).

### Implementation

Register a new provider concept `property_valuation` in `Provider::Registry`:

```ruby
# app/models/provider/concepts/property_valuation.rb
module Provider::Concepts::PropertyValuation
  def fetch_property_value(address:)
    # Returns Provider::ProviderResponse with estimated value
  end
end
```

Add `Address` association to `Property` accountable (address model already exists). During account sync, if property has an address and a property valuation provider is configured, fetch and create a new `Valuation` entry.

### Self-Hosted

For self-hosted users without API access, property values remain manual (existing behavior). The provider is optional.

## 5. Crypto Exchange Connectivity

### Concept

Connect to crypto exchanges (Coinbase, Kraken, etc.) to auto-sync holdings and transactions.

### Implementation

Register crypto exchange providers in `Provider::Registry`:

```ruby
# app/models/provider/concepts/crypto_exchange.rb
module Provider::Concepts::CryptoExchange
  def fetch_balances(api_key:, api_secret:)
  def fetch_transactions(api_key:, api_secret:, since:)
end
```

Store API credentials encrypted in `Setting` model (existing pattern with Active Record Encryption).

Create `CryptoSyncJob` that:

1. Fetches balances from exchange API
2. Creates/updates Holdings on the Crypto account
3. Fetches transactions and creates Trade entries
4. Runs as part of family sync

### Supported Exchanges (Phase 1)

- Coinbase (already partially supported via Plaid)
- Manual API key entry for: Kraken, Binance (read-only API keys)

## Implementation Notes

- Notification system is a new model but follows existing patterns (Sidekiq jobs, mailers)
- PWA enhancements build on existing service worker infrastructure
- Tax reporting is mostly a UI/report feature — minimal model changes
- Property valuation and crypto exchange follow existing Provider pattern
- All new gems (web-push) require strong justification per Convention 1
- Prefer building without new gems where possible (e.g., Web Push can be done with raw HTTP)
- Use functional design tokens for notification badges and status indicators
