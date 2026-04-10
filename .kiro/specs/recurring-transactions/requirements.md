# Requirements Document

## Introduction

This document defines the requirements for the Recurring Transactions / Subscription Detection feature in Maybe. The feature automatically detects recurring financial patterns (bills, subscriptions, income) from a family's transaction history, surfaces them in a dedicated UI with calendar and dashboard views, and allows manual management of recurring series.

## Glossary

- **RecurringSeries**: A model representing a detected or manually created recurring financial pattern belonging to a Family.
- **Detector**: The `Family::RecurringSeriesDetector` PORO responsible for analyzing transaction history and identifying recurring patterns.
- **Series_Type**: Classification of a recurring series as `bill`, `subscription`, or `income`.
- **Frequency**: The recurrence interval of a series: `weekly`, `biweekly`, `monthly`, `quarterly`, `semi_annual`, or `annual`.
- **Confidence_Score**: A float between 0.0 and 1.0 representing the detection algorithm's certainty that a transaction group is truly recurring.
- **Amount_Bucket**: A grouping of transactions from the same merchant whose amounts are within 15% of the bucket average.
- **Projected_Date**: A calculated future date when a recurring payment is expected, based on frequency and last observed date.
- **Stale_Series**: A recurring series that has not had a matching transaction within 2.5× its expected interval.
- **Family_Sync**: The existing background sync pipeline that orchestrates account syncing, enrichment, and now recurring detection.

## Requirements

### Requirement 1: Recurring Series Data Model

**User Story:** As a developer, I want a RecurringSeries model that stores recurring financial patterns, so that the system can track and manage detected and manually created recurring transactions.

#### Acceptance Criteria

1. THE RecurringSeries SHALL belong to a Family and store title, frequency, series_type, status, average_amount, and currency as required fields
2. WHEN a RecurringSeries is created, THE RecurringSeries SHALL validate that frequency is one of: weekly, biweekly, monthly, quarterly, semi_annual, or annual
3. WHEN a RecurringSeries is created, THE RecurringSeries SHALL validate that series_type is one of: bill, subscription, or income
4. WHEN a RecurringSeries is created, THE RecurringSeries SHALL validate that status is one of: active, cancelled, or paused
5. WHEN a RecurringSeries is created, THE RecurringSeries SHALL validate that average_amount is positive
6. THE RecurringSeries SHALL optionally reference a Merchant and a Category belonging to the same Family
7. THE RecurringSeries SHALL track first_observed_date, last_observed_date, next_expected_date, confidence score, and auto_detected flag

### Requirement 2: Monthly and Annual Cost Normalization

**User Story:** As a user, I want to see the monthly and annual cost of each recurring series regardless of its frequency, so that I can understand my total recurring financial commitments.

#### Acceptance Criteria

1. THE RecurringSeries SHALL calculate monthly_cost by normalizing average_amount based on frequency (weekly × 52/12, biweekly × 26/12, monthly × 1, quarterly × 1/3, semi_annual × 1/6, annual × 1/12)
2. THE RecurringSeries SHALL calculate annual_cost as monthly_cost multiplied by 12
3. THE RecurringSeries SHALL return a positive value for monthly_cost for any valid series

### Requirement 3: Recurring Series Status Queries

**User Story:** As a user, I want to know which recurring payments are overdue or upcoming, so that I can stay on top of my financial obligations.

#### Acceptance Criteria

1. WHEN a RecurringSeries is active and next_expected_date is before the current date, THE RecurringSeries SHALL report itself as overdue
2. WHEN a RecurringSeries is active and next_expected_date is within a specified number of days from the current date, THE RecurringSeries SHALL report itself as upcoming
3. WHEN a RecurringSeries is cancelled or paused, THE RecurringSeries SHALL not report itself as overdue or upcoming

### Requirement 4: Transaction Grouping by Merchant and Amount

**User Story:** As a developer, I want the detection algorithm to group transactions by merchant and similar amounts, so that recurring patterns can be identified from transaction history.

#### Acceptance Criteria

1. WHEN the Detector groups transactions, THE Detector SHALL only consider transactions with an assigned merchant from the last 2 years
2. WHEN the Detector groups transactions by merchant, THE Detector SHALL sub-group them into Amount_Buckets where all transactions in a bucket have amounts within 15% of the bucket average
3. THE Detector SHALL place every input transaction into exactly one Amount_Bucket
4. WHEN an Amount_Bucket contains fewer than 3 transactions, THE Detector SHALL skip that group for recurrence analysis

### Requirement 5: Frequency Detection from Transaction Intervals

**User Story:** As a developer, I want the system to detect the frequency of recurring transactions from the intervals between them, so that the correct recurrence pattern is identified.

#### Acceptance Criteria

1. WHEN the Detector analyzes a group of transactions, THE Detector SHALL calculate the median interval in days between consecutive transactions
2. WHEN the median interval falls within a defined frequency range (weekly: 5-9, biweekly: 12-16, monthly: 25-35, quarterly: 80-100, semi_annual: 170-195, annual: 350-380), THE Detector SHALL assign the corresponding frequency
3. WHEN the median interval does not fall within any defined frequency range, THE Detector SHALL discard the group from detection

### Requirement 6: Confidence Scoring

**User Story:** As a developer, I want the detection algorithm to score its confidence in each detected pattern, so that only sufficiently reliable patterns are surfaced to users.

#### Acceptance Criteria

1. THE Detector SHALL calculate a Confidence_Score between 0.0 and 1.0 for each candidate group
2. THE Detector SHALL weight interval regularity at 70% and sample size at 30% in the Confidence_Score calculation
3. WHEN a candidate group has a Confidence_Score below 0.6, THE Detector SHALL discard that group from detection
4. WHEN a RecurringSeries is auto-detected, THE RecurringSeries SHALL have a Confidence_Score of at least 0.6

