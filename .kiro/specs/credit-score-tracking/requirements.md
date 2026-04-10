# Requirements Document

## Introduction

This document defines the requirements for the Credit Score Tracking feature in Maybe. The feature allows users to view, track, and monitor their credit score over time through both automated provider fetching and manual entry. It includes a gauge visualization, historical chart, score factors, rating classification, and a dashboard widget.

## Glossary

- **CreditScore**: A model representing a single credit score observation for a user on a specific date, with score value, type, source, and optional factors.
- **Score_Type**: The scoring model used: `fico`, `vantage`, or `other`.
- **Source**: How the score was obtained: `manual`, `plaid`, or `synth`.
- **Rating**: A human-readable classification of a score: poor (300–579), fair (580–669), good (670–739), very_good (740–799), excellent (800–850).
- **SCORE_RANGES**: The constant mapping of rating symbols to score ranges, covering 300–850 with no gaps or overlaps.
- **Factors**: An optional JSONB array of strings describing what influences the user's credit score (e.g., "Low credit utilization").
- **Provider**: A pluggable data source registered in `Provider::Registry` under the `:credit_scores` concept.
- **Gauge**: An SVG/D3 arc visualization showing the current score and its rating band.

## Requirements

### Requirement 1: CreditScore Data Model

**User Story:** As a developer, I want a CreditScore model that stores credit score observations for a user, so that the system can track scores over time from multiple sources.

#### Acceptance Criteria

1. THE CreditScore SHALL belong to a User and store score (integer), score_type, source, and recorded_on as required fields
2. WHEN a CreditScore is created, THE CreditScore SHALL validate that score is an integer between 300 and 850 inclusive
3. WHEN a CreditScore is created, THE CreditScore SHALL validate that score_type is one of: fico, vantage, or other
4. WHEN a CreditScore is created, THE CreditScore SHALL validate that source is one of: manual, plaid, or synth
5. WHEN a CreditScore is created, THE CreditScore SHALL validate that recorded_on is present
6. THE CreditScore SHALL enforce a unique constraint on the combination of user_id, recorded_on, and score_type so that only one score exists per user per date per type
7. THE CreditScore SHALL store an optional factors field as a JSONB array, and factors_list SHALL return the stored array or an empty array if factors is nil or empty

### Requirement 2: Rating Classification

**User Story:** As a user, I want my credit score classified into a rating (poor through excellent), so that I can quickly understand where my score falls.

#### Acceptance Criteria

1. THE CreditScore SHALL classify scores into ratings: poor (300–579), fair (580–669), good (670–739), very_good (740–799), excellent (800–850)
2. THE SCORE_RANGES SHALL cover the entire 300–850 range with no gaps and no overlaps
3. FOR any valid CreditScore, the rating method SHALL return exactly one of: :poor, :fair, :good, :very_good, or :excellent

### Requirement 3: Score Change Tracking

**User Story:** As a user, I want to see how my credit score has changed compared to my previous score, so that I can track my progress.

#### Acceptance Criteria

1. THE CreditScore SHALL calculate score_change_since(other_score) as the arithmetic difference between the current score and the other score
2. WHEN other_score is nil, score_change_since SHALL return nil
3. THE score change SHALL be a positive integer for improvement, negative for decline, and zero for no change

### Requirement 4: Score History and Querying

**User Story:** As a user, I want to view my credit score history over configurable time periods, so that I can see trends in my score.

#### Acceptance Criteria

1. THE CreditScore chronological scope SHALL return scores ordered by recorded_on in ascending order
2. THE CreditScore for_period scope SHALL return only scores with recorded_on within the specified start and end dates inclusive
3. THE CreditScoresController SHALL support period parameters: 3m, 6m, 1y, 2y, and all, each mapping to the correct start date relative to the current date
4. WHEN an unknown period parameter is provided, THE CreditScoresController SHALL default to 1 year

### Requirement 5: Manual Credit Score Entry

**User Story:** As a user, I want to manually enter my credit score, so that I can track my score even without an automated provider.

#### Acceptance Criteria

1. WHEN a user submits the credit score form, THE CreditScoresController SHALL create a CreditScore with the provided score, score_type, and recorded_on
2. THE CreditScoresController SHALL set source to "manual" server-side regardless of any user-submitted source parameter
3. WHEN a user deletes a credit score, THE CreditScoresController SHALL remove the record from the database
4. WHEN validation fails on manual entry, THE CreditScoresController SHALL re-render the form with error messages

### Requirement 6: Automated Provider Fetching

**User Story:** As a user, I want my credit score fetched automatically from a configured provider, so that my score stays up to date without manual effort.

#### Acceptance Criteria

1. THE CreditScore::Provided concern SHALL look up the credit score provider from Provider::Registry for the :credit_scores concept
2. WHEN no provider is configured, THE CreditScore::Provided SHALL return nil without raising an error
3. WHEN a provider returns a successful response, THE CreditScore::Provided SHALL upsert a CreditScore record keyed on user_id, recorded_on, and score_type
4. WHEN a provider returns an error response, THE CreditScore::Provided SHALL return nil without creating or modifying any records
5. THE CreditScoreFetchJob SHALL execute the provider fetch in a background job and handle exceptions by logging and reporting to Sentry without re-raising

### Requirement 7: User Scoping and Security

**User Story:** As a user, I want my credit score data to be private to me, so that other users (even in my family) cannot see my scores.

#### Acceptance Criteria

1. ALL CreditScoresController actions SHALL scope queries to Current.user so that a user cannot access, modify, or delete another user's credit scores
2. THE CreditScore model SHALL belong to User (not Family), ensuring credit scores are personal data

### Requirement 8: Credit Score Overview Page

**User Story:** As a user, I want a dedicated page showing my current credit score with a gauge visualization, rating, score change, and factors, so that I have a comprehensive view of my credit health.

#### Acceptance Criteria

1. WHEN a user visits the credit score page, THE CreditScoresController SHALL display the latest score, its rating, and the score change from the previous score
2. WHEN a user has score factors, THE credit score page SHALL display the factors list
3. WHEN a user has no credit scores, THE credit score page SHALL display an empty state encouraging manual entry or provider setup
4. THE credit score page SHALL include an SVG gauge visualization showing the score position within the 300–850 range with the rating band highlighted

### Requirement 9: Historical Chart

**User Story:** As a user, I want a line chart showing my credit score history, so that I can visualize trends over time.

#### Acceptance Criteria

1. THE credit score history view SHALL display a D3.js line chart of scores over the selected time period
2. THE chart SHALL use the chronological scope to plot scores by recorded_on date
3. THE chart SHALL support period filtering via query parameters (3m, 6m, 1y, 2y, all) using Turbo frames for navigation without full page reload

### Requirement 10: Dashboard Widget

**User Story:** As a user, I want a credit score widget on my dashboard, so that I can see my current score at a glance.

#### Acceptance Criteria

1. THE dashboard widget SHALL display the user's latest credit score and rating
2. WHEN the user has no credit scores, THE dashboard widget SHALL display a prompt to add a score
3. THE dashboard widget SHALL be rendered via a Turbo frame for independent loading
