# Tasks

## Task 1: Database Migration and CreditScore Model

- [ ] 1.1 Create migration `create_credit_scores` with UUID primary key, user reference, score (integer, not null), score_type (string, not null, default "fico"), source (string, not null, default "manual"), recorded_on (date, not null), factors (jsonb, default []), timestamps, unique index on [user_id, recorded_on, score_type], and index on [user_id, recorded_on]
- [ ] 1.2 Create `app/models/credit_score.rb` with belongs_to :user, enums for score_type (fico/vantage/other) and source (manual/plaid/synth), SCORE_RANGES constant, validations (score 300–850 integer, required fields, uniqueness on user+date+type), scopes (chronological, reverse_chronological, for_period)
- [ ] 1.3 Implement `rating`, `score_change_since(other_score)`, and `factors_list` methods on CreditScore
- [ ] 1.4 Add `has_many :credit_scores, dependent: :destroy` to the User model
- [ ] 1.5 Create `test/fixtures/credit_scores.yml` with fixture data covering each score_type, source, and rating range
- [ ] 1.6 Create `test/models/credit_score_test.rb` with unit tests for validations, enums, scopes, rating, score_change_since, and factors_list

## Task 2: Provider Integration — CreditScore::Provided Concern

- [ ] 2.1 Add `:credit_scores` to `Provider::Registry::CONCEPTS` and configure available providers in the `available_providers` method
- [ ] 2.2 Create `app/models/credit_score/provided.rb` with class methods: `provider` (looks up from registry) and `fetch_for_user(user)` (fetches from provider, upserts CreditScore record)
- [ ] 2.3 Include the `Provided` concern in the CreditScore model
- [ ] 2.4 Create `app/jobs/credit_score_fetch_job.rb` that calls `CreditScore.fetch_for_user(user)` with error rescue (log + Sentry)
- [ ] 2.5 Create `test/models/credit_score/provided_test.rb` with tests for fetch_for_user: successful fetch, provider error, no provider configured, and upsert idempotency

## Task 3: CreditScoresController and Routes

- [ ] 3.1 Create `app/controllers/credit_scores_controller.rb` with show (latest score + previous + all scores), index (period-filtered history), new, create (manual entry with forced source: "manual"), and destroy actions — all scoped to Current.user
- [ ] 3.2 Add routes for credit_scores (show, index, new, create, destroy) in `config/routes.rb`
- [ ] 3.3 Create `test/controllers/credit_scores_controller_test.rb` with tests for each action, period filtering, user scoping, manual source enforcement, and validation error handling

## Task 4: Views — Credit Score Overview Page

- [ ] 4.1 Create `app/views/credit_scores/show.html.erb` with SVG gauge visualization (score position in 300–850 range with rating band), current score display, rating label, score change indicator, and factors list
- [ ] 4.2 Create `app/views/credit_scores/_gauge.html.erb` partial with SVG arc rendering the score position and rating color
- [ ] 4.3 Create `app/views/credit_scores/new.html.erb` and `_form.html.erb` partial for manual score entry (score, score_type, recorded_on fields)
- [ ] 4.4 Create empty state view for when user has no credit scores, with prompts for manual entry

## Task 5: Views — Historical Chart

- [ ] 5.1 Create `app/views/credit_scores/index.html.erb` with period filter tabs (3m, 6m, 1y, 2y, all) using Turbo frames and D3.js line chart of score history
- [ ] 5.2 Create or extend a Stimulus controller for the D3.js credit score line chart, accepting score data via data attributes

## Task 6: Dashboard Widget

- [ ] 6.1 Create `app/views/credit_scores/_dashboard_widget.html.erb` partial showing latest score, rating, and score change — or empty state prompt if no scores exist
- [ ] 6.2 Integrate the credit score widget into the dashboard page via Turbo frame

## Task 7: Property-Based Tests

- [ ] 7.1 (PBT) Property 2: Rating classification completeness — for any integer score 300–850, rating returns exactly one valid rating symbol and SCORE_RANGES covers the full range with no gaps or overlaps
- [ ] 7.2 (PBT) Property 3: Rating classification correctness — for any score, rating returns the correct bucket: poor for 300–579, fair for 580–669, good for 670–739, very_good for 740–799, excellent for 800–850
- [ ] 7.3 (PBT) Property 5: Score change calculation — for any two CreditScores, score_change_since equals the arithmetic difference; for nil, returns nil
- [ ] 7.4 (PBT) Property 7: Period filtering — for any date range, for_period returns only scores with recorded_on within [start, end]
- [ ] 7.5 (PBT) Property 10: Provider fetch idempotency — fetching twice for the same user, date, and score_type produces exactly one CreditScore record
- [ ] 7.6 (PBT) Property 11: Factors storage integrity — for any CreditScore, factors_list returns the stored array or empty array if factors is nil/empty
