# Requirements Document

## Introduction

This document defines the requirements for the Receipt Scanning with Item-Level Categorization feature in Maybe. The feature enables users to upload receipt images, extract line items via OCR (OpenAI Vision API), auto-match receipts to existing transactions, and split transactions into item-level sub-entries with individual category assignments.

## Glossary

- **Receipt**: A model storing an uploaded receipt image (Active Storage), OCR results, and optional link to an Entry.
- **ReceiptItem**: A line item extracted from a receipt via OCR, with description, amount, quantity, and optional category.
- **Receipt::Processor**: A PORO that parses OCR responses, creates ReceiptItems, and auto-matches receipts to entries.
- **Entry::Splitter**: A PORO that splits a parent transaction entry into multiple sub-entries by category based on receipt items.
- **Parent Entry**: The original transaction entry that gets split; marked as `excluded: true` after splitting.
- **Sub-Entry**: A child entry created by splitting, referencing the parent via `parent_entry_id`.
- **Auto-Match**: Automatic association of a receipt to an existing transaction by merchant name, amount proximity, and date proximity.
- **OCR**: Optical Character Recognition performed by OpenAI Vision API to extract text and structured data from receipt images.

## Requirements

### Requirement 1: Receipt Data Model

**User Story:** As a user, I want to upload receipt images that are stored with their processing status and extracted data, so that I can track and manage my receipts digitally.

#### Acceptance Criteria

1. THE Receipt SHALL belong to a Family and have a required status field with values: pending, processing, processed, or failed
2. THE Receipt SHALL have an Active Storage image attachment that is required and validated for content types: image/jpeg, image/png, image/webp, or image/heic
3. THE Receipt SHALL optionally belong to an Entry, with a unique constraint on entry_id (one receipt per entry)
4. THE Receipt SHALL store merchant_name (string), total_amount (decimal, positive when present), currency (string), receipt_date (date), raw_text (text), and scanned_at (datetime)
5. THE Receipt SHALL provide scopes: chronological (ordered by created_at desc) and unmatched (entry_id nil, status processed)
6. THE Receipt SHALL report `matched?` as true when entry_id is present, and calculate `total_items_amount` as the sum of its receipt_items' amounts

#### Correctness Properties

- **Property 1 (Validation Completeness):** For any Receipt, if status is not in {pending, processing, processed, failed} OR image is not attached, the record SHALL be invalid
- **Property 2 (Amount Positivity):** For any Receipt with a non-nil total_amount, total_amount SHALL be greater than 0
- **Property 3 (Unique Entry Association):** For any two Receipts r1 and r2 where r1.id ≠ r2.id, if both have non-nil entry_id, then r1.entry_id ≠ r2.entry_id

### Requirement 2: ReceiptItem Data Model

**User Story:** As a user, I want extracted line items from my receipts stored individually, so that I can review, categorize, and use them to split transactions.

#### Acceptance Criteria

1. THE ReceiptItem SHALL belong to a Receipt with required description (string) and amount (decimal, greater than 0)
2. THE ReceiptItem SHALL optionally belong to a Category and have an optional quantity (integer, greater than 0 when present, default 1)
3. WHEN a ReceiptItem is created with a zero or negative amount, THE validation SHALL fail

#### Correctness Properties

- **Property 4 (Item Amount Positivity):** For any valid ReceiptItem, amount SHALL be greater than 0
- **Property 5 (Quantity Positivity):** For any ReceiptItem with non-nil quantity, quantity SHALL be greater than 0

### Requirement 3: OCR Processing Pipeline

**User Story:** As a user, I want my uploaded receipts to be automatically processed in the background to extract merchant name, total, date, and line items, so that I don't have to manually enter receipt data.

#### Acceptance Criteria

1. WHEN a Receipt is created with status pending, THE ReceiptProcessingJob SHALL be enqueued to process it asynchronously via Sidekiq
2. THE ReceiptProcessingJob SHALL send the receipt image (base64 encoded) to the OpenAI Vision API and receive a structured JSON response
3. THE Receipt::Processor SHALL parse the OCR response and update the receipt with merchant_name, total_amount, receipt_date, raw_text, and scanned_at
4. THE Receipt::Processor SHALL create a ReceiptItem for each line item in the OCR response with description, amount, and quantity
5. WHEN the OpenAI API call fails or the response is unparseable, THE Receipt status SHALL be set to failed and the raw response stored in raw_text
6. WHEN the OpenAI provider is not configured, THE job SHALL fail with a descriptive error and the receipt SHALL remain in pending status

#### Correctness Properties

- **Property 6 (Processing Completeness):** For any Receipt that completes processing successfully, status SHALL be "processed" AND scanned_at SHALL be non-nil AND at least merchant_name or total_amount SHALL be present
- **Property 7 (Item Creation Integrity):** For any successfully processed Receipt, the count of ReceiptItems SHALL equal the count of line items in the OCR response

### Requirement 4: Transaction Auto-Matching

**User Story:** As a user, I want my receipts to be automatically matched to existing transactions when possible, so that I can link receipts to my financial records without manual effort.

#### Acceptance Criteria

1. WHEN a Receipt has no entry_id after OCR processing, THE Receipt::Processor SHALL search for matching entries in the same family
2. THE auto-match SHALL find entries where: the entry is a Transaction type, the amount is within 5% of the receipt total_amount, and the date is within 7 days of receipt_date
3. WHEN multiple candidate entries match, THE Receipt::Processor SHALL select the one with the closest date to receipt_date
4. WHEN a Receipt already has an entry_id before processing, THE auto-match SHALL not overwrite it
5. WHEN no matching entry is found, THE Receipt entry_id SHALL remain nil (unmatched)

