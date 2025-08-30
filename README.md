
# RSpec: Writing Maintainable Test Suites

Welcome to Lesson 24! In this lesson, we're going to get *seriously* opinionated about how to keep your RSpec test suite clean, maintainable, and effective as your Rails app grows. We'll cover best practices for test organization, naming, avoiding over-mocking, testing behavior (not implementation), keeping your suite fast, and making your tests a joy for your future self (and teammates) to work with. If you know Ruby and Rails but are new to automated testing, this is your roadmap to a healthy, long-lived test suite. We'll explain everything in multiple ways, with code, anti-patterns, and reflection prompts. Let's get your test garden thriving!

---

## Why Does Test Suite Maintainability Matter?

A test suite is like a garden: if you don't tend it, it gets messy, slow, and full of weeds (a.k.a. flaky tests). A clean, maintainable test suite:

- Catches bugs early
- Gives you confidence to refactor
- Saves time for everyone on your team
- Makes onboarding new devs easier

---

## Organizing Your Specs

**A well-organized test suite is easy to navigate, easy to extend, and easy to debug.**

- Group specs by type: `/spec/models/`, `/spec/requests/`, `/spec/system/`, `/spec/support/`, etc.
- Use descriptive file and example names. A good test name tells you what broke *before* you read the code.
- Use `describe`, `context`, and `it` blocks to structure your tests. Each block should read like a story:

```ruby
# /spec/models/user_spec.rb
RSpec.describe User, type: :model do
  describe "validations" do
    context "when username is missing" do
      it "is invalid" do
        user = User.new(username: nil)
        expect(user).not_to be_valid
      end
    end
  end
end
```

- Use shared examples and shared contexts to DRY up repeated logic:

```ruby
# /spec/support/shared_examples/soft_deletable.rb
RSpec.shared_examples "soft deletable" do
  it "can be soft deleted" do
    # ...
  end
end
```

- Prefer one expectation per `it` block. If you need more, make sure they're tightly related.

- Use `let` and `let!` for setup, and `before`/`after` hooks for repeated setup/teardown.

**Example:**

```ruby
# /spec/models/post_spec.rb
RSpec.describe Post, type: :model do
  let(:user) { create(:user) }

  before(:each) do
    @post = Post.create(title: "Hello", user: user)
  end

  after(:all) do
    Post.delete_all
  end

  it "belongs to a user" do
    expect(@post.user).to eq(user)
  end
end
```

**Anti-pattern:**

```ruby
# /spec/models/user_spec.rb
it "does a million things" do
  # ...100 lines of test code...
end
```

*Don't do this! Break it up for clarity and easier debugging.*

---

## Writing Descriptive `it` Blocks

Clear, descriptive test names make failures easy to understand.

```ruby
# ❌ Vague
it "does something" do
  # ...
end

# ✅ Descriptive
it "returns an error when email is missing" do
  # ...
end
```

---

## Test Behavior, Not Implementation

**Your tests should describe what your code does, not how it does it.**

- Avoid testing private methods or internal state. Test the public API.
- Focus on inputs and outputs, not the steps in between.
- If you refactor the internals, your tests shouldn't break (unless the behavior changes).

```ruby
# /spec/models/calculator_spec.rb
RSpec.describe Calculator, type: :model do
  it "adds two numbers" do
    calc = Calculator.new
    expect(calc.add(2, 3)).to eq(5)
  end
end
```

**Anti-pattern:**

```ruby
# /spec/models/calculator_spec.rb
RSpec.describe Calculator, type: :model do
  it "calls the add_numbers helper" do
    calc = Calculator.new
    expect(calc).to receive(:add_numbers).with(2, 3)
    calc.add(2, 3)
  end
end
```

*This test will break if you refactor, even if the behavior is the same!*

---

## Avoid Over-Mocking and Over-Stubbing

**Mocks and stubs are powerful, but overusing them can make your tests brittle and misleading.**

- Use doubles and mocks to isolate dependencies, but don't mock everything.
- Prefer real objects when possible—mocks can hide bugs and make refactoring harder.
- Only mock external services, slow dependencies, or things outside your control (APIs, emails, etc).

```ruby
# /spec/services/payment_processor_spec.rb
RSpec.describe PaymentProcessor, type: :model do
  it "calls the external API" do
    api = double("API")
    allow(api).to receive(:charge).and_return(true)
    processor = PaymentProcessor.new(api)
    expect(processor.charge(100)).to be true
  end
end
```

**Anti-pattern:**

```ruby
# /spec/models/user_spec.rb
RSpec.describe User, type: :model do
  it "mocks everything" do
    user = double("User", name: "Bob", email: "bob@example.com")
    expect(user.name).to eq("Bob")
  end
end
```

*This doesn't test your real code!*

---

## Keep Your Suite Fast

**A slow test suite is a test suite nobody runs.**

- Use `build_stubbed` instead of `create` when you don't need to hit the database (it's much faster!)

**Example:**

