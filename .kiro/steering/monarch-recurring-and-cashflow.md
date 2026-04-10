---
inclusion: manual
---

# Recurring Transactions, Subscriptions & Cash Flow

Implements Monarch parity features #1 (Recurring/subscription detection), #2 (Cash flow forecasting), and #10 (Bill calendar with alerts).

## 1. Recurring Transaction Detection

### Domain Model

```
RecurringTransaction (new model, belongs_to :family)
  - merchant_name: string
  - category_id: references (optional)
  - account_id: references
  - frequency: enum (weekly, biweekly, monthly, quarterly, semi_annual, annual)
  - average_amount: decimal
  - currency: string
  - last_occurred_on: date
  - next_expected_on: date
  - status: enum (active, paused, cancelled)
  - recurring_type: enum (bill, subscription, income)
  - auto_detected: boolean (true if system-detected, false if user-created)
  - notes: text
```

### Detection Logic

Place in `app/models/recurring_transaction/detector.rb` as a PORO:

- Scan family transactions for the last 6 months
- Group by normalized merchant name + account
- Identify patterns: same merchant, similar amounts (±15%), regular intervals
- Frequency detection: calculate median interval between occurrences
  - 6-8 days → weekly
  - 13-16 days → biweekly
  - 26-35 days → monthly
  - 85-100 days → quarterly
  - 170-200 days → semi_annual
  - 340-400 days → annual
- Require minimum 2 occurrences for monthly+, 3 for weekly/biweekly
- Calculate `next_expected_on` from frequency + last occurrence
- Run detection as a Sidekiq job after each family sync

### Subscription Tracking

Subscriptions are `RecurringTransaction` records where `recurring_type: :subscription`. The UI should:

- Show a dedicated "Recurring" page (like Monarch's)
- Group by: bills, subscriptions, income
- Show monthly/annual cost totals
- Allow users to mark as cancelled (soft status change, not deletion)
- Allow manual creation for cash/untracked subscriptions

### Bill Calendar

- Calendar view showing upcoming bills/subscriptions by expected date
- Use `next_expected_on` to populate future dates
- Turbo frame for month navigation
- Color-code by: paid (green), upcoming (blue), overdue (red)
- Link each calendar entry to the recurring transaction detail

## 2. Cash Flow Forecasting

### Domain Model

Place logic in `app/models/family/cash_flow_forecast.rb`:

```ruby
class Family::CashFlowForecast
  def initialize(family, period: 30.days)
  def projected_income    # sum of recurring income for period
  def projected_expenses  # sum of recurring bills/subscriptions for period
  def projected_net       # income - expenses
  def daily_forecast      # array of { date:, projected_balance: }
  def surplus_or_deficit  # projected_net as Money object
end
```

### Calculation

- Start from current total cash balance (sum of depository account balances)
- Layer in known recurring transactions by their expected dates
- Project forward 30/60/90 days
- Show as a line chart (D3.js time series, reuse existing chart patterns)

### Cash Flow Insights

- "You have X bills totaling $Y due in the next 7 days"
- "Your projected balance on [date] is $Z"
- "You spent $X more on subscriptions this month vs last month"
- Surface these in the dashboard and via the AI assistant

## Implementation Notes

- Follow Convention 2: all logic in models, not services
- Use `Syncable` pattern — detection runs as part of family sync
- Use Turbo frames for calendar month navigation
- Server-side date formatting via `format_date` helper
- Use functional design tokens for calendar styling (bg-container, text-primary, etc.)
