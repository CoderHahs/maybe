# Requirements Document

## Introduction

This document defines the requirements for the Cash Flow Forecasting and Insights feature in Maybe. The feature projects future account balances by combining current cash position with known recurring transactions and historical spending patterns, surfaces actionable insights, and provides enhanced cash flow visualization with income vs expense breakdowns.

## Glossary

- **CashFlowForecast**: The `Family::CashFlowForecast` PORO that computes day-by-day projected cash balances for a given horizon.
- **Cash Position**: The sum of all visible Depository account balances, converted to the family's base currency.
- **DailyProjection**: An in-memory struct representing the projected balance, inflows, outflows, and recurring items for a single day.
- **ForecastInsight**: A structured data object representing an actionable insight derived from the forecast (e.g., upcoming bills, spending comparison).
- **Horizon**: The number of days into the future the forecast projects (30, 60, or 90 days).
- **Variable Rate**: The daily expense/income rate derived from historical averages minus known recurring amounts, used to fill gaps in the projection.
- **RecurringSeries**: A model from Feature #1 representing a detected or manually created recurring financial pattern.

## Requirements

### Requirement 1: Forecast Configuration and Projection Structure

**User Story:** As a user, I want to generate a cash flow forecast for 30, 60, or 90 days, so that I can plan my finances over different time horizons.

#### Acceptance Criteria

1. WHEN a user requests a forecast, THE CashFlowForecast SHALL accept a horizon_days parameter and clamp it to the range [1, 365]
2. THE CashFlowForecast SHALL produce exactly horizon_days DailyProjection structs, one per day starting from the current date
3. WHEN no horizon is specified, THE CashFlowForecast SHALL default to 30 days
4. EACH DailyProjection SHALL contain date, balance, inflows, outflows, net_flow, and recurring_items fields

### Requirement 2: Cash Position Calculation

**User Story:** As a user, I want the forecast to start from my actual cash balance across all bank accounts, so that projections are grounded in reality.

#### Acceptance Criteria

1. THE CashFlowForecast SHALL calculate current_cash_balance as the sum of balances from all visible Depository accounts only, excluding Investment, Property, Vehicle, Crypto, CreditCard, Loan, and other account types
2. WHEN a Depository account uses a different currency than the family's base currency, THE CashFlowForecast SHALL convert the balance using the current ExchangeRate, falling back to a rate of 1.0 if no rate is available
3. WHEN the family has no visible Depository accounts, THE CashFlowForecast SHALL return a current_cash_balance of 0

### Requirement 3: Recurring Flow Integration

**User Story:** As a user, I want the forecast to include my known recurring bills, subscriptions, and income on their expected dates, so that projections reflect my committed financial obligations.

#### Acceptance Criteria

1. THE CashFlowForecast SHALL query all active RecurringSeries for the family and place their projected amounts on the corresponding projected dates within the forecast horizon
2. WHEN calculating variable expense/income rates, THE CashFlowForecast SHALL subtract the total monthly recurring cost from the historical median to avoid double-counting
3. THE variable expense rate and variable income rate SHALL always be non-negative (≥ 0), clamped to 0 when recurring costs exceed historical medians
4. WHEN the family has no RecurringSeries records, THE CashFlowForecast SHALL proceed using only historical averages for projections

### Requirement 4: Forecast Insight Generation

**User Story:** As a user, I want actionable insights about my cash flow, so that I can make informed financial decisions.

#### Acceptance Criteria

1. WHEN there are recurring bill or subscription items due within the next 7 days, THE CashFlowForecast SHALL generate an upcoming_bills insight with the count and total amount
2. THE CashFlowForecast SHALL generate a spending_comparison insight comparing current month expenses to prior month expenses, showing the percentage change
3. THE CashFlowForecast SHALL always generate a surplus_deficit insight showing the projected net change over the forecast horizon
4. WHEN any projected daily balance is negative, THE CashFlowForecast SHALL generate a low_balance_warning insight identifying the earliest date of negative balance
5. EACH ForecastInsight SHALL contain kind, title, description, value (as Money in family currency), and metadata fields

### Requirement 5: Forecast Chart Series

**User Story:** As a user, I want to see a time series chart showing my projected balance trajectory, so that I can visually understand my future cash flow.

#### Acceptance Criteria

1. THE CashFlowForecast SHALL produce a Series object compatible with the existing time-series-chart Stimulus controller, containing both historical balance data and projected future data
2. THE Series values SHALL use the family's base currency and include date, formatted date, value (as Money), and trend information
3. WHEN the family has no depository accounts, THE CashFlowForecast SHALL produce a Series with only projected values starting from a balance of 0

### Requirement 6: Balance Projection Integrity

**User Story:** As a developer, I want the projection calculations to be mathematically consistent, so that the forecast is trustworthy.

#### Acceptance Criteria

1. FOR each DailyProjection at index i > 0, THE balance SHALL equal projection[i-1].balance + projection[i].net_flow
2. FOR the first DailyProjection (index 0), THE balance SHALL equal current_cash_balance + projection[0].net_flow
3. FOR each DailyProjection, THE net_flow SHALL equal inflows minus outflows
4. THE dates in projections SHALL be consecutive, with projection[i].date equal to start_date + i days

### Requirement 7: Cash Flow Forecast Page

**User Story:** As a user, I want a dedicated cash flow page showing my forecast chart, insights, and income vs expense breakdown, so that I have a comprehensive view of my cash flow.

#### Acceptance Criteria

1. THE CashFlowForecastsController SHALL scope all data access to Current.family to prevent cross-family data leakage
2. WHEN a user visits the cash flow forecast page, THE controller SHALL display the forecast chart, insights, current cash balance, projected end balance, and net change
3. THE page SHALL include a horizon selector (30/60/90 days) that updates the forecast via query parameter without full page reload
4. THE page SHALL display an income vs expense breakdown for a selectable period using the existing IncomeStatement, rendered in a Turbo frame
5. WHEN the page loads, THE controller SHALL default to a 30-day horizon and last_30_days period

### Requirement 8: Dashboard Integration

**User Story:** As a user, I want to see a cash flow forecast summary on my dashboard, so that I'm aware of my projected financial position at a glance.

#### Acceptance Criteria

1. THE dashboard SHALL include a forecast widget partial showing the projected balance trend and top insights for a 30-day horizon
2. THE forecast widget SHALL display the current cash balance, projected end balance, and any upcoming_bills or low_balance_warning insights
3. THE forecast widget SHALL render within a Turbo frame for independent loading

### Requirement 9: Graceful Degradation

**User Story:** As a user, I want the forecast to work even with limited data, so that I can start getting value immediately.

#### Acceptance Criteria

1. WHEN the family has no RecurringSeries records, THE CashFlowForecast SHALL produce projections using only historical spending averages
2. WHEN the family has no transaction history, THE CashFlowForecast SHALL produce a flat projection from the current cash balance (no variable flows)
3. WHEN the family has no depository accounts and no history, THE CashFlowForecast SHALL produce projections with all values at 0
4. THE CashFlowForecast SHALL not raise exceptions for any combination of missing data
