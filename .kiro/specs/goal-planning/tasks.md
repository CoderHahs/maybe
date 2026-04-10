# Tasks

## Task 1: Database Migrations

- [ ] 1.1 Create migration `create_goals` with UUID primary key, family reference, name, goal_type, status (default "on_track"), priority, icon, color, target_amount (decimal 19,4), current_amount (decimal 19,4 default 0), monthly_contribution (decimal 19,4 default 0), currency, target_date, started_on, timestamps, and composite indexes on (family_id, status) and (family_id, goal_type)
- [ ] 1.2 Create migration `create_goal_accounts` with UUID primary key, goal and account references (both NOT NULL, foreign keys, UUID), timestamps, and a unique index on (goal_id, account_id)
- [ ] 1.3 Create migration `create_goal_contributions` with UUID primary key, goal reference (NOT NULL, foreign key, UUID), entry reference (optional, foreign key, UUID), amount (decimal 19,4 NOT NULL), currency (NOT NULL), date (NOT NULL), description (string), timestamps, and index on (goal_id, date)

## Task 2: Goal Model

- [ ] 2.1 Create `app/models/goal.rb` with belongs_to :family, has_many :goal_accounts + :accounts through, has_many :goal_contributions, Monetizable concern, enums for goal_type/status/priority, validations (name, goal_type, target_amount > 0, currency presence), and scopes (active, completed, by_priority)
- [ ] 2.2 Implement `progress_percent`, `remaining_amount`, `projected_completion_date`, `on_track?`, and `days_remaining` methods on Goal
- [ ] 2.3 Add `has_many :goals, dependent: :destroy` to the Family model
- [ ] 2.4 Create `test/fixtures/goals.yml` with fixture data covering each goal_type and status
- [ ] 2.5 Create `test/models/goal_test.rb` with unit tests for validations, enums, scopes, progress_percent, remaining_amount, projected_completion_date, on_track?, and days_remaining

## Task 3: GoalAccount Model

- [ ] 3.1 Create `app/models/goal_account.rb` with belongs_to :goal, belongs_to :account, uniqueness validation on account_id scoped to goal_id
- [ ] 3.2 Create `test/fixtures/goal_accounts.yml` with fixture data linking goals to accounts
- [ ] 3.3 Create `test/models/goal_account_test.rb` with unit tests for uniqueness constraint and associations

## Task 4: GoalContribution Model

- [ ] 4.1 Create `app/models/goal_contribution.rb` with belongs_to :goal, optional belongs_to :entry, Monetizable concern, validations (amount not zero, currency and date presence), and scopes (contributions, withdrawals, chronological)
- [ ] 4.2 Create `test/fixtures/goal_contributions.yml` with fixture data for contributions and withdrawals
- [ ] 4.3 Create `test/models/goal_contribution_test.rb` with unit tests for validations, scopes, and associations

## Task 5: Goal::ProgressCalculator

- [ ] 5.1 Create `app/models/goal/progress_calculator.rb` with `calculate` method that computes current_amount based on goal_type, monthly_contribution rate from 3-month lookback, and status determination
- [ ] 5.2 Implement `compute_current_amount` — savings: sum asset balances, debt_payoff: target minus abs liability balances, retirement: sum investment balances, custom: sum contributions
- [ ] 5.3 Implement `compute_monthly_contribution_rate` — average monthly balance change over last 3 months from linked account balances
- [ ] 5.4 Implement `determine_status` — completed when current >= target, preserve paused, ahead when projected > 30 days early, behind when projected late, on_track otherwise
- [ ] 5.5 Create `test/models/goal/progress_calculator_test.rb` with unit tests for each goal_type calculation, monthly rate computation, status determination, and edge cases (no accounts, zero rate, already completed)

## Task 6: GoalsController and Routes

- [ ] 6.1 Create `app/controllers/goals_controller.rb` with index, show, new, create, edit, update, destroy actions — all scoped to Current.family, with account linking logic in create/update
- [ ] 6.2 Add resourceful routes for goals in `config/routes.rb`
- [ ] 6.3 Create `test/controllers/goals_controller_test.rb` with tests for each action, family scoping, account linking, and progress calculation trigger on create/update

## Task 7: Views — Goals Index Page

- [ ] 7.1 Create `app/views/goals/index.html.erb` with goal cards showing name, icon, goal_type badge, progress bar (progress_percent), current_amount / target_amount, status indicator, and projected completion date, ordered by priority
- [ ] 7.2 Create `app/views/goals/_goal_card.html.erb` partial for individual goal card with progress bar visualization

## Task 8: Views — Goal Detail Page

- [ ] 8.1 Create `app/views/goals/show.html.erb` with goal header (name, type, status badge, icon), progress section (progress bar, current/target amounts, remaining, projected completion, days remaining), linked accounts list, and contribution history
- [ ] 8.2 Add D3.js progress chart via Turbo frame showing historical progress over time using the existing time-series-chart Stimulus controller pattern
- [ ] 8.3 Create `app/views/goals/_contribution_list.html.erb` partial for chronological contribution/withdrawal history

## Task 9: Views — Goal Form

- [ ] 9.1 Create `app/views/goals/new.html.erb` and `app/views/goals/edit.html.erb` with shared `_form.html.erb` partial containing fields for name, goal_type select, target_amount, target_date, priority select, icon, color, and account multi-select (filtered to Current.family accounts)

## Task 10: Dashboard Widget

- [ ] 10.1 Create `app/views/goals/_dashboard_widget.html.erb` partial showing active goals summary with name, mini progress bar, and status badge
- [ ] 10.2 Integrate the goals widget into the dashboard page via Turbo frame

## Task 11: Property-Based Tests

- [ ] 11.1 (PBT) Property 2: Progress percent bounds — for any Goal with positive target_amount and any non-negative current_amount, progress_percent is between 0.0 and 100.0, and returns 0 when target_amount is zero
- [ ] 11.2 (PBT) Property 3: Remaining amount non-negativity — for any Goal, remaining_amount is always >= 0
- [ ] 11.3 (PBT) Property 4: Projected completion date — for any Goal with monthly_contribution > 0 and remaining_amount > 0, projected_completion_date is a future date; for monthly_contribution <= 0, it is nil
- [ ] 11.4 (PBT) Property 5: Status determination — for any Goal where current_amount >= target_amount, status is completed; for paused goals, status remains paused
- [ ] 11.5 (PBT) Property 7: Contribution amount non-zero — for any GoalContribution, amount is never zero
- [ ] 11.6 (PBT) Property 8: Custom goal progress — for any custom Goal, current_amount equals the sum of all GoalContribution amounts
- [ ] 11.7 (PBT) Property 12: Days remaining — for any Goal with target_date, days_remaining is nil when target_date is nil, 0 when past, and positive when future