```ruby
# /spec/models/user_spec.rb
user1 = build_stubbed(:user) # Fast, doesn't save to DB
user2 = create(:user)        # Slower, saves to DB
```

- Use factories efficiently—avoid creating unnecessary records, especially in `before` blocks
- Tag slow specs with `:slow` and run them less often (e.g., only on CI)
- Profile your suite with tools like `rspec --profile` or gems like `test-prof`
- Use parallel test runners (e.g., `parallel_tests`, `knapsack_pro`) for big suites

**Pro Tip:**
Keep your feedback loop tight. Run only the specs you're working on with `rspec path/to/spec.rb`.

---

## Dealing with Flaky Tests

**Flaky tests are the enemy of trust.**

- Flaky tests are tests that sometimes pass and sometimes fail for no good reason
- Common causes: timeouts, randomness, external dependencies, race conditions, order dependencies

### Example: Flaky test due to time

```ruby
# /spec/models/event_spec.rb
RSpec.describe Event, type: :model do
  it "expires after a day" do
    event = Event.create!(expires_at: 1.day.from_now)
    travel_to 2.days.from_now do
      expect(event.expired?).to be true
    end
  end
end
```

### Example: Flaky test due to external API

```ruby
# /spec/services/weather_service_spec.rb
RSpec.describe WeatherService, type: :model do
  it "fetches the weather" do
    VCR.use_cassette("weather_api") do
      expect(WeatherService.new.fetch).to eq("sunny")
    end
  end
end
```

- Use `travel_to` or `Timecop` to control time
- Use `VCR` or `WebMock` to stub external HTTP requests
- Use `DatabaseCleaner` or Rails transactional fixtures to keep your DB clean between tests
- Investigate and fix flaky tests ASAP—they erode trust in your suite and slow down development

**Debugging Tip:**
Run flaky specs repeatedly with `rspec --seed 12345` or `rspec --bisect` to isolate the cause.

---

## Test Suite Organization Tips

- Use `/spec/support/` for helpers, shared examples, and custom matchers. Require files in `rails_helper.rb`:

```ruby
# /spec/rails_helper.rb
Dir[Rails.root.join('spec/support/**/*.rb')].each { |f| require f }
```

- Use `rails_helper.rb` for Rails-specific config, `spec_helper.rb` for general config
- Tag slow or special specs with `:slow`, `:js`, etc. and filter them with `rspec --tag js`
- Use `before(:all)` and `after(:all)` sparingly—prefer `before(:each)` for isolation
- Keep spec files short and focused. If a file is over 100 lines, consider splitting it up.

- Use custom matchers for repeated expectations

### Example: Custom matcher

```ruby
# /spec/support/matchers/be_valid_user.rb
RSpec::Matchers.define :be_valid_user do
  match do |user|
    user.valid? && user.email.present?
  end
end

# Usage in a spec:
RSpec.describe User, type: :model do
  it "is a valid user" do
    user = build(:user, email: "test@example.com")
    expect(user).to be_valid_user
  end
end
```

**Naming Tip:**
Name your spec files and examples so failures are self-explanatory. "does the thing" is not helpful—"returns error if email is missing" is!

---

## Debugging and Improving Your Suite

- Use `save_and_open_page` or `save_and_open_screenshot` (Capybara) to debug UI specs
- Use `binding.pry` or `puts` to debug failing specs
- Use `rspec --only-failures` to rerun only failed specs
- Regularly prune unused factories, helpers, and shared examples
- Review and refactor old specs as your app evolves

---

## Practice Prompts & Reflection Questions

Try these exercises to reinforce your learning:

1. Refactor a spec file to use shared examples or contexts. How does it improve readability and maintainability?
2. Identify a test that is over-mocking. How could you use a real object or integration test instead?
3. Find a slow spec and make it faster. What tools or techniques did you use? Did you use `build_stubbed`, parallelization, or better factories?
4. Write a spec that tests behavior, not implementation. How would you rewrite a test that checks private state?
5. Review your spec file and rename any vague `it` blocks to be more descriptive.
6. Add a custom matcher or shared example to DRY up repeated expectations.
7. Why is it important to keep your test suite fast and reliable? How does it affect your team's productivity?

Reflect: What could go wrong if your test suite is slow, flaky, or hard to understand? How would that impact your team, your users, and your ability to ship features?

---

## Resources

- [Better Specs: Organization](https://www.betterspecs.org/#organization)
- [Better Specs: Fast](https://www.betterspecs.org/#fast)
- [Better Specs: Naming](https://www.betterspecs.org/#it)
- [RSpec: Shared Examples](https://relishapp.com/rspec/rspec-core/v/3-10/docs/example-groups/shared-examples)
- [RSpec: Custom Matchers](https://relishapp.com/rspec/rspec-expectations/v/3-10/docs/custom-matchers/define-matcher)
- [Thoughtbot: Test Suite Maintainability](https://thoughtbot.com/blog/maintainable-test-suites)
- [Rails Guides: Testing Rails Applications](https://guides.rubyonrails.org/testing.html)
