# Tasks

## Task 1: Database Migration and RecurringSeries Model

- [ ] 1.1 Create migration `create_recurring_series` with UUID primary key, family/merchant/category references, title, frequency, series_type, status, average_amount, currency, date fields, confidence, auto_detected, and composite indexes
- [ ] 1.2 Create migration to add `recurring_series_id` (UUID, foreign key) to the `transactions` table
- [ ] 1.3 Create `app/models/recurring_series.rb` with belongs_to :family, optional belongs_to :merchant and :category, has_many :transactions, enums for frequency/series_type/status, Monetizable concern, validations, and scopes (active, bills_and_subscriptions, incomes, due_within)
- [ ] 1.4 Add `has_many :recurring_series` to the Family model
- [ ] 1.5 Add `belongs_to :recurring_series, optional: true` to the Transaction model
- [ ] 1.6 Implement `overdue?`, `upcoming?`, `monthly_cost`, `annual_cost`, `projected_dates_in_range`, and `frequency_interval_days` methods on RecurringSeries
- [ ] 1.7 Create `test/fixtures/recurring_series.yml` with fixture data covering each frequency and series_type
- [ ] 1.8 Create `test/models/recurring_series_test.rb` with unit tests for validations, scopes, overdue?, upcoming?, monthly_cost, annual_cost, and projected_dates_in_range

## Task 2: Detection Algorithm — Family::RecurringSeriesDetector

- [ ] 2.1 Create `app/models/family/recurring_series_detector.rb` with constants (MINIMUM_OCCURRENCES, AMOUNT_TOLERANCE, CONFIDENCE_THRESHOLD, STALE_MULTIPLIER, FREQUENCY_RANGES)
- [ ] 2.2 Implement `group_transactions_by_merchant_and_amount` — query transactions with merchants from last 2 years, group by merchant, sub-group with `bucket_by_amount`
- [ ] 2.3 Implement `bucket_by_amount` — sort transactions by amount, place each into a bucket within 15% tolerance of bucket average
- [ ] 2.4 Implement `detect_frequency` — calculate median interval from sorted dates, match against FREQUENCY_RANGES
- [ ] 2.5 Implement `calculate_confidence` — weighted score: 70% interval regularity + 30% sample size
- [ ] 2.6 Implement `infer_series_type` — income for negative amounts, subscription for low variance outflows (<5%), bill otherwise
- [ ] 2.7 Implement `detect` main method — iterate groups, filter by minimum occurrences, detect frequency, score confidence, upsert RecurringSeries, link transactions, mark stale series as paused
- [ ] 2.8 Create `test/models/family/recurring_series_detector_test.rb` with unit tests for each sub-method and integration test for the full detection pipeline

## Task 3: Sync Pipeline Integration

- [ ] 3.1 Integrate `Family::RecurringSeriesDetector` into `Family::Syncer#perform_post_sync` with error rescue (log + Sentry, non-blocking)
- [ ] 3.2 Add test in `test/models/family/syncer_test.rb` verifying detector runs during post-sync and errors are handled gracefully

## Task 4: RecurringTransactionsController and Routes

- [ ] 4.1 Create `app/controllers/recurring_transactions_controller.rb` with index (filter tabs), calendar, show, new, create, update, destroy actions — all scoped to Current.family
- [ ] 4.2 Add routes for recurring_transactions (index, calendar, show, new, create, update, destroy) in `config/routes.rb`
- [ ] 4.3 Create `test/controllers/recurring_transactions_controller_test.rb` with tests for each action, filter behavior, family scoping, and manual CRUD

## Task 5: Views — Recurring Transactions Page

- [ ] 5.1 Create `app/views/recurring_transactions/index.html.erb` with filter tabs (all, bills, subscriptions, income) using Turbo frames, series list showing title, merchant, frequency, average amount, next expected date, and monthly/annual cost totals
- [ ] 5.2 Create `app/views/recurring_transactions/show.html.erb` with series detail and linked transaction history
- [ ] 5.3 Create `app/views/recurring_transactions/new.html.erb` and `_form.html.erb` partial for manual series creation/editing

## Task 6: Views — Bill Calendar

- [ ] 6.1 Create `app/views/recurring_transactions/calendar.html.erb` with monthly calendar grid showing projected and actual payment dates for active bills and subscriptions
- [ ] 6.2 Implement Turbo frame month navigation (prev/next) that renders the calendar partial without full page reload, defaulting to current month

## Task 7: Dashboard Widget — Upcoming Bills

- [ ] 7.1 Create a partial `app/views/recurring_transactions/_upcoming_widget.html.erb` showing active bills/subscriptions due within 7 days, with visual distinction for overdue items
- [ ] 7.2 Integrate the upcoming bills widget into the dashboard page via Turbo frame

## Task 8: Property-Based Tests

- [ ] 8.1 (PBT) Property 2: Monthly cost normalization — for any valid frequency and positive average_amount, monthly_cost equals amount × correct multiplier, annual_cost equals monthly_cost × 12, and monthly_cost is positive
- [ ] 8.2 (PBT) Property 5: Amount bucketing integrity — for any set of transactions from the same merchant, every transaction lands in exactly one bucket and all amounts within a bucket are within 15% of the bucket average
- [ ] 8.3 (PBT) Property 7: Frequency detection — for any array of positive intervals, detect_frequency returns the correct frequency when median is in range, nil otherwise
- [ ] 8.4 (PBT) Property 8: Confidence score bounds — for any valid inputs, confidence score is between 0.0 and 1.0
- [ ] 8.5 (PBT) Property 10: Series type inference — for any transaction group, infer_series_type returns income for negative amounts, subscription for low-variance outflows, bill for higher-variance outflows
- [ ] 8.6 (PBT) Property 11: Detection idempotency — for any transaction dataset, running detection twice produces the same RecurringSeries records with no duplicates
- [ ] 8.7 (PBT) Property 14: Projected dates within range — for any active series and date range, all projected dates fall within [start, end] and inactive series return empty
