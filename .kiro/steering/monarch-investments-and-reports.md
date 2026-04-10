---
inclusion: manual
---

# Enhanced Investments, Reports & Connectivity

Implements Monarch parity features #11 (Equity/RSU tracking), #13 (Saved reports), #15 (Investment benchmarking), #17 (Connectivity dashboard).

## 1. Equity / RSU / Stock Option Tracking

### Concept

Track vested vs unvested equity compensation alongside regular investments.

### Domain Changes

Add to `Holding` model:

- `vesting_status`: enum (vested, unvested, partially_vested) — default nil for regular holdings
- `vesting_date`: date (when fully vested)
- `grant_date`: date (when granted)
- `grant_price`: decimal (strike price for options)

Add `holding_type` enum to `Holding`:

- `standard` (default — regular stock/ETF/fund)
- `rsu` (restricted stock unit)
- `stock_option` (employee stock option)
- `espp` (employee stock purchase plan)

### Equity Value Calculation

```ruby
# app/models/holding/equity_calculator.rb
class Holding::EquityCalculator
  def initialize(holding)

  def current_value
    case holding.holding_type
    when "rsu"
      holding.qty * holding.security_price  # vested shares × market price
    when "stock_option"
      gain = holding.security_price - holding.grant_price
      gain > 0 ? holding.qty * gain : 0  # intrinsic value only
    else
      holding.qty * holding.security_price
    end
  end

  def unvested_value
    # Only for partially_vested or unvested holdings
  end
end
```

### Net Worth Inclusion

- Vested equity: included in net worth (it's a real asset)
- Unvested equity: shown separately as "pending" — not included in net worth by default
- User preference to include/exclude unvested from net worth calculations

### UI

- Holdings page shows vesting status badge per holding
- Equity section groups RSUs, options, ESPP separately from regular holdings
- Vesting timeline visualization (simple bar showing vested vs unvested)
- Grant details: grant date, grant price, vesting schedule

## 2. Saved / Favorite Reports

### Concept

Users can configure a report (filters, date range, categories, accounts) and save it for quick access later.

### Domain Model

```
SavedReport (new model, belongs_to :user)
  - name: string
  - report_type: enum (cash_flow, spending, income, net_worth, custom)
  - filters: jsonb (serialized filter state)
  - pinned: boolean (default false — pinned reports show on reports index)
  - last_viewed_at: datetime
```

### Filter Schema (jsonb)

```json
{
  "date_range": { "start": "2025-01-01", "end": "2025-12-31" },
  "period": "monthly",
  "account_ids": [1, 2, 3],
  "category_ids": [5, 10],
  "tag_ids": [2],
  "merchant_ids": [],
  "view": "mine"
}
```

### UI

- "Save Report" button on any report page
- Name the report in a dialog
- Saved reports list on reports index page
- Pinned reports appear at top with quick-access cards
- Click a saved report → loads the report page with all filters pre-applied via query params
- Delete / rename saved reports

### Implementation

Reports page reconstructs state from query params (already the convention). Saving a report just persists those query params as JSON. Loading a saved report redirects to the report URL with those params.

## 3. Investment Benchmarking

### Concept

Compare portfolio performance against market benchmarks (S&P 500, NASDAQ, etc.).

### Implementation

Add benchmark data to the existing security/provider system:

```ruby
# app/models/holding/benchmark.rb
class Holding::Benchmark
  BENCHMARKS = {
    sp500: { ticker: "SPY", name: "S&P 500" },
    nasdaq: { ticker: "QQQ", name: "NASDAQ 100" },
    total_market: { ticker: "VTI", name: "Total US Market" },
  }

  def initialize(family, benchmark: :sp500, period: 1.year)

  def portfolio_return    # time-weighted return of family portfolio
  def benchmark_return    # return of benchmark over same period
  def alpha               # portfolio_return - benchmark_return
  def comparison_series   # array of { date:, portfolio:, benchmark: } for chart
end
```

### Data Source

Benchmark prices come from the existing security price provider (Synth/FMP). Fetch benchmark ticker prices alongside regular holdings during sync.

### UI

- Performance chart on investments page with toggle to overlay benchmark line
- Summary card: "Your portfolio returned X% vs S&P 500 at Y%"
- Time period selector: 1M, 3M, 6M, YTD, 1Y, All

## 4. Connectivity Dashboard

### Concept

Show the health status of all connected accounts — which are syncing properly, which have errors, when each last synced.

### Implementation

This data already exists on `Sync` and `PlaidItem` models. Build a dedicated view:

```ruby
# In Account model or concern
def connection_status
  last_sync = syncs.order(created_at: :desc).first
  return :never_synced if last_sync.nil?
  return :error if last_sync.failed?
  return :stale if last_sync.created_at < 2.days.ago
  :healthy
end
```

### UI

- Settings > Connections page (or standalone /connections route)
- List all connected accounts with status indicators:
  - Green dot: healthy (synced within 24h)
  - Yellow dot: stale (no sync in 24-48h)
  - Red dot: error (last sync failed)
  - Gray dot: never synced / manual account
- Last sync timestamp per account
- "Sync Now" button per account
- Error details expandable per account (from Sync record)
- Plaid Item reconnection flow for broken connections

## Implementation Notes

- Equity tracking extends existing Holding model (migration adds columns)
- Saved reports are a lightweight model — filters stored as JSON, loaded as query params
- Benchmarking reuses existing security price infrastructure
- Connectivity dashboard queries existing Sync records — no new data needed
- All calculations in models (Convention 2)
- Use Turbo frames for benchmark chart period switching
- Connectivity page uses functional tokens for status colors (text-success, text-warning, text-destructive)