#### Correctness Properties

- **Property 8 (Match Threshold Correctness):** For any receipt and entry pair, auto-match SHALL return a match if and only if |entry.amount| is within 5% of receipt.total_amount AND |entry.date - receipt.receipt_date| ≤ 7 days
- **Property 9 (Existing Match Preservation):** For any Receipt with a non-nil entry_id before processing, entry_id SHALL remain unchanged after processing
- **Property 10 (Best Match Selection):** For any set of candidate entries, the selected match SHALL have the minimum |entry.date - receipt.receipt_date| among all candidates

### Requirement 5: Transaction Splitting

**User Story:** As a user, I want to split a transaction into multiple sub-entries based on receipt line items, so that I can categorize individual items from a single purchase.

#### Acceptance Criteria

1. THE Entry::Splitter SHALL accept a parent entry and an array of items with {description, amount, category_id} and create sub-entries within a database transaction
2. THE Entry::Splitter SHALL mark the parent entry as excluded (excluded: true) so it is not double-counted in analytics
3. EACH sub-entry SHALL have parent_entry_id set to the parent entry's id, and inherit account_id, date, and currency from the parent
4. THE Entry::Splitter SHALL preserve the amount sign convention: if parent amount is positive (outflow), sub-entry amounts are positive; if negative (inflow), sub-entry amounts are negative
5. WHEN the sum of item amounts does not equal the parent entry's absolute amount, THE Entry::Splitter SHALL create a remainder entry for the difference
6. THE total of all sub-entry amounts (including remainder) SHALL equal the parent entry amount exactly
7. WHEN the sum of item amounts exceeds the parent entry's absolute amount, THE Entry::Splitter SHALL raise an error and create no entries
8. THE Entry::Splitter SHALL trigger an account sync after a successful split
9. THE Entry::Splitter SHALL only accept Transaction-type entries (not Valuation or Trade)

#### Correctness Properties

- **Property 11 (Amount Conservation):** For any successful split, the sum of all sub-entry amounts SHALL equal the parent entry amount
- **Property 12 (Sign Preservation):** For any sub-entry created by splitting, sign(sub_entry.amount) SHALL equal sign(parent_entry.amount) when sub_entry.amount ≠ 0
- **Property 13 (Parent Exclusion):** For any successfully split entry, the parent entry's excluded field SHALL be true
- **Property 14 (Attribute Inheritance):** For any sub-entry, account_id, date, and currency SHALL equal the parent entry's account_id, date, and currency
- **Property 15 (Remainder Correctness):** For any split where item amounts sum to less than |parent.amount|, a remainder entry SHALL exist and |remainder.amount| = |parent.amount| - sum(item amounts)
- **Property 16 (No Oversplit):** For any set of items where sum(amounts) > |parent.amount|, the split SHALL be rejected with no entries created

### Requirement 6: Entry Parent-Child Relationship

**User Story:** As a developer, I want entries to support a parent-child relationship, so that split transactions can be tracked and managed as a group.

#### Acceptance Criteria

1. THE entries table SHALL have a parent_entry_id column (nullable UUID, foreign key to entries)
2. THE Entry model SHALL have `has_many :sub_entries` (entries where parent_entry_id = self.id) and `belongs_to :parent_entry, optional: true`
3. THE Entry SHALL report `split?` as true when it has one or more sub_entries
4. THE Entry SHALL report `sub_entry?` as true when parent_entry_id is present

#### Correctness Properties

- **Property 17 (Split Consistency):** For any Entry, `split?` SHALL be true if and only if sub_entries.count > 0
- **Property 18 (Sub-Entry Consistency):** For any Entry, `sub_entry?` SHALL be true if and only if parent_entry_id is not nil

### Requirement 7: Receipts Controller and UI

**User Story:** As a user, I want to upload, view, and manage my receipts through a web interface with family-scoped access, so that I can interact with my receipt data securely.

#### Acceptance Criteria

1. THE ReceiptsController SHALL scope all queries to Current.family, preventing access to other families' receipts
2. THE upload form SHALL use a native HTML file input with `accept="image/jpeg,image/png,image/webp,image/heic"` and `capture="environment"` for mobile camera access
3. THE receipt index SHALL display receipts ordered by creation date with status badges (pending, processed, failed) and support pagination
4. THE receipt show page SHALL display the receipt image, extracted metadata (merchant, total, date), line items with amounts, and category assignment controls
5. WHEN a receipt is matched to an entry, THE show page SHALL display a link to the associated transaction
6. THE show page SHALL provide a "Split Transaction" action that sends item category assignments to Entry::Splitter
7. WHEN a user uploads a receipt, THE controller SHALL create the Receipt and enqueue ReceiptProcessingJob, then redirect to the receipt show page

#### Correctness Properties

- **Property 19 (Family Scoping):** For any request to ReceiptsController, only receipts belonging to Current.family SHALL be accessible

### Requirement 8: Image Validation and Security

**User Story:** As a user, I want my receipt uploads to be validated for size and format, so that the system handles only appropriate files securely.

#### Acceptance Criteria

1. THE Receipt SHALL reject images larger than 10MB with a validation error
2. THE Receipt SHALL reject images with content types not in the allowed list (image/jpeg, image/png, image/webp, image/heic)
3. THE Receipt image SHALL be served via Active Storage signed URLs with expiration

#### Correctness Properties

- **Property 20 (Size Limit):** For any uploaded file larger than 10MB, Receipt validation SHALL fail
