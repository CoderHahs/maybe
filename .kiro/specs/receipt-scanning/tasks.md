# Tasks

## Task 1: Database Migrations

- [ ] 1.1 Create migration `add_parent_entry_id_to_entries` adding `parent_entry_id` as nullable UUID foreign key referencing entries table, with index
- [ ] 1.2 Create migration `create_receipts` with UUID primary key, family_id (not null, FK), entry_id (nullable, FK, unique index), status (string, not null, default "pending"), merchant_name, total_amount (decimal 19,4), currency, receipt_date, raw_text, scanned_at, timestamps, and composite index on [family_id, status]
- [ ] 1.3 Create migration `create_receipt_items` with UUID primary key, receipt_id (not null, FK), category_id (nullable, FK), description (string, not null), amount (decimal 19,4, not null), quantity (integer, default 1), timestamps, and index on receipt_id

## Task 2: Entry Parent-Child Relationship

- [ ] 2.1 Add `belongs_to :parent_entry, class_name: "Entry", optional: true` and `has_many :sub_entries, class_name: "Entry", foreign_key: :parent_entry_id, dependent: :destroy` to Entry model
- [ ] 2.2 Implement `split?` method (returns true when sub_entries exist) and `sub_entry?` method (returns true when parent_entry_id present) on Entry
- [ ] 2.3 Create `test/models/entry_split_test.rb` with tests for parent-child associations, `split?`, and `sub_entry?` methods

## Task 3: Receipt Model

- [ ] 3.1 Create `app/models/receipt.rb` with belongs_to :family, optional belongs_to :entry, has_many :receipt_items (dependent: destroy), has_one_attached :image, enum for status, validations (status presence, image presence, image content type, total_amount positive when present), scopes (chronological, unmatched)
- [ ] 3.2 Implement `matched?`, `total_items_amount`, and `items_match_total?` methods on Receipt
- [ ] 3.3 Add `has_many :receipts, dependent: :destroy` to Family model
- [ ] 3.4 Create `test/fixtures/receipts.yml` with fixture data covering each status
- [ ] 3.5 Create `test/models/receipt_test.rb` with unit tests for validations, scopes, matched?, total_items_amount, items_match_total?, and image content type validation

## Task 4: ReceiptItem Model

- [ ] 4.1 Create `app/models/receipt_item.rb` with belongs_to :receipt, optional belongs_to :category, validations (description presence, amount presence and greater than 0, quantity greater than 0 when present)
- [ ] 4.2 Create `test/fixtures/receipt_items.yml` with fixture data
- [ ] 4.3 Create `test/models/receipt_item_test.rb` with unit tests for validations and associations

## Task 5: Receipt::Processor PORO

- [ ] 5.1 Create `app/models/receipt/processor.rb` with initialize(receipt), process(ocr_response) method that parses response, updates receipt metadata, creates ReceiptItems, and calls auto_match_entry
- [ ] 5.2 Implement `parse_ocr_response` private method that extracts merchant, total, date, items array, and raw_text from OpenAI Vision JSON response with error handling for malformed data
- [ ] 5.3 Implement `auto_match_entry` private method that searches family entries for Transaction-type entries within 5% amount tolerance and 7-day date window, selecting closest date match
- [ ] 5.4 Create `test/models/receipt/processor_test.rb` with tests for: successful parsing, item creation, auto-matching within thresholds, no match outside thresholds, closest date selection, existing entry_id preservation, and malformed response handling

## Task 6: ReceiptProcessingJob

- [ ] 6.1 Create `app/jobs/receipt_processing_job.rb` with queue_as :medium_priority, perform method that encodes image to base64, calls OpenAI Vision API via Provider::Registry, invokes Receipt::Processor, and handles errors (set status to failed, log, re-raise)
- [ ] 6.2 Build the Vision API prompt as a private method that instructs OpenAI to return structured JSON with merchant, total, date, and items array
- [ ] 6.3 Create `test/jobs/receipt_processing_job_test.rb` with tests for: successful processing (stubbed OpenAI), failure handling (status set to failed), and missing provider error

## Task 7: Entry::Splitter PORO

- [ ] 7.1 Create `app/models/entry/splitter.rb` with initialize(entry), split(items_with_categories) method that validates entry is Transaction type, validates sum doesn't exceed parent, wraps in DB transaction, marks parent excluded, creates sub-entries with parent_entry_id, handles remainder, and triggers account sync
- [ ] 7.2 Create `test/models/entry/splitter_test.rb` with tests for: basic split, parent exclusion, parent_entry_id set on sub-entries, attribute inheritance (account, date, currency), sign preservation for outflow and inflow, remainder creation, exact amount split (no remainder), oversplit rejection, and account sync trigger

## Task 8: ReceiptsController and Routes

- [ ] 8.1 Add routes for receipts in `config/routes.rb`: resources :receipts (index, show, new, create, destroy) with member route for split action
- [ ] 8.2 Create `app/controllers/receipts_controller.rb` with before_action to scope to Current.family, index (paginated, chronological), show, new, create (attach image, enqueue job), destroy, and split (invoke Entry::Splitter)
- [ ] 8.3 Create `test/controllers/receipts_controller_test.rb` with tests for: index listing, show, create with image upload, destroy, split action, and family scoping (cannot access other family's receipts)

## Task 9: Views

- [ ] 9.1 Create `app/views/receipts/index.html.erb` with receipt list showing image thumbnail, merchant name, total amount, status badge, date, and matched/unmatched indicator, with pagination
- [ ] 9.2 Create `app/views/receipts/show.html.erb` with receipt image display, extracted metadata (merchant, total, date), line items table with description/amount/quantity, category select dropdowns per item, link to matched transaction, and "Split Transaction" button/form
- [ ] 9.3 Create `app/views/receipts/new.html.erb` with upload form using native HTML file input (`accept="image/jpeg,image/png,image/webp,image/heic"` and `capture="environment"`), optional entry_id select, and submit button
- [ ] 9.4 Create `app/views/receipts/_receipt.html.erb` partial for rendering a receipt in the index list
- [ ] 9.5 Create `app/views/receipts/_status_badge.html.erb` partial for rendering status badges (pending/processing/processed/failed) with appropriate colors

## Task 10: Navigation and Integration

- [ ] 10.1 Add "Receipts" link to the application navigation menu
- [ ] 10.2 Add receipt upload link/button on the transaction show page for easy receipt attachment to existing transactions

## Task 11: Property-Based Tests

- [ ] 11.1 (PBT) Property 11: Amount conservation — for any valid parent entry and set of item amounts that sum to ≤ |parent.amount|, the sum of all sub-entry amounts (including remainder) equals the parent entry amount exactly
- [ ] 11.2 (PBT) Property 12: Sign preservation — for any parent entry (positive or negative amount) and valid item set, all sub-entries have the same sign as the parent entry
- [ ] 11.3 (PBT) Property 14: Attribute inheritance — for any split, all sub-entries have the same account_id, date, and currency as the parent entry
- [ ] 11.4 (PBT) Property 8: Auto-match threshold correctness — for any receipt total_amount and entry amount, auto-match returns a match if and only if the amounts are within 5% and dates within 7 days
- [ ] 11.5 (PBT) Property 15: Remainder correctness — for any split where item amounts sum to less than |parent.amount|, a remainder entry exists with amount equal to the difference