### Requirement 7: Series Type Inference

**User Story:** As a user, I want the system to automatically classify recurring transactions as bills, subscriptions, or income, so that I can view them in meaningful categories.

#### Acceptance Criteria

1. WHEN the Detector analyzes a group of inflow transactions (negative amounts), THE Detector SHALL classify the series as income
2. WHEN the Detector analyzes a group of outflow transactions with amount variance below 5%, THE Detector SHALL classify the series as subscription
3. WHEN the Detector analyzes a group of outflow transactions with amount variance at or above 5%, THE Detector SHALL classify the series as bill

### Requirement 8: Detection Pipeline Integration with Family Sync

**User Story:** As a user, I want recurring transaction detection to run automatically when my accounts sync, so that new patterns are discovered without manual intervention.

#### Acceptance Criteria

1. WHEN a Family_Sync completes, THE Detector SHALL run as part of the sync pipeline
2. WHEN the Detector creates or updates a RecurringSeries, THE Detector SHALL use upsert semantics keyed on family, merchant, and series_type
3. WHEN the Detector identifies transactions matching a RecurringSeries, THE Detector SHALL link those transactions to the series via recurring_series_id
4. WHEN a RecurringSeries has not had a matching transaction within 2.5× its expected interval, THE Detector SHALL mark the series status as paused
5. IF the Detector encounters an error during detection, THEN THE Detector SHALL log the error, report to Sentry, and allow the sync to continue without affecting other sync operations

### Requirement 9: Detection Idempotency

**User Story:** As a developer, I want detection to produce the same results when run multiple times on the same data, so that repeated syncs do not create duplicate series.

#### Acceptance Criteria

1. WHEN the Detector runs twice on the same transaction data, THE Detector SHALL produce the same set of RecurringSeries records
2. THE Detector SHALL not create duplicate RecurringSeries for the same family, merchant, and series_type combination

### Requirement 10: Projected Date Calculation

**User Story:** As a user, I want to see projected future payment dates for my recurring series, so that I can plan my finances ahead.

#### Acceptance Criteria

1. WHEN a date range is provided, THE RecurringSeries SHALL generate projected dates spaced by the series frequency interval within that range
2. THE RecurringSeries SHALL return only dates that fall within the requested start and end dates
3. WHEN a RecurringSeries is not active or has no last_observed_date, THE RecurringSeries SHALL return an empty list of projected dates

### Requirement 11: Recurring Transactions Page

**User Story:** As a user, I want a dedicated page listing all my recurring transactions grouped by type, so that I can review my recurring financial commitments in one place.

#### Acceptance Criteria

1. WHEN a user visits the recurring transactions page, THE RecurringTransactionsController SHALL display all active recurring series ordered by next_expected_date
2. WHEN a user selects a filter tab (all, bills, subscriptions, income), THE RecurringTransactionsController SHALL filter the displayed series by the selected series_type
3. WHEN displaying recurring series, THE RecurringTransactionsController SHALL show the title, merchant, frequency, average amount, and next expected date for each series
4. WHEN a user views a single recurring series detail page, THE RecurringTransactionsController SHALL display the series information along with its linked transaction history

### Requirement 12: Bill Calendar View

**User Story:** As a user, I want a calendar view showing when my bills and subscriptions are due each month, so that I can visually plan for upcoming payments.

#### Acceptance Criteria

1. WHEN a user visits the bill calendar, THE RecurringTransactionsController SHALL display a monthly calendar grid with projected and actual payment dates for active bills and subscriptions
2. WHEN a user navigates to the next or previous month via Turbo frame, THE RecurringTransactionsController SHALL render the updated calendar for the requested month without a full page reload
3. WHEN no month parameter is provided, THE RecurringTransactionsController SHALL default to the current month

### Requirement 13: Dashboard Widget for Upcoming Bills

**User Story:** As a user, I want to see upcoming bills on my dashboard, so that I am aware of imminent payments at a glance.

#### Acceptance Criteria

1. WHEN the dashboard loads, THE Dashboard_Widget SHALL display active bills and subscriptions due within the next 7 days
2. WHEN a recurring series is overdue, THE Dashboard_Widget SHALL visually distinguish overdue items from upcoming items

### Requirement 14: Manual Recurring Series Management

**User Story:** As a user, I want to manually create, edit, and delete recurring series, so that I can track recurring transactions that the system did not auto-detect or correct ones it detected incorrectly.

#### Acceptance Criteria

1. WHEN a user creates a recurring series manually, THE RecurringTransactionsController SHALL save the series with auto_detected set to false
2. WHEN a user updates a recurring series, THE RecurringTransactionsController SHALL persist the changes to status, frequency, series_type, or other editable fields
3. WHEN a user deletes a recurring series, THE RecurringTransactionsController SHALL remove the series and unlink any associated transactions
4. THE RecurringTransactionsController SHALL scope all operations to Current.family to prevent cross-family data access

### Requirement 15: Error Handling for Edge Cases

**User Story:** As a user, I want the system to handle edge cases gracefully during detection, so that incomplete data does not cause errors or incorrect results.

#### Acceptance Criteria

1. WHEN a Family has no transactions with assigned merchants, THE Detector SHALL complete detection with zero results and no error
2. WHEN a Family has fewer than 3 transactions for any merchant, THE Detector SHALL produce no RecurringSeries for that merchant
3. IF a previously active RecurringSeries becomes stale, THEN THE Detector SHALL update its status to paused rather than deleting it
