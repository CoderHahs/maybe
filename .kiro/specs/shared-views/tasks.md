# Tasks

## Task 1: Database Migration and Account Model Changes

- [ ] 1.1 Create migration `add_ownership_to_accounts` adding `ownership` string column (default: "shared", null: false), `owner_id` UUID column with foreign key to users table, and composite indexes on `(family_id, ownership)` and `(family_id, owner_id)`
- [ ] 1.2 Create `app/models/concerns/ownership_filterable.rb` with `ownership` enum (shared/personal, default: shared), `belongs_to :owner` (class_name: "User", optional: true), `for_view(view, user)` class method, `owner_required_when_personal` validation, `owner_belongs_to_family` validation, and `normalize_ownership` before_save callback that clears owner_id when shared
- [ ] 1.3 Include `OwnershipFilterable` concern in the Account model
- [ ] 1.4 Update `test/fixtures/accounts.yml` to add ownership and owner_id fields to existing fixtures, plus new fixtures for personal accounts owned by different users
- [ ] 1.5 Create `test/models/concerns/ownership_filterable_test.rb` with unit tests for: ownership enum defaults, owner_required_when_personal validation, owner_belongs_to_family validation, normalize_ownership callback, and `for_view` scope with all four view values

## Task 2: ViewFilterable Controller Concern

- [ ] 2.1 Create `app/controllers/concerns/view_filterable.rb` with `VALID_VIEWS` constant, `set_view_filter` before_action, `current_view` helper method, `view_filtered_accounts` helper method, and `view_filter_params` helper method
- [ ] 2.2 Include `ViewFilterable` in `ApplicationController` (so all controllers inherit it)
- [ ] 2.3 Create `test/controllers/concerns/view_filterable_test.rb` with tests for: valid view params, invalid view param defaults to "all", helper methods return correct values, view_filter_params returns empty hash for "all"

## Task 3: BalanceSheet Integration

- [ ] 3.1 Update `BalanceSheet#initialize` to accept an optional `accounts:` keyword argument; when provided, use it instead of querying all visible family accounts
- [ ] 3.2 Update `BalanceSheet::AccountTotals` to accept and use the optional filtered accounts scope
- [ ] 3.3 Update `PagesController#dashboard` to pass `view_filtered_accounts` to `BalanceSheet.new(Current.family, accounts: view_filtered_accounts)`
- [ ] 3.4 Add tests in `test/models/balance_sheet_test.rb` verifying BalanceSheet respects the filtered accounts parameter

## Task 4: Apply View Filter to Existing Controllers

- [ ] 4.1 Update `PagesController#dashboard` to use `view_filtered_accounts` for `@accounts`, `@balance_sheet`, and income statement calculations
- [ ] 4.2 Update `TransactionsController#index` to scope transactions through `view_filtered_accounts` instead of `Current.family.entries`
- [ ] 4.3 Update the accounts sidebar partial (`app/views/accounts/_account_sidebar_tabs.html.erb`) to filter displayed accounts using `view_filtered_accounts`
- [ ] 4.4 Update `BudgetsController` to scope budget calculations through `view_filtered_accounts`
- [ ] 4.5 Add integration tests verifying each page returns filtered data when `?view=mine` is passed

## Task 5: View Switcher UI

- [ ] 5.1 Create `app/views/shared/_view_switcher.html.erb` partial rendering tab buttons for All/Mine/Partner's/Shared, highlighting the active tab via `current_view`, linking to the current path with the `?view` param, and conditionally rendered only when `Current.family.users.count > 1`
- [ ] 5.2 Add the view switcher partial to the application layout (in the main content header area, near breadcrumbs)
- [ ] 5.3 Style the view switcher using existing DS::Tabs or TailwindCSS design tokens to match the app's visual language
- [ ] 5.4 Add a Stimulus controller (if needed) to handle preserving the view param when navigating via Turbo — or ensure `view_filter_params` is used in all relevant link helpers

## Task 6: Account Ownership Settings UI

- [ ] 6.1 Add ownership fields to the account edit/settings form: ownership dropdown (shared/personal) and owner selector (dropdown of family users), with the owner selector shown/hidden based on ownership value using Stimulus
- [ ] 6.2 Update `AccountsController` (or the relevant account settings controller) to permit `ownership` and `owner_id` params, with a before_action restricting ownership changes to admin users
- [ ] 6.3 Add ownership badge/indicator to account cards in the sidebar and account detail pages — show owner name for personal accounts, "Shared" or nothing for shared accounts
- [ ] 6.4 Add controller tests verifying: admin can update ownership, member cannot update ownership, ownership params are permitted correctly

## Task 7: Property-Based Tests

- [ ] 7.1 (PBT) Property 2: View filter completeness — for any family with mixed ownership accounts and any user, the union of for_view("mine", user) and for_view("partner", user) equals for_view("all", user)
- [ ] 7.2 (PBT) Property 3: Personal account exclusivity — for any personal account and any user, the account appears in exactly one of for_view("mine", user) or for_view("partner", user)
- [ ] 7.3 (PBT) Property 4: Shared accounts always visible — for any shared account and any valid view value, the account appears in the for_view result
- [ ] 7.4 (PBT) Property 5: Owner validation integrity — for any account with ownership "personal", the owner_id references a user in the same family; mismatched owner_id is rejected
- [ ] 7.5 (PBT) Property 6: Ownership normalization — for any account updated to ownership "shared", owner_id is NULL after save
- [ ] 7.6 (PBT) Property 7: View param sanitization — for any string not in {"all", "mine", "partner", "shared"}, ViewFilterable defaults to "all"
