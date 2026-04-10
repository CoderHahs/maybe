---
inclusion: manual
---

# Goals, Planning & Debt Payoff

Implements Monarch parity features #3 (Enhanced goal planning), #14 (Flexible budget buckets), and #22 (Debt payoff strategies).

## 1. Goal Planning System

### Domain Model

```
Goal (new model, belongs_to :family)
  - name: string
  - goal_type: enum (savings, debt_payoff, retirement, custom)
  - target_amount: decimal
  - current_amount: decimal (computed or manual)
  - currency: string
  - target_date: date (optional)
  - account_id: references (optional — linked tracking account)
  - monthly_contribution: decimal (optional)
  - status: enum (on_track, behind, ahead, completed, paused)
  - icon: string (lucide icon key)
  - color: string (design system color)
  - priority: integer
```

### Goal Types

**Savings Goal**: Track progress toward a target amount. Can be linked to a specific account (e.g., savings account). Progress = account balance or manual tracking. Examples: emergency fund, vacation, down payment.

**Debt Payoff Goal**: Track progress paying down a liability account. Progress = reduction in loan/credit card balance from starting amount. Auto-calculated from linked liability account balance changes.

**Retirement Goal**: Single per-family goal. Aggregates all investment/retirement account balances. Shows projected growth based on current contribution rate and expected return. Target = retirement number.

**Custom Goal**: Freeform goal with manual progress tracking. For goals not tied to a specific account.

### Goal Progress Calculation

Place in `app/models/goal/progress_calculator.rb`:

- If linked to account: `current_amount` = account balance (for savings) or `starting_balance - current_balance` (for debt)
- Calculate `monthly_contribution` from average of last 3 months of inflows to linked account
- Project completion date: `(target_amount - current_amount) / monthly_contribution`
- Status: on_track if projected completion ≤ target_date, behind if after, ahead if before

### Goal Spending

Users can "spend from" a goal (reduce current_amount). This models withdrawals from savings goals and shows impact on timeline. Create `GoalTransaction` join to Entry for tracking.

### UI

- Goals index page with progress bars and status badges
- Goal detail page with historical progress chart (D3 time series)
- "Add Goal" dialog using DS::Dialog component
- Dashboard widget showing top 3 goals with progress

## 2. Flexible Budget Buckets

Enhance existing Budget model to support Monarch's Fixed / Flex / Non-monthly system:

### Budget Category Types

Add `bucket_type` enum to `BudgetCategory`:

- `fixed` — predictable recurring expenses (rent, insurance, subscriptions)
- `flex` — variable spending (groceries, dining, entertainment)
- `non_monthly` — irregular expenses (quarterly taxes, annual memberships)

### Flex Number

The "Flex Number" = remaining budget after fixed expenses and savings goals. This is the key insight Monarch provides — how much discretionary spending is available.

```ruby
# app/models/budget/flex_calculator.rb
class Budget::FlexCalculator
  def initialize(budget)
  def total_income          # projected monthly income
  def fixed_expenses        # sum of fixed bucket targets
  def savings_contributions # sum of goal monthly contributions
  def flex_number           # income - fixed - savings - non_monthly_amortized
end
```

### Non-Monthly Amortization

For non-monthly expenses, calculate monthly set-aside amount:

- Quarterly expense of $300 → $100/month set-aside
- Annual expense of $1200 → $100/month set-aside

## 3. Debt Payoff Strategies

### Snowball Method

Order debts by balance (smallest first). Minimum payments on all, extra goes to smallest balance. When one is paid off, roll payment into next.

### Avalanche Method

Order debts by interest rate (highest first). Minimum payments on all, extra goes to highest rate. Saves more in interest over time.

### Implementation

Place in `app/models/family/debt_payoff_plan.rb`:

```ruby
class Family::DebtPayoffPlan
  def initialize(family, strategy:, extra_monthly: 0)
  def ordered_debts        # sorted by strategy
  def payoff_schedule      # array of { account:, payoff_date:, total_interest: }
  def total_interest_paid  # sum across all debts
  def debt_free_date       # when last debt is paid off
end
```

Show as a timeline visualization with milestones for each debt payoff.

## Implementation Notes

- All logic in `app/models/` per Convention 2
- Goals page at `/goals` with standard RESTful routes
- Use Turbo frames for goal detail panels
- Progress charts reuse existing D3 time series patterns
- Budget bucket type is a migration adding column to budget_categories
