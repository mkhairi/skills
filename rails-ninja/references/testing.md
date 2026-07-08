# Testing Reference (37signals Style: Minitest + Fixtures)

37signals uses **Minitest**, not RSpec. **Fixtures**, not FactoryBot. Tests ship with features
in the same commit — not strict TDD, not afterthought.

---

## Project Setup

```ruby
# Gemfile — no rspec, no factory_bot
group :test do
  gem "capybara"
  gem "selenium-webdriver"
end
```

---

## Test Organization

```
test/
  models/           # Unit tests — business logic
  controllers/      # Integration tests via ActionDispatch::IntegrationTest
  system/           # Full browser tests with Capybara
  jobs/             # Job tests
  mailers/          # Mailer tests
  fixtures/         # YAML fixture files
  test_helper.rb
```

---

## Fixtures (Not FactoryBot)

```yaml
# test/fixtures/cards.yml
logo:
  account: 37s
  board: writebook
  creator: david
  title: "Logo Design"
  number: 1
  status: published

shipping:
  account: 37s
  board: writebook
  creator: david
  title: "Shipping Labels"
  number: 2
  status: published
```

```yaml
# test/fixtures/closures.yml
shipping_closed:
  account: 37s
  card: shipping
  user: david
```

```yaml
# test/fixtures/users.yml
david:
  name: David Heinemeier Hansson
  email: david@example.com

kevin:
  name: Kevin
  email: kevin@example.com
  role: admin
```

### Using fixtures in tests
```ruby
# Access via helper method — cards(:logo), users(:david)
card = cards(:logo)
user = users(:david)
```

---

## Model Tests

```ruby
# test/models/card_test.rb
class CardTest < ActiveSupport::TestCase
  setup do
    Current.session = sessions(:david)
  end

  test "close creates a closure record" do
    card = cards(:logo)
    assert_not card.closed?

    assert_difference "Closure.count", 1 do
      card.close
    end

    assert card.reload.closed?
  end

  test "reopen destroys the closure" do
    card = cards(:shipping)
    assert card.closed?

    assert_difference "Closure.count", -1 do
      card.reopen
    end

    assert_not card.reload.closed?
  end

  test "closed scope returns only closed cards" do
    assert_includes Card.closed, cards(:shipping)
    assert_not_includes Card.closed, cards(:logo)
  end

  test "close is idempotent" do
    card = cards(:shipping)  # already closed
    assert_no_difference "Closure.count" do
      card.close
    end
  end
end
```

---

## Controller / Integration Tests (Preferred over unit controller tests)

```ruby
# test/controllers/cards/closures_controller_test.rb
class Cards::ClosuresControllerTest < ActionDispatch::IntegrationTest
  setup { sign_in_as :kevin }

  test "create closes the card" do
    card = cards(:logo)
    assert_not card.closed?

    assert_changes -> { card.reload.closed? }, from: false, to: true do
      post card_closure_path(card), as: :turbo_stream
    end

    assert_response :ok
  end

  test "destroy reopens the card" do
    card = cards(:shipping)
    assert card.closed?

    delete card_closure_path(card), as: :json

    assert_response :no_content
    assert_not card.reload.closed?
  end

  test "cannot close another account's card" do
    post card_closure_path(cards(:other_account_card)), as: :turbo_stream
    assert_response :not_found
  end
end
```

### Auth helper for integration tests
```ruby
# test/test_helper.rb
class ActionDispatch::IntegrationTest
  def sign_in_as(fixture_name)
    user = users(fixture_name)
    post sessions_path, params: { email: user.email }
    # or use a direct session fixture
    cookies.signed[:session_token] = sessions(fixture_name).signed_id
  end
end
```

---

## Request Tests (for JSON APIs)

```ruby
class Api::V1::PostsTest < ActionDispatch::IntegrationTest
  setup { sign_in_as :david }

  test "GET /api/v1/posts returns published posts" do
    get api_v1_posts_path, as: :json
    assert_response :ok
    assert_equal 2, response.parsed_body.size
  end

  test "POST /api/v1/posts creates a post" do
    assert_difference "Post.count", 1 do
      post api_v1_posts_path,
           params: { post: { title: "New", body: "Content" } },
           as: :json
    end
    assert_response :created
  end

  test "POST /api/v1/posts returns 422 on invalid params" do
    post api_v1_posts_path,
         params: { post: { title: "" } },
         as: :json
    assert_response :unprocessable_entity
    assert_includes response.parsed_body["errors"], "Title can't be blank"
  end
end
```

---

## System Tests (Capybara)

```ruby
# test/system/cards_test.rb
class CardsTest < ApplicationSystemTestCase
  setup { sign_in_as users(:david) }

  test "create a card" do
    visit board_path(boards(:writebook))
    click_on "New Card"

    fill_in "Title", with: "My new card"
    click_on "Save"

    assert_text "My new card"
  end

  test "close a card" do
    visit card_path(cards(:logo))
    click_on "Close"

    assert_text "Closed"
  end
end

# test/application_system_test_case.rb
class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  driven_by :selenium, using: :headless_chrome, screen_size: [1400, 1400]
end
```

---

## Job Tests

```ruby
class SendWelcomeEmailJobTest < ActiveJob::TestCase
  test "enqueues the job" do
    assert_enqueued_with(job: SendWelcomeEmailJob) do
      SendWelcomeEmailJob.perform_later(users(:david).id)
    end
  end

  test "sends the email" do
    assert_emails 1 do
      SendWelcomeEmailJob.perform_now(users(:david).id)
    end
  end
end
```

---

## Mailer Tests

```ruby
class UserMailerTest < ActionMailer::TestCase
  test "welcome email" do
    user = users(:david)
    email = UserMailer.welcome_email(user)

    assert_emails 1 do
      email.deliver_now
    end

    assert_equal [user.email], email.to
    assert_equal "Welcome!", email.subject
    assert_match user.name, email.body.encoded
  end
end
```

---

## Key Principles

1. **Tests ship with features** — same commit, always
2. **Security/bug fixes include regression tests** — prove it can't happen again
3. **Fixtures over factories** — deterministic, fast, no setup overhead
4. **Integration tests over unit controller tests** — test the HTTP layer
5. **Test behavior, not implementation** — `assert card.closed?`, not `assert closure.present?`
6. **Small, focused test methods** — one assertion per test where practical
7. **Use `assert_changes` / `assert_difference`** for state-changing operations
