---
inclusion: fileMatch
fileMatchPattern: "test/**"
---

# Testing Conventions

## Framework

- ALWAYS use Minitest + fixtures — NEVER RSpec or FactoryBot
- Mocking/stubbing via `mocha` gem
- VCR + WebMock for external API testing
- Capybara + Selenium for system tests
- SimpleCov for coverage (opt-in via `COVERAGE=true`)

## Test Structure

```
test/
  models/          # Model unit tests
  controllers/     # Controller tests
  components/      # ViewComponent tests
  helpers/         # Helper tests
  jobs/            # Background job tests
  mailers/         # Mailer tests
  system/          # System/integration tests (Capybara)
  integration/     # Integration tests
  interfaces/      # Interface tests
  services/        # Service tests
  lib/             # Library tests
  fixtures/        # YAML fixtures
  support/         # Test helpers (e.g., entries_test_helper.rb)
  vcr_cassettes/   # VCR recorded API responses
```

## Fixtures

- Keep fixtures minimal: 2-3 per model for base cases
- Edge cases created on-the-fly within the test context
- For tests needing many records, use helpers from `test/support/` (e.g., `entries_test_helper.rb`)

## What to Test

- Test critical domain business logic
- Test command boundaries (verify commands called with correct params)
- Test query outputs
- Do NOT test ActiveRecord built-in functionality
- Do NOT test implementation details of other classes in a class's test suite
- System tests sparingly — they slow down the suite

```ruby
# GOOD — testing critical domain logic
test "syncs balances" do
  Holding::Syncer.any_instance.expects(:sync_holdings).returns([]).once
  assert_difference "@account.balances.count", 2 do
    Balance::Syncer.new(@account, strategy: :forward).sync_balances
  end
end

# BAD — testing ActiveRecord functionality
test "saves balance" do
  balance_record = Balance.new(balance: 100, currency: "USD")
  assert balance_record.save
end
```

## Stubs and Mocks

- Use `mocha` gem
- Prefer `OpenStruct` for mock instances, or a mock class for complex cases
- Only mock what's necessary — don't mock return values if not testing them

## Test Helpers

```ruby
# Sign in a user
sign_in(user)

# Override environment variables
with_env_overrides(SYNTH_API_KEY: "test") { ... }

# Test in self-hosted mode
with_self_hosting { ... }

# Test password
user_password_test  # => "maybetestpassword817983172"
```

## Running Tests

```sh
bin/rails test                              # All tests
bin/rails test test/models/account_test.rb  # Specific file
bin/rails test test/models/account_test.rb:42  # Specific line
bin/rails test:system                       # System tests only
DISABLE_PARALLELIZATION=true bin/rails test:system  # System tests without parallelization
COVERAGE=true bin/rails test                # With coverage report
```

## Configuration

- Tests run in parallel by default (`parallelize workers: :number_of_processors`)
- Plaid set to sandbox mode in tests
- VCR filters sensitive data (API keys, tokens)
- All fixtures loaded for all tests
