# Requirements Document

## Introduction

This document defines the requirements for the Enhanced Goal Planning feature in Maybe. The feature allows users to create and track financial goals (savings, debt payoff, retirement, custom), link them to accounts, track progress via account balances or manual contributions, see projected completion dates, and visualize progress on a dedicated page and dashboard widget.

## Glossary

- **Goal**: A model representing a financial goal belonging to a Family, with a target amount, optional target date, and progress tracking.
- **GoalAccount**: A join model linking a Goal to one or more Accounts for balance-based progress tracking.
- **GoalContribution**: A record of an explicit contribution to or withdrawal from a goal, optionally linked to an Entry.
- **Goal_Type**: Classification of a goal as `savings`, `debt_payoff`, `retirement`, or `custom`.
- **Status**: The goal's progress state: `on_track`, `behind`, `ahead`, `completed`, or `paused`.
- **ProgressCalculator**: The `Goal::ProgressCalculator` PORO responsible for computing current progress, monthly contribution rate, projected completion, and status.
- **Monthly_Contribution**: The average monthly change in goal progress over the last 3 months, used for projection.
- **Projected_Completion_Date**: A calculated future date when the goal is expected to be met, based on current progress rate.

## Requirements

### Requirement 1: Goal Data Model

**User Story:** As a developer, I want a Goal model that stores financial goal data, so that the system can track and manage user-created goals.

#### Acceptance Criteria

1. THE Goal SHALL belong to a Family and store name, goal_type, target_amount, and currency as required fields
2. WHEN a Goal is created, THE Goal SHALL validate that goal_type is one of: savings, debt_payoff, retirement, or custom
3. WHEN a Goal is created, THE Goal SHALL validate that target_amount is greater than 0
4. THE Goal SHALL store optional fields: target_date, started_on, icon, color, priority, and status with a default of on_track
5. WHEN a Goal is created, THE Goal SHALL validate that status is one of: on_track, behind, ahead, completed, or paused
6. WHEN a Goal is created with a priority, THE Goal SHALL validate that priority is one of: high, medium, or low

### Requirement 2: Progress Percentage and Remaining Amount

**User Story:** As a user, I want to see my progress toward each goal as a percentage and remaining amount, so that I can understand how close I am to achieving it.

#### Acceptance Criteria

1. THE Goal SHALL calculate progress_percent as (current_amount / target_amount × 100), capped at 100.0
2. WHEN target_amount is zero, THE Goal SHALL return 0 for progress_percent
3. THE Goal SHALL calculate remaining_amount as (target_amount - current_amount), with a minimum of 0

### Requirement 3: Projected Completion and Time Tracking

**User Story:** As a user, I want to see when I'm projected to reach my goal and how many days remain, so that I can plan accordingly.

#### Acceptance Criteria

1. WHEN monthly_contribution is greater than 0 and remaining_amount is greater than 0, THE Goal SHALL calculate projected_completion_date as the current date plus the ceiling of (remaining_amount / monthly_contribution) months
2. WHEN monthly_contribution is nil or less than or equal to 0, THE Goal SHALL return nil for projected_completion_date
3. WHEN target_date is present, THE Goal SHALL calculate days_remaining as the number of days between the current date and target_date, returning 0 for past dates and nil when target_date is absent

### Requirement 4: Status Determination

**User Story:** As a user, I want the system to automatically determine whether I'm on track, ahead, behind, or have completed my goal, so that I can take action if needed.

#### Acceptance Criteria

1. WHEN current_amount is greater than or equal to target_amount, THE ProgressCalculator SHALL set status to completed
2. WHEN the Goal is paused, THE ProgressCalculator SHALL preserve the paused status regardless of progress
3. WHEN projected_completion_date is more than 30 days before target_date, THE ProgressCalculator SHALL set status to ahead
4. WHEN projected_completion_date is after target_date, THE ProgressCalculator SHALL set status to behind
5. WHEN projected_completion_date is within 30 days of or equal to target_date, THE ProgressCalculator SHALL set status to on_track

### Requirement 5: Goal-Account Linking

**User Story:** As a user, I want to link one or more accounts to a goal, so that the system can track my progress based on account balances.

#### Acceptance Criteria

1. THE GoalAccount SHALL enforce uniqueness on the combination of goal_id and account_id
2. WHEN a Goal is linked to accounts, THE Goal SHALL be able to access those accounts through the goal_accounts association
3. WHEN an Account is destroyed, THE corresponding GoalAccount records SHALL be destroyed

