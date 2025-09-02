
# RSpec: Writing Maintainable Test Suites

In this lesson, we're going to get *seriously* opinionated about how to keep your RSpec test suite clean, maintainable, and effective as your Rails app grows. We'll cover best practices for test organization, naming, avoiding over-mocking, testing behavior (not implementation), keeping your suite fast, and making your tests a joy for your future self (and teammates) to work with. If you know Ruby and Rails but are new to automated testing, this is your roadmap to a healthy, long-lived test suite. We'll explain everything in multiple ways, with code, anti-patterns, and reflection prompts. Let's get your test garden thriving!

---

## Testing Core Values

Before diving into specific guidelines, it's important to understand the three core values that drive our testing philosophy:

### ✏️ Documentation

A successful spec describes the functionality of the code it covers. This is especially important for features with complex processes and/or unexpected logic. The combination of English-language descriptions and programming language assertions transforms specs into a powerful implementation outline. When writing tests, think of them as documentation that helps ensure we're doing the right thing.

### 📖 Readability

Perhaps more than any other type of code, tests are meant to be readable. How can documentation be effective if it's not readable? The idea is to use them to gain clarity on how the system is expected to function. Specs should be concise and informative.

### 🛡️ Protection

We often write features and then walk away from them for months/years without further thought. Proper spec coverage ensures code continues to function as intended. Specs allow us to work on related parts of the codebase without unintentionally altering established functionality and disrupting workflows.

---

## Why Does Test Suite Maintainability Matter?

A test suite is like a garden: if you don't tend it, it gets messy, slow, and full of weeds (a.k.a. flaky tests). A clean, maintainable test suite:

- Catches bugs early
- Gives you confidence to refactor
- Saves time for everyone on your team
- Makes onboarding new devs easier

---

## Use Request Specs

Request specs in Rails offer a more comprehensive evaluation of the entire application stack (from the request to the response) compared to controller specs that focus narrowly on individual controller actions. This broader scope better simulates real user interactions, thereby catching edge cases and integration issues more effectively. We prefer request specs as they align with the RSpec and Rails community recommendations, simplify test setups, and improve the maintainability and accuracy of our tests.

**❌ Bad (Controller Spec):**

```ruby
require 'rails_helper'

RSpec.describe SessionsController, type: :controller do
  describe "POST #create" do
    context "with valid credentials" do
      it "logs in the user and redirects to the dashboard" do
        user = FactoryBot.create(:user, username: 'testuser', password: 'securepassword')
        post :create, params: { session: { username: 'testuser', password: 'securepassword' } }
        expect(response).to redirect_to(dashboard_path)
      end
    end
  end
end
```

**✅ Good (Request Spec):**

```ruby
require 'rails_helper'

RSpec.describe "User Authentication", type: :request do
  describe "POST /login" do
    context "with valid credentials" do
      it "logs in the user and redirects to the dashboard" do
        user = FactoryBot.create(:user, username: 'testuser', password: 'securepassword')
        post login_path, params: { session: { username: 'testuser', password: 'securepassword' } }
        expect(response).to redirect_to(dashboard_path)
      end
    end
  end
end
```

---

## Organizing Your Specs

**A well-organized test suite is easy to navigate, easy to extend, and easy to debug.**

- Group specs by type: `/spec/models/`, `/spec/requests/`, `/spec/system/`, `/spec/support/`, etc.
- Use descriptive file and example names. A good test name tells you what broke *before* you read the code.
- Use `describe`, `context`, and `it` blocks to structure your tests. Each block should read like a story.

### Describing Your Methods

Be clear about what method you are describing. Use the Ruby documentation convention of `.` when referring to a class method's name and `#` when referring to an instance method's name.

**❌ Bad:**

```ruby
describe "the authenticate method for User" do
describe "if the user is an admin" do
```

**✅ Good:**

```ruby
describe ".authenticate" do
  # your class-level method test here
end

describe "#admin?" do
  # your instance-level method test here
end
```

### Using Contexts

Contexts are a powerful method to make your tests clear and well organized. When describing a context, start its description with 'when', 'with' or 'without'. Please note that contexts should be used simply -- they lose their effectiveness if they are so deeply-nested that the tests are no longer easy to read.

**❌ Bad:**

```ruby
it "has 200 status code if logged in" do
  expect(response).to respond_with 200
end

it "has 401 status code if not logged in" do
  expect(response).to respond_with 401
end
```

**✅ Good:**

```ruby
context "when logged in" do
  it { is_expected.to respond_with 200 }
end

context "when logged out" do
  it { is_expected.to respond_with 401 }
end
```

### Keep Your Description Short

A spec description should never be longer than 40 characters. This helps maintain readability in test output.

**❌ Bad:**

