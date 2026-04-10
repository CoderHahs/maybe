---
inclusion: manual
---

# Enhanced AI Assistant & Smart Insights

Implements Monarch parity features #7 (Enhanced AI assistant) and #9 (Customizable dashboard widgets).

## 1. Enhanced AI Assistant

### Current State

Maybe already has an AI chat system (OpenAI) with `Chat`, `Message`, `Assistant` models and `AssistantResponseJob`.

### Enhancements Needed

**Weekly Financial Summary** (scheduled job):

Create `WeeklySummaryJob` (Sidekiq cron, runs Monday morning):

- Gather: total spending, income, net worth change, top categories, unusual transactions
- Generate a natural language summary via OpenAI
- Create a new message in the user's default chat
- Optionally send via email (if SMTP configured)

```ruby
# app/jobs/weekly_summary_job.rb
class WeeklySummaryJob < ApplicationJob
  def perform
    Family.find_each do |family|
      family.users.each do |user|
        summary = Family::WeeklySummary.new(family, user: user).generate
        chat = user.last_viewed_chat || user.chats.create!(title: "Weekly Summary")
        # Create system message with summary content
      end
    end
  end
end
```

**Trend Surfacing**:

Place in `app/models/family/trend_detector.rb`:

- Compare current month spending by category vs 3-month average
- Flag categories with >20% increase as "trending up"
- Flag unusual single transactions (>2x average for that merchant)
- Surface in AI chat context so assistant can reference them

**Proactive Insights**:

The AI assistant should have access to structured financial context:

- Current month budget status (over/under by category)
- Upcoming bills in next 7 days
- Goal progress (on track / behind)
- Net worth trend (up/down vs last month)
- Unreviewed transaction count

Inject this as system context into OpenAI prompts so the assistant can proactively mention relevant insights.

**Money Questions**:

Enhance the assistant to answer questions like:

- "How much did I spend on dining this month?"
- "What's my biggest expense category?"
- "Am I on track for my vacation goal?"
- "How does my spending compare to last month?"

This requires tool_call functions that query the database. Extend existing `ToolCall` model with new function definitions.

## 2. Customizable Dashboard Widgets

### Concept

Users can choose which widgets appear on their dashboard and reorder them. Monarch shows: cash flow, upcoming bills, budget progress, goals, net worth trends, investments.

### Domain Model

```
DashboardWidget (new model, belongs_to :user)
  - widget_type: enum (net_worth, cash_flow, budget_summary, goals,
      upcoming_bills, recent_transactions, investments, unreviewed_count,
      spending_by_category, income_vs_expenses)
  - position: integer
  - visible: boolean (default true)
  - config: jsonb (widget-specific settings like time range)
```

### Widget Registry

Each widget type maps to a ViewComponent:

```ruby
# app/models/dashboard_widget/registry.rb
WIDGETS = {
  net_worth:           UI::Dashboard::NetWorthWidget,
  cash_flow:           UI::Dashboard::CashFlowWidget,
  budget_summary:      UI::Dashboard::BudgetWidget,
  goals:               UI::Dashboard::GoalsWidget,
  upcoming_bills:      UI::Dashboard::UpcomingBillsWidget,
  recent_transactions: UI::Dashboard::RecentTransactionsWidget,
  investments:         UI::Dashboard::InvestmentsWidget,
  spending_by_category: UI::Dashboard::SpendingCategoryWidget,
  income_vs_expenses:  UI::Dashboard::IncomeExpensesWidget,
}
```

### Customization UI

- "Customize" button in dashboard header opens a dialog
- Drag-to-reorder (Stimulus controller with sortable behavior)
- Toggle visibility per widget
- Save via PATCH to dashboard_widgets endpoint
- Default widget set created on user registration

### Dashboard Rendering

```erb
<%# app/views/pages/dashboard.html.erb %>
<div class="grid grid-cols-1 md:grid-cols-2 gap-4">
  <% Current.user.dashboard_widgets.visible.ordered.each do |widget| %>
    <%= render DashboardWidget::Registry.component_for(widget) %>
  <% end %>
</div>
```

Each widget is a Turbo frame for independent loading/refresh.

## Implementation Notes

- Weekly summary uses existing Sidekiq cron infrastructure
- AI context injection extends existing assistant system prompt building
- Dashboard widgets are ViewComponents in `app/components/UI/Dashboard/`
- Widget reordering uses Stimulus (good client-side use case per Convention 3)
- All financial calculations in models (Convention 2)
- Use functional design tokens for widget cards (bg-container, border-primary)