### Requirement 6: Goal Contributions

**User Story:** As a user, I want to record contributions to and withdrawals from my goals, so that I can track explicit progress on custom goals.

#### Acceptance Criteria

1. THE GoalContribution SHALL validate that amount is not zero
2. THE GoalContribution SHALL treat positive amounts as contributions and negative amounts as withdrawals
3. THE GoalContribution SHALL require amount, currency, and date fields
4. THE GoalContribution SHALL optionally link to an Entry for transaction-based tracking

### Requirement 7: Progress Calculation by Goal Type

**User Story:** As a user, I want my goal progress to be calculated differently based on the goal type, so that savings, debt payoff, retirement, and custom goals all track correctly.

#### Acceptance Criteria

1. WHEN a Goal has goal_type custom, THE ProgressCalculator SHALL compute current_amount as the sum of all GoalContribution amounts
2. WHEN a Goal has goal_type savings, THE ProgressCalculator SHALL compute current_amount as the sum of linked asset account balances
3. WHEN a Goal has goal_type debt_payoff, THE ProgressCalculator SHALL compute current_amount as target_amount minus the absolute value of linked liability account balances
4. WHEN a Goal has goal_type retirement, THE ProgressCalculator SHALL compute current_amount as the sum of linked Investment account balances
5. THE ProgressCalculator SHALL compute monthly_contribution as the average monthly balance change over the last 3 months of linked account data

### Requirement 8: Goals Index Page

**User Story:** As a user, I want a dedicated page listing all my goals with progress bars, so that I can see all my financial goals at a glance.

#### Acceptance Criteria

1. WHEN a user visits the goals page, THE GoalsController SHALL display all active goals ordered by priority
2. WHEN displaying goals, THE GoalsController SHALL show the name, goal_type, progress_percent as a progress bar, current_amount, target_amount, and status for each goal
3. THE GoalsController SHALL include completed goals in a separate section or filter

### Requirement 9: Goal Detail Page

**User Story:** As a user, I want a detail page for each goal showing a historical progress chart and contribution history, so that I can understand my progress over time.

#### Acceptance Criteria

1. WHEN a user views a goal detail page, THE GoalsController SHALL display goal information including name, type, target, current amount, status, projected completion date, and days remaining
2. WHEN a user views a goal detail page, THE GoalsController SHALL render a D3.js progress chart showing historical progress over time
3. WHEN a goal has contributions, THE GoalsController SHALL display a chronological list of contributions and withdrawals

### Requirement 10: Family Scoping and Security

**User Story:** As a developer, I want all goal operations scoped to the current family, so that users cannot access other families' goals.

#### Acceptance Criteria

1. THE GoalsController SHALL scope all queries and mutations to Current.family
2. WHEN linking accounts to a goal, THE GoalsController SHALL verify that the accounts belong to Current.family
3. THE GoalsController SHALL use the existing authentication concern for access control

### Requirement 11: Goal CRUD Operations

**User Story:** As a user, I want to create, edit, and delete goals, so that I can manage my financial planning.

#### Acceptance Criteria

1. WHEN a user creates a goal, THE GoalsController SHALL save the goal with the family's currency and trigger an initial progress calculation
2. WHEN a user updates a goal, THE GoalsController SHALL persist changes and recalculate progress
3. WHEN a user deletes a goal, THE GoalsController SHALL destroy the goal and all associated goal_accounts and goal_contributions
4. WHEN a user creates or edits a goal, THE GoalsController SHALL allow linking and unlinking accounts

### Requirement 12: Dashboard Widget

**User Story:** As a user, I want to see a summary of my active goals on the dashboard, so that I can monitor progress at a glance.

#### Acceptance Criteria

1. WHEN the dashboard loads and the user has active goals, THE Dashboard_Widget SHALL display a summary of active goals with name, progress bar, and status
2. THE Dashboard_Widget SHALL be rendered via a Turbo frame for non-blocking page load

### Requirement 13: Error Handling

**User Story:** As a user, I want the system to handle edge cases gracefully, so that missing data does not cause errors.

#### Acceptance Criteria

1. WHEN a Goal has no linked accounts and goal_type is not custom, THE ProgressCalculator SHALL return current_amount of 0
2. WHEN a linked Account is destroyed, THE Goal SHALL continue to function with remaining linked accounts
3. WHEN progress calculation encounters an error, THE ProgressCalculator SHALL log the error and preserve the goal's previous state
