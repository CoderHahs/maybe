---
inclusion: manual
---

# Credit Score, Receipt Scanning & Retail Sync

Implements Monarch parity features #5 (Credit score tracking), #6 (Receipt scanning), #12 (Retail purchase sync).

## 1. Credit Score Tracking

### Concept

Display the user's credit score with historical tracking. Monarch pulls this from connected accounts or third-party providers.

### Domain Model

```
CreditScore (new model, belongs_to :user)
  - score: integer
  - score_type: enum (fico, vantage)
  - provider: string (e.g., "plaid", "manual")
  - factors: jsonb (array of positive/negative factors)
  - recorded_on: date
```

### Data Sources

- **Plaid**: Use Plaid's credit score product (Liabilities endpoint) for managed mode
- **Manual entry**: Allow users to manually log their score (self-hosted mode)
- **Provider concept**: Register as a new provider concept `credit_score` in `Provider::Registry` so alternative providers can be swapped in

### UI

- Credit score page showing current score with gauge visualization
- Historical score chart (D3 line chart, monthly data points)
- Score factors list (positive: on-time payments, low utilization; negative: high balances, etc.)
- Score range indicator (poor / fair / good / very good / excellent)
- Dashboard widget option showing current score and trend arrow

### Score Ranges

```ruby
# app/models/credit_score/rating.rb
def rating
  case score
  when 800..850 then :excellent
  when 740..799 then :very_good
  when 670..739 then :good
  when 580..669 then :fair
  else :poor
  end
end
```

## 2. Receipt Scanning

### Concept

Users photograph receipts. The system extracts line items, matches to a transaction, and optionally splits the transaction into item-level categories.

### Domain Model

```
Receipt (new model, belongs_to :entry)
  - image: ActiveStorage attachment
  - scanned_at: datetime
  - raw_text: text (OCR output)
  - status: enum (pending, processed, failed)
  - merchant_name: string (extracted)
  - total_amount: decimal (extracted)

ReceiptItem (new model, belongs_to :receipt)
  - description: string
  - amount: decimal
  - category_id: references (optional)
  - quantity: integer (default 1)
```

### Processing Pipeline

1. User uploads receipt image (Active Storage, already configured)
2. `ReceiptProcessingJob` runs asynchronously:
   a. OCR extraction via OpenAI Vision API (reuse existing OpenAI integration)
   b. Parse merchant, date, line items, total from OCR text
   c. Match to existing transaction by: merchant name similarity + amount + date (±2 days)
   d. If matched, create ReceiptItems and optionally auto-split the transaction
3. User reviews and confirms the match + categorization

### Transaction Splitting

When a receipt has multiple items in different categories, split the parent Entry:

```ruby
# app/models/entry/splitter.rb
class Entry::Splitter
  def initialize(entry, splits:)
    # splits = [{ amount: 25.00, category_id: 1 }, { amount: 15.00, category_id: 2 }]
  end

  def split!
    # Create child entries for each split
    # Mark parent as split (add `split: boolean` to entries)
    # Child entries reference parent via `parent_entry_id`
  end
end
```

### UI

- Upload button on transaction detail page
- Camera capture on mobile (HTML file input with `capture="environment"`)
- Receipt preview with extracted items
- Confirm/edit extracted data before saving
- Split transaction dialog showing item-level breakdown

## 3. Retail Purchase Sync (Amazon/Target Style)

### Concept

For self-hosted users, provide a mechanism to import order history from retailers and auto-split/categorize transactions.

### Approach

Since Maybe is self-hosted (no browser extension infrastructure like Monarch), implement as a CSV/data import:

```
RetailImport (new model, belongs_to :family)
  - retailer: enum (amazon, target, walmart, custom)
  - status: enum (pending, processing, completed, failed)
  - file: ActiveStorage attachment
  - matched_count: integer
  - unmatched_count: integer
```

### Processing

1. User downloads order history CSV from retailer (Amazon: "Order History Reports")
2. Upload to Maybe via import interface
3. `RetailImportJob` processes:
   a. Parse order items with dates, amounts, item descriptions
   b. Match to existing transactions by date + amount + merchant
   c. Split matched transactions into item-level entries with categories
   d. Use AI (OpenAI) to suggest categories for each item description
4. User reviews matches and confirms

### UI

- New import type in existing imports flow
- Retailer selection (Amazon, Target, etc.)
- Match review screen showing proposed splits
- Bulk confirm/reject matches

## Implementation Notes

- Receipt OCR uses existing OpenAI gem (ruby-openai) — no new dependency needed
- Active Storage already configured for file uploads
- Credit score provider follows existing Provider::Registry pattern
- Transaction splitting is a new concept — add `parent_entry_id` to entries table
- All processing via Sidekiq jobs (existing pattern)
- Receipt upload UI uses native HTML file input (Convention 3)