```ruby
it "has 422 status code if an unexpected params will be added" do
```

**✅ Good:**

```ruby
context "when not valid" do
  it { is_expected.to respond_with 422 }
end
```

**Example:**

```ruby
# /spec/models/user_spec.rb
RSpec.describe User, type: :model do
  describe "#admin?" do
    context "when user has admin role" do
      it { is_expected.to be_admin }
    end
  end
end
```

- Use shared examples and shared contexts to DRY up repeated logic, but beware of the extra cognitive complexity and maintenance it adds:

```ruby
# /spec/support/shared_examples/soft_deletable.rb
RSpec.shared_examples "soft deletable" do
  it "can be soft deleted" do
    # ...
  end
end
```

### DAMP vs DRY

DRY (Don't Repeat Yourself) is a programming principle that aims to reduce duplication in code. However, it can be harder to figure out what is being tested when all of the logic is outside of the actual test. You should aim to make tests readable and easy to understand, even if you duplicate some bits of code. This is sometimes known as DAMP (Descriptive and Meaningful Phrases).

The aim is to strike a balance between DAMP and DRY and be okay with some duplication to help increase readability.

### Use let and let! Concisely

When you have to assign a variable, instead of using a before block to create an instance variable, you can use `let`. The drawback to these methods is that the logic can live outside your `it` block and be used in multiple places, making it more difficult to understand what is happening.

Please use these powerful methods sparingly and with judgment. A good rule of thumb is only to use them at the top level of your rspec file, just under the class definition.

**❌ Bad:**

```ruby
describe ".in_time_zone" do
  before  { @territory = FactoryBot.create :territory, time_zone: "Arizona" }

  it "returns territories in the right timezone" do
    expect(Territory.in_time_zone("Arizona")).to include @territory
  end
end
```

**✅ Good:**

```ruby
describe ".in_time_zone" do
  it "returns territories in the right timezone" do
    phx = create :territory, time_zone: "Arizona"
    expect(Territory.in_time_zone("Arizona")).to include phx
  end
end
```

### Using Subject

Similar to `let`, if you have several tests related to the same subject, you can use `subject{}` to DRY them up. However, this should be used thoughtfully. Think of it as syntactic sugar over `let` and use it to help make your specs readable.

**❌ Bad:**

```ruby
it { expect(assigns('message')).to match /it was born in Belville/ }
```

**✅ Good:**

```ruby
subject { assigns("message") }

it { is_expected.to match /it was born in Billville/ }
```

### Single Expectation Testing

Each test should make only one assertion in isolated unit specs. Multiple expectations in the same example are a signal that you may be specifying multiple behaviors.

**✅ Good (isolated):**

```ruby
it { is_expected.to respond_with_content_type(:json) }
it { is_expected.to assign_to(:resource) }
```

In tests that are not isolated (e.g. ones that integrate with a DB, an external webservice, or end-to-end-tests), you take a massive performance hit to do the same setup over and over again. In these slower tests, it's fine to specify more than one isolated behavior.

When creating tests with multiple expectations, `aggregate_failures` can be used to see all the failures at once:

**✅ Good (not isolated):**

```ruby
it "creates a resource", :aggregate_failures do
  expect(response).to respond_with_content_type(:json)
  expect(response).to assign_to(:resource)
end
```

**Example:**

```ruby
# /spec/models/post_spec.rb
RSpec.describe Post, type: :model do
  let(:user) { create(:user) }

  it "belongs to a user" do
    post = create(:post, user: user)
    expect(post.user).to eq(user)
  end
end
```

### Testing All Possible Cases

Testing is a good practice, but if you do not test the edge cases, it will not be useful. Test valid, edge and invalid cases in order to provide proper protection.

**❌ Bad:**

```ruby
it "shows the resource"
```

**✅ Good:**

```ruby
describe "#destroy" do
  context "when resource is found" do
    it "responds with 200"
    it "shows the resource"
  end

  context "when resource is not found" do
    it "responds with 404"
  end

  context "when resource is not owned" do
    it "responds with 404"
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

## Expect vs Should Syntax

Always use the expect syntax. Never use the should syntax.

**❌ Bad:**

```ruby
it "creates a resource" do
  response.should respond_with_content_type(:json)
end
```

**✅ Good:**

```ruby
it "creates a resource" do
  expect(response).to respond_with_content_type(:json)
end
```

Configure RSpec to only accept the expect syntax:

```ruby
# spec_helper.rb
RSpec.configure do |config|
  config.expect_with :rspec do |c|
    c.syntax = :expect
  end
end
```

On one line expectations or with implicit subject, use `is_expected.to`:

**❌ Bad:**

```ruby
context "when not valid" do
  it { should respond_with 422 }
end
```

**✅ Good:**

```ruby
context "when not valid" do
  it { is_expected.to respond_with 422 }
end
```

---

## Writing Descriptive Test Names

Clear, descriptive test names make failures easy to understand. Avoid using "should" in test descriptions.

**❌ Bad:**

```ruby
it "should not change timings" do
  # ...
end

it "does something" do
  # ...
end
```

**✅ Good:**

```ruby
it "does not change timings" do
  expect(consumption.occur_at).to eq(valid.occur_at)
end

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

As a general rule, do not (over)use mocks and test real behavior when possible, as testing real cases is useful when validating your application flow.

**Critical Rule: Mocking should never be used on the subject of your tests.** Otherwise, you run the risk of false-positives and test behavior that is too different from production behavior to provide any real protection.

- Use doubles and mocks to isolate dependencies, but don't mock everything.
- Prefer real objects when possible—mocks can hide bugs and make refactoring harder.
- Only mock external services, slow dependencies, or things outside your control (APIs, emails, etc).
- When mocking, use `instance_double` when possible; use straight `double` for duck typing.

**✅ Good:**

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

**❌ Anti-pattern:**

```ruby
# /spec/models/user_spec.rb
RSpec.describe User, type: :model do
  it "mocks everything" do
    user = double("User", name: "Bob", email: "bob@example.com")
    expect(user.name).to eq("Bob")  # This doesn't test your real code!
  end
end
```

---

## Use Factories, Not Fixtures

Do not use fixtures because they are difficult to control; use factories instead. FactoryBot is a powerful tool to help create objects that mimic data in production. Use them to reduce the verbosity on creating new data.

**❌ Bad:**

```ruby
user = User.create(
  name: "Genoveffa",
  surname: "Piccolina",
  city: "Billyville",
  birth: "17 August 1982",
  active: true
)
```

**✅ Good:**

```ruby
user = FactoryBot.create :user
```

### FactoryBot Basics

The default factory (for an ActiveRecord model) should be the minimal valid instance of the model. For example, `FactoryBot.create(:my_model)` should not fail, but attributes and associations that are not required to be present for the model to be valid should not be defined on the default factory.

If you want to extend and/or specialize the default factory using traits, use names that reinforce readability (e.g., `FactoryBot.create(:user, :remodeling_consultant)` or `FactoryBot.create(:territory, :phoenix)`).

### FactoryBot: Create Only the Data You Need

Test suites can be heavy to run. To solve this problem, it's important not to load more data than needed. FactoryBot can create a cascading effect of multiple created objects that may not be needed, making specs slower.

When in doubt, use `#build` to instantiate objects rather than `#create` and think clearly about ways you can tell the correct effects have happened without making trips to the database. `#build` will instantiate the objects without saving them to the database and thus save time.

Other options to explore are `#build_stubbed` and `#attributes_for`.

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

## Stubbing HTTP Requests

Sometimes you need to access external services. In these cases you can't rely on the real service but you should stub it with solutions like WebMock. **Never run tests that call external dependencies.**

**✅ Good:**

```ruby
context "with unauthorized access" do
  let(:uri) { "http://api.lelylan.com/types" }

  before do
    stub_request(:get, uri).to_return(status: 401, body: fixture("401.json"))
  end

  it "gets a not authorized notification" do
    page.driver.get uri
    expect(page).to have_content "Access denied"
  end
end
```

Learn more about [WebMock](https://github.com/bblimke/webmock) and [VCR](https://github.com/vcr/vcr).

---

## Use Readable Matchers

Use readable matchers and double check the available RSpec matchers.

**❌ Bad:**

```ruby
lambda { model.save! }.to raise_error Mongoid::Errors::DocumentNotFound
```

**✅ Good:**

```ruby
expect { model.save! }.to raise_error Mongoid::Errors::DocumentNotFound
```

---

## Shared Examples

You can use shared examples to DRY your test suite up, but beware of the extra cognitive complexity and maintenance it adds and use them thoughtfully.

**❌ Bad:**

```ruby
describe "GET /devices" do
  let!(:resource) { FactoryBot.create :device, created_from: user.id }
  let!(:uri)      { "/devices" }
  # ... lots of repeated test logic
end
```

**✅ Good:**

```ruby
describe "GET /devices" do
  let!(:resource) { FactoryBot.create :device, created_from: user.id }
  let!(:uri)       { "/devices" }

  it_behaves_like "a listable resource"
  it_behaves_like "a paginable resource"
  it_behaves_like "a searchable resource"
  it_behaves_like "a filterable list"
end
```

---

## Dealing with Flaky Tests

**Flaky tests are the enemy of trust.**

Flaky tests are tests that sometimes pass and sometimes fail for no good reason. We all depend on each other to keep our code running smoothly, and few places demonstrate this as much as our test suites.

### Reporting Random Failures

Please report random, unrelated failures that happen on both the master branch and on PRs. A report should include the following:

- Which component it is related to
- Which spec it's related to (path/to/spec.rb:linenumber123)
- Which seed it ran with randomly
- A link to the failure on CI

**❌ Bad:**

```md
Here's a spec that failed today: https://ci.powerapp.cloud/...
```

**✅ Good:**

```md
[Flaky spec in the ApiChai component](https://ci.powerapp.cloud/...)

`components/api_chai/spec/api_chai_spec.rb:4`

ran with seed 12345
```

### Prioritizing Random Failures

When a test in your codebase shows flakiness, immediately prioritize addressing it. If you cannot resolve the flakiness immediately, you can mark it as flaky:

```ruby
it_is_flaky "returns the correct date", "https://runway.powerapp.cloud/backlog_items/FLAKY-1111" do
  expect(Date.current).to eql "2024-12-11"
end
```

### Common Causes of Flakiness

#### Test Order Dependencies

One tell-tale sign that specs are order-dependent is if your configuration is set to run tests in a defined order.

**❌ Bad:**

```ruby
config.order = :defined
```

**✅ Good:**

```ruby
# Specs should run in random order
config.order = :rand
```

#### Time-Dependent Tests

Tests that rely on the current time can be flaky if the timing is off slightly. Use Rails' built-in TimeHelpers, not Timecop.

**✅ Good:**

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

#### Asynchronous Operations

It is not recommended to test async side effects in tests. Background jobs can be tested in isolation, and you can always test that the jobs get called.

**❌ Bad:**

```ruby
RSpec.describe User do
  it "syncs timezone to territory" do
    user = create(:user, territory: phoenix)
    expect(user.time_zone).to eq("Arizona")  # Testing async side effect
  end
end
```

**✅ Good:**

```ruby
RSpec.describe User do
  it "enqueues SyncTimeZoneToTerritoryJob" do
    user = create(:user)
    expect { user.update(territory: phoenix) }
      .to have_enqueued_job(SyncTimeZoneToTerritoryJob)
      .with(user.id)
  end
end

RSpec.describe SyncTimeZoneToTerritoryJob do
  it "updates user's time_zone" do
    # Test the job in isolation
  end
end
```

#### External Services

Use WebMock to stub external HTTP requests. Never call external dependencies in tests.

```ruby
# /spec/services/weather_service_spec.rb
RSpec.describe WeatherService, type: :model do
  it "fetches the weather" do
    stub_request(:get, "http://api.weather.com")
      .to_return(status: 200, body: '{"weather": "sunny"}')

    expect(WeatherService.new.fetch).to eq("sunny")
  end
end
```

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
2. Identify a test that is over-mocking. How could you use a real object instead? Remember: never mock the subject of your test.
3. Find a slow spec and make it faster. What tools or techniques did you use? Did you use `build_stubbed`, better factories, or avoid unnecessary database hits?
4. Write a spec that tests behavior, not implementation. How would you rewrite a test that checks private state?
5. Review your spec file and rename any vague `it` blocks to be more descriptive. Keep descriptions under 40 characters.
6. Practice using contexts with 'when', 'with', or 'without' to organize your tests clearly.
7. Add edge case testing to ensure you're testing all possible scenarios, not just the happy path.
8. Configure your RSpec to use expect syntax only and run tests in random order.
9. Why is it important to keep your test suite fast and reliable? How does it affect your team's productivity?

Reflect: What could go wrong if your test suite is slow, flaky, or hard to understand? How would that impact your team, your users, and your ability to ship features?

---

## Resources

- [Better Specs for Nitro](https://portal.powerapp.cloud/docs/default/component/bt-handbook/technical-standards/better-specs-for-nitro/)
- [Betterspecs](https://www.betterspecs.org/)
- [Test Smells](http://xunitpatterns.com/Test%20Smells.html)
- [FactoryBot cheat sheet](https://devhints.io/factory_bot)
- [Effective Testing in RSpec 3](https://pragprog.com/titles/rspec3/effective-testing-with-rspec-3/)
- [Refactoring Rails](https://www.refactoringrails.io/)
- [What is DRY development?](https://www.digitalocean.com/community/tutorials/what-is-dry-development)
- [RSpec: Shared Examples](https://relishapp.com/rspec/rspec-core/v/3-10/docs/example-groups/shared-examples)
- [RSpec: Custom Matchers](https://relishapp.com/rspec/rspec-expectations/v/3-10/docs/custom-matchers/define-matcher)
- [Rails Guides: Testing Rails Applications](https://guides.rubyonrails.org/testing.html)
