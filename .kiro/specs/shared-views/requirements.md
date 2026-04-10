# Requirements Document

## Introduction

This document defines the requirements for the Shared Views with Yours/Mine/Ours Labeling feature in Maybe. The feature allows partners in a Family to label accounts as "personal" or "shared" and view their finances from different perspectives (All, Mine, Partner's, Shared Only) via a global view filter that applies across all major pages.

## Glossary

- **Ownership**: An enum on Account with values `shared` (default, "ours") or `personal` (belongs to a specific user).
- **Owner**: The User who owns a personal account, referenced by `owner_id` on Account.
- **View Filter**: A query parameter (`?view=all|mine|partner|shared`) that controls which accounts and their associated data are displayed.
- **View Switcher**: A UI component in the global nav that lets users switch between view perspectives.
- **OwnershipFilterable**: A model concern on Account providing the `for_view` scope.
- **ViewFilterable**: A controller concern that reads the `?view` param and provides filtered account helpers.

## Requirements

### Requirement 1: Account Ownership Data Model

**User Story:** As a family member, I want to label my accounts as "personal" or "shared" so that my partner and I can see finances from different perspectives.

#### Acceptance Criteria

1. THE Account model SHALL have an `ownership` enum with values "shared" and "personal", defaulting to "shared"
2. WHEN an Account's ownership is set to "shared", THE Account SHALL automatically set `owner_id` to NULL
3. WHEN an Account's ownership is set to "personal", THE Account SHALL require a non-null `owner_id`
4. WHEN an Account's `owner_id` is set, THE Account SHALL validate that the referenced User belongs to the same Family as the Account
5. THE migration SHALL set all existing accounts to ownership "shared" with NULL owner_id (backward compatible)
6. THE Account model SHALL include composite indexes on `(family_id, ownership)` and `(family_id, owner_id)` for query performance

### Requirement 2: View Filter Scoping (OwnershipFilterable)

**User Story:** As a user, I want to filter my financial view to see only my accounts, my partner's accounts, shared accounts, or everything, so that I can understand finances from different perspectives.

#### Acceptance Criteria

1. WHEN the view filter is "all", THE `for_view` scope SHALL return all accounts in the family (unfiltered)
2. WHEN the view filter is "mine", THE `for_view` scope SHALL return accounts where ownership is "shared" OR where ownership is "personal" and owner_id equals the current user's id
3. WHEN the view filter is "partner", THE `for_view` scope SHALL return accounts where ownership is "shared" OR where ownership is "personal" and owner_id does NOT equal the current user's id
4. WHEN the view filter is "shared", THE `for_view` scope SHALL return only accounts where ownership is "shared"
5. FOR any set of family accounts, the union of for_view("mine") and for_view("partner") SHALL equal for_view("all")

### Requirement 3: View Filter Controller Integration (ViewFilterable)

**User Story:** As a developer, I want a reusable controller concern that reads the view query param and provides filtered data helpers, so that all pages can consistently apply the view filter.

#### Acceptance Criteria

1. WHEN a request includes a `view` query param not in {"all", "mine", "partner", "shared"}, THE ViewFilterable concern SHALL default to "all"
2. THE ViewFilterable concern SHALL expose `current_view`, `view_filtered_accounts`, and `view_filter_params` as helper methods available in views
3. THE ViewFilterable concern SHALL provide `view_filter_params` that returns an empty hash for "all" (clean URLs) and `{ view: value }` otherwise
4. WHEN navigating between pages, THE view filter param SHALL be preserved in links generated with `view_filter_params`

### Requirement 4: View Filter Applied Across Pages

**User Story:** As a user, I want the view filter to apply consistently across dashboard, transactions, accounts, budgets, and net worth pages, so that I get a coherent filtered perspective everywhere.

#### Acceptance Criteria

1. WHEN the view filter is active, THE dashboard SHALL display net worth, assets, liabilities, and cashflow based only on the filtered accounts
2. WHEN the view filter is active, THE transactions page SHALL display only transactions from the filtered accounts
3. WHEN the view filter is active, THE accounts sidebar SHALL display only the filtered accounts
4. WHEN the view filter is active, THE budgets page SHALL calculate budget totals based only on transactions from the filtered accounts
5. WHEN the view filter is active, THE net worth / balance sheet SHALL calculate totals based only on the filtered accounts

### Requirement 5: View Switcher UI

**User Story:** As a user, I want a view switcher in the navigation so that I can quickly switch between financial perspectives.

#### Acceptance Criteria

1. WHEN a Family has only one User, THE view switcher SHALL NOT be rendered
2. WHEN a Family has more than one User, THE view switcher SHALL be rendered in the global layout area
3. THE view switcher SHALL display tabs or buttons for "All", "Mine", "Partner's", and "Shared" perspectives
4. THE view switcher SHALL visually indicate the currently active view
5. WHEN a user clicks a view tab, THE page SHALL reload with the corresponding `?view` query param

### Requirement 6: Account Ownership Settings

**User Story:** As a family admin, I want to set the ownership of each account from the account settings page, so that I can designate which accounts are personal and which are shared.

#### Acceptance Criteria

1. THE account edit/settings page SHALL include an ownership section with a dropdown for ownership (shared/personal) and a user selector for the owner
2. WHEN ownership is set to "shared", THE owner selector SHALL be hidden or disabled
3. WHEN ownership is set to "personal", THE owner selector SHALL display all users in the family
4. ONLY admin users SHALL be able to change account ownership settings
5. THE ownership section SHALL display the current ownership status and owner name (if personal)

### Requirement 7: Ownership Display on Account UI

**User Story:** As a user, I want to see which accounts are personal and who owns them, so that I understand the ownership structure at a glance.

#### Acceptance Criteria

1. WHEN an account is personal, THE account card/row SHALL display an ownership badge or indicator showing the owner's name
2. WHEN an account is shared, THE account card/row SHALL display a "Shared" badge or no special indicator (shared is the default)
3. THE ownership indicator SHALL be visible in the accounts sidebar and account detail pages

### Requirement 8: BalanceSheet Integration with View Filter

**User Story:** As a developer, I want the BalanceSheet model to accept a filtered account set so that net worth calculations respect the active view filter.

#### Acceptance Criteria

1. THE BalanceSheet SHALL accept an optional `accounts` parameter to scope its calculations
2. WHEN an `accounts` parameter is provided, THE BalanceSheet SHALL use only those accounts for asset/liability totals and net worth
3. WHEN no `accounts` parameter is provided, THE BalanceSheet SHALL use all visible family accounts (backward compatible)
