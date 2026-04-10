# Tasks

## Task 1: Family::CashFlowForecast Core Engine

- [ ] 1.1 Create `app/models/family/cash_flow_forecast.rb` with HORIZONS constant, DailyProjection/RecurringItem/ForecastInsight Data.define structs, initialize(family, horizon_days:) with clamping, and attr_readers
- [ ] 1.2 Implement `calculate_current_cash_balance` — sum visible Depository account balances converted to family currency via ExchangeRate (fallback rate 1.0)
- [ ] 1.3 Implement `build_recurring_flow_map` — query active RecurringSeries, call projected_dates_in_range for the forecast horizon, return Hash<Date, Array<RecurringItem>>
- [ ] 1.4 Implement `daily_variable_expense_rate` and `daily_variable_income_rate` — historical median from IncomeStatement minus recurring monthly cost, clamped to 0, divided by 30
- [ ] 1.5 Implement `build_daily_projections` — iterate horizon_days, layer recurring flows + variable rates, track running balance, return array of DailyProjection structs
- [ ] 1.6 Implement `generate_insights` — produce ForecastInsight structs for upcoming_bills (7-day window), spending_comparison (current vs prior month), surplus_deficit (always), low_balance_warning (if negative)
- [ ] 1.7 Implement `to_series` and `build_historical_balance_values` — combine last 30 days of actual depository balance data with projected values into a Series object
- [ ] 1.8 Add `cash_flow_forecast(horizon_days:)` convenience method to `app/models/family.rb`

## Task 2: Unit Tests for CashFlowForecast

- [ ] 2.1 Create test fixtures: family with depository accounts (single and multi-currency), active RecurringSeries (monthly bill, weekly income), and historical transactions for IncomeStatement medians
- [ ] 2.2 Create `test/models/family/cash_flow_forecast_test.rb` with tests for: projection count matches horizon, balance continuity, net_flow consistency, date consecutiveness, horizon clamping, default horizon
- [ ] 2.3 Add tests for `calculate_current_cash_balance`: depository-only sum, multi-currency conversion, zero when no depository accounts
- [ ] 2.4 Add tests for `build_recurring_flow_map`: recurring items placed on correct dates, empty map when no series
- [ ] 2.5 Add tests for variable rate calculation: non-negativity, zero when recurring exceeds historical, correct subtraction
- [ ] 2.6 Add tests for `generate_insights`: each insight type present/absent based on conditions, correct values and metadata
- [ ] 2.7 Add tests for `to_series`: returns valid Series object, contains historical + projected data, handles no-account edge case
- [ ] 2.8 Add edge case tests: no accounts, no recurring series, no history, all three missing simultaneously

## Task 3: CashFlowForecastsController and Routes

- [ ] 3.1 Create `app/controllers/cash_flow_forecasts_controller.rb` with `show` action — parse horizon param (default 30, clamp 30-90), build forecast, load income/expense totals via IncomeStatement with Periodable, set breadcrumbs
- [ ] 3.2 Add route `resource :cash_flow_forecast, only: [:show]` in `config/routes.rb`
- [ ] 3.3 Create `test/controllers/cash_flow_forecasts_controller_test.rb` with tests for: show renders, horizon param handling, period param handling, family scoping

## Task 4: Cash Flow Forecast Page Views

- [ ] 4.1 Create `app/views/cash_flow_forecasts/show.html.erb` with: forecast summary (current balance, projected end balance, net change), horizon selector (30/60/90 as links with query params), forecast chart container, insights list, income vs expense breakdown in Turbo frame
- [ ] 4.2 Create `app/views/cash_flow_forecasts/_insights.html.erb` partial rendering each ForecastInsight with icon, title, description, and formatted value using DS components and design tokens
- [ ] 4.3 Create `app/views/cash_flow_forecasts/_income_expense_breakdown.html.erb` partial with income and expense totals for the selected period, wrapped in a Turbo frame for period switching

## Task 5: Dashboard Forecast Widget

- [ ] 5.1 Create `app/views/pages/dashboard/_forecast_widget.html.erb` partial showing 30-day forecast summary: current cash balance, projected end balance, trend indicator, and top 2 insights (prioritize upcoming_bills and low_balance_warning)
- [ ] 5.2 Integrate the forecast widget into `app/views/pages/dashboard.html.erb` — add forecast widget call in the dashboard layout, build forecast in PagesController#dashboard, wrap in Turbo frame

## Task 6: Forecast Chart Integration

- [ ] 6.1 Wire the forecast Series JSON into the existing `time-series-chart` Stimulus controller via `data-time-series-chart-data-value` attribute in the show view, using `forecast.to_series.to_json`
- [ ] 6.2 Add visual distinction for the projected portion of the chart — extend the time-series-chart controller to support a `split-at` value that renders the projected region with a dashed line style (reuse existing `_installTrendlineSplit` pattern)

## Task 7: Property-Based Tests

- [ ] 7.1 (PBT) Property 1 & 12: Projection count and horizon clamping — for any horizon_days in [1, 365], projections.size == horizon_days; for values outside range, horizon is clamped
- [ ] 7.2 (PBT) Property 2: Balance continuity — for any forecast, projection[i].balance == projection[i-1].balance + projection[i].net_flow for all i
- [ ] 7.3 (PBT) Property 3: Net flow consistency — for any DailyProjection, net_flow == inflows - outflows
- [ ] 7.4 (PBT) Property 4: Date consecutiveness — for any forecast, projection[i].date == start_date + i for all i
- [ ] 7.5 (PBT) Property 6 & 7: Variable rate non-negativity and no double-counting — for any combination of recurring costs and historical medians, variable rates are >= 0
- [ ] 7.6 (PBT) Property 9: Insight generation completeness — surplus_deficit always present; upcoming_bills present iff bills in first 7 days; low_balance_warning present iff any balance < 0
