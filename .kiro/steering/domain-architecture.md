# Domain Architecture

## Core Identity

- `Current.user` — always use this, NEVER `current_user`
- `Current.family` — always use this, NEVER `current_family`
- `Family` — top-level entity. Owns accounts, subscriptions, preferences, currency setting
- `User` — belongs to a Family. Role is `admin` (head of household) or `member`. Sessions are at user level.

## Currency

Each Family selects a base currency. All records are normalized to this currency via `ExchangeRate` records. The `Money` class handles conversion and formatting.

## Accounts (Delegated Type)

`Account` is the central domain model. It uses Rails delegated types with `classification` of `asset` or `liability`.

Asset accountables:

- `Depository` — bank accounts (checking, savings)
- `Investment` — brokerage, 401k (has holdings)
- `Crypto` — crypto holdings
- `Property` — real estate
- `Vehicle` — vehicles
- `OtherAsset` — catch-all assets

Liability accountables:

- `CreditCard` — credit card debt
- `Loan` — mortgage, student loans
- `OtherLiability` — catch-all liabilities

## Account Balances

`Account::Balance` — a single balance value for an account on a specific date. Calculated daily by `Account::BalanceCalculator`. For investment accounts, balance = cash_balance + holdings_value.

## Account Holdings

`Holding` — applies to `Investment` accounts. Tracks `qty` of a `Security` at a `price` on a `date`. Calculated by `Holding::BaseCalculator`.

## Account Entries (Delegated Type)

`Entry` — any record that modifies an account's balance/holdings. Has `date`, `amount`, `currency`.

Amount signing convention:

- Negative = inflow (income to checking, payment to credit card, sell trade)
- Positive = outflow (expense from checking, charge on credit card, buy trade)

Entry types (`Entryable`):

- `Valuation` — absolute account value on a date
- `Transaction` — alters balance by amount (income/expense)
- `Trade` — buy/sell of a holding (investment accounts only), has `qty` and `price`

## Transfers

`Transfer` — movement of money between two accounts. Has inflow and outflow transactions. Auto-matched by: different accounts, within 4 days, same currency, opposite values.

- Regular transfers: excluded from income/expense calculations
- Debt payments (receiver is a Loan): counted as expense

## Plaid Integration

- `PlaidItem` — connection to Plaid (managed mode)
- `PlaidAccount` — 1:1 with internal `Account`
- ETL: PlaidItem fetches → PlaidAccount stores → Account/Entry normalized

## Sync System

`Syncable` concern. Creates `Sync` records for audit/status tracking.

- `Account` sync: auto-match transfers, calculate balances/holdings, enrich transactions
- `PlaidItem` sync: ETL from Plaid API → internal models → triggers Account syncs
- `Family` sync: orchestrates all PlaidItem + Account syncs. Runs daily via `AutoSync` concern.

Account sync triggers on every Entry update.

## Data Providers

Pluggable provider system via `Provider::Registry`. Configured at runtime through `Setting` model.

Two types:

1. **Concept data** — swappable providers for generic data (exchange rates, security prices). Interfaces in `app/models/provider/concepts/`. Accessed via `Provided` concerns on domain models.
2. **One-off data** — provider-specific methods called directly (e.g., `Provider::Registry.get_provider(:synth)&.usage`)

Provider implementations inherit from `Provider` and return `with_provider_response` (auto-catches errors).

```
app/models/provider.rb          # Base class
app/models/provider/registry.rb # Available providers by concept
app/models/provider/concepts/   # Concept interfaces
app/models/provider/synth.rb    # Concrete implementation
app/models/exchange_rate/provided.rb  # "Provided" concern example
```

## Background Jobs

Sidekiq handles: account syncing, import processing, AI chat responses, scheduled maintenance (sidekiq-cron). Jobs live in `app/jobs/`.

## API Architecture

- Internal: Controllers serve JSON via Turbo for SPA-like interactions
- External: `/api/v1/` namespace with Doorkeeper OAuth + API key auth (JWT)
- JSON rendering via Jbuilder templates
- Rate limiting via Rack Attack
