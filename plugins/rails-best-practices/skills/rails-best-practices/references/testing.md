# Testing Patterns (RSpec)

## Test Organization

```
spec/
├── models/              # Unit tests for models
├── requests/            # Integration tests for controllers
├── system/              # End-to-end browser tests
├── forms/               # Form object tests
├── queries/             # Query object tests
├── jobs/                # Background job tests
├── mailers/             # Mailer tests
├── components/          # ViewComponent tests
├── factories/           # FactoryBot definitions
└── support/             # Helpers and configuration
```

## What to Test

### Models

Test business logic, validations, and associations:

```ruby
# spec/models/cloud_spec.rb
RSpec.describe Cloud do
  describe "associations" do
    it { is_expected.to belong_to(:participant).counter_cache(true) }
    it { is_expected.to have_many(:cards).dependent(:destroy) }
  end

  describe "validations" do
    it { is_expected.to validate_presence_of(:title) }
    it { is_expected.to validate_length_of(:title).is_at_most(100) }
  end

  describe "scopes" do
    describe ".recent" do
      it "orders by created_at desc" do
        old = create(:cloud, created_at: 1.day.ago)
        new = create(:cloud, created_at: 1.hour.ago)

        expect(Cloud.recent).to eq([new, old])
      end
    end
  end

  describe "#display_name" do
    it "returns title when present" do
      cloud = build(:cloud, title: "My Cloud")
      expect(cloud.display_name).to eq("My Cloud")
    end

    it "returns fallback when title blank" do
      cloud = create(:cloud, title: "")
      expect(cloud.display_name).to eq("Cloud ##{cloud.id}")
    end
  end

  describe "state transitions" do
    it "can transition from uploaded to analyzing" do
      cloud = create(:cloud, state: :uploaded)
      expect { cloud.analyzing! }.to change(cloud, :state).to("analyzing")
    end
  end
end
```

### Request Specs

Test HTTP endpoints, not controller internals:

```ruby
# spec/requests/clouds_spec.rb
RSpec.describe "Clouds" do
  describe "GET /clouds" do
    it "returns clouds list" do
      create_list(:cloud, 3, state: :generated)

      get clouds_path

      expect(response).to have_http_status(:ok)
      expect(response.body).to include("Cloud")
    end
  end

  describe "GET /clouds/:id" do
    it "returns cloud details" do
      cloud = create(:cloud, title: "Test Cloud")

      get cloud_path(cloud)

      expect(response).to have_http_status(:ok)
      expect(response.body).to include("Test Cloud")
    end

    it "returns 404 for missing cloud" do
      get cloud_path(id: "nonexistent")

      expect(response).to have_http_status(:not_found)
    end
  end
end

# spec/requests/participant/clouds_spec.rb
RSpec.describe "Participant Clouds" do
  let(:participant) { create(:participant) }

  describe "POST /participant/clouds" do
    context "when authenticated" do
      before { sign_in(participant) }

      it "creates a cloud" do
        expect {
          post participant_clouds_path, params: { cloud: { title: "New Cloud" } }
        }.to change(Cloud, :count).by(1)

        expect(response).to redirect_to(participant_cloud_path(Cloud.last))
      end

      it "enqueues processing job" do
        post participant_clouds_path, params: { cloud: { title: "New Cloud" } }

        expect(CloudProcessingJob).to have_been_enqueued
      end

      it "rejects invalid params" do
        post participant_clouds_path, params: { cloud: { title: "" } }

        expect(response).to have_http_status(:unprocessable_entity)
      end
    end

    context "when not authenticated" do
      it "redirects to login" do
        post participant_clouds_path, params: { cloud: { title: "New Cloud" } }

        expect(response).to redirect_to(login_path)
      end
    end
  end
end
```

### System Specs

Test critical user flows (excluded from default run):

```ruby
# spec/system/cloud_creation_spec.rb
RSpec.describe "Cloud Creation", type: :system do
  let(:participant) { create(:participant) }

  before { sign_in(participant) }

  it "allows creating a cloud with all steps" do
    visit new_participant_cloud_path

    # Step 1: Basic info
    fill_in "Title", with: "My Beautiful Sunset"
    fill_in "Description", with: "A sunset at the beach"
    click_button "Next"

    # Step 2: Upload
    attach_file "Source image", Rails.root.join("spec/fixtures/sunset.jpg")
    click_button "Next"

    # Step 3: Settings
    select "Vintage", from: "Style"
    click_button "Create Cloud"

    expect(page).to have_content("Cloud created successfully")
    expect(page).to have_content("My Beautiful Sunset")
  end
end
```

### Job Specs

```ruby
# spec/jobs/cloud_processing_job_spec.rb
RSpec.describe CloudProcessingJob do
  let(:cloud) { create(:cloud, state: :uploaded) }

  describe "#perform" do
    it "processes cloud through all stages" do
      described_class.perform_now(cloud)

      expect(cloud.reload).to be_generated
    end

    it "handles failures gracefully" do
      allow(Cloud::Analyzer).to receive(:new).and_raise(StandardError, "API error")

      described_class.perform_now(cloud)

      expect(cloud.reload).to be_failed
      expect(cloud.processing_error).to include("API error")
    end

    it "skips already processed clouds" do
      cloud.update!(state: :generated)

      expect(Cloud::Analyzer).not_to receive(:new)
      described_class.perform_now(cloud)
    end
  end
end
```

## FactoryBot Patterns

### Basic Factory

```ruby
# spec/factories/clouds.rb
FactoryBot.define do
  factory :cloud do
    participant
    title { Faker::Lorem.sentence(word_count: 3) }
    state { :uploaded }

    trait :with_image do
      after(:build) do |cloud|
        cloud.source_image.attach(
          io: File.open(Rails.root.join("spec/fixtures/test.jpg")),
          filename: "test.jpg",
          content_type: "image/jpeg"
        )
      end
    end

    trait :generated do
      state { :generated }
      with_image
    end

    trait :failed do
      state { :failed }
      processing_error { "Something went wrong" }
    end
  end
end
```

### Factory with Associations

```ruby
# spec/factories/participants.rb
FactoryBot.define do
  factory :participant do
    sequence(:email) { |n| "participant#{n}@example.com" }
    name { Faker::Name.name }
    password { "password123" }

    trait :with_clouds do
      transient do
        clouds_count { 3 }
      end

      after(:create) do |participant, evaluator|
        create_list(:cloud, evaluator.clouds_count, participant: participant)
      end
    end

    trait :premium do
      plan { :premium }
      premium_until { 1.year.from_now }
    end
  end
end
```

### Usage

```ruby
# Basic
cloud = create(:cloud)
cloud = build(:cloud)  # Not persisted

# With traits
cloud = create(:cloud, :generated)
cloud = create(:cloud, :with_image, state: :analyzing)

# With overrides
cloud = create(:cloud, title: "Custom Title", participant: my_participant)

# Lists
clouds = create_list(:cloud, 5)
clouds = create_list(:cloud, 3, :generated)
```

## Test Helpers

```ruby
# spec/support/authentication_helper.rb
module AuthenticationHelper
  def sign_in(participant)
    post login_path, params: {
      email: participant.email,
      password: "password123"
    }
  end

  def auth_headers(participant)
    { "Authorization" => "Bearer #{participant.access_token}" }
  end
end

RSpec.configure do |config|
  config.include AuthenticationHelper, type: :request
  config.include AuthenticationHelper, type: :system
end
```

## What NOT to Test

### Framework Behavior

```ruby
# Don't test Rails internals
it "has a title attribute" do  # Unnecessary
  cloud = Cloud.new(title: "Test")
  expect(cloud.title).to eq("Test")
end

# Don't test simple delegations
it "delegates email to participant" do  # If using delegate
  expect(cloud.participant_email).to eq(cloud.participant.email)
end
```

### Obvious Code

```ruby
# Don't test trivial methods
def full_name
  "#{first_name} #{last_name}"
end

# Unless there's edge case handling
def full_name
  [first_name, middle_name, last_name].compact.join(" ")
end
# ^ This might deserve a test for nil handling
```

### Generated Code

Don't test:
- Default scopes that just wrap `where`
- Simple validation presence (use shoulda-matchers)
- ActiveRecord callbacks that just call methods

## RSpec Configuration

```ruby
# spec/rails_helper.rb
RSpec.configure do |config|
  # Use transactional fixtures
  config.use_transactional_fixtures = true

  # Filter system specs by default
  config.filter_run_excluding type: :system

  # Include URL helpers
  config.include Rails.application.routes.url_helpers

  # Database cleaner for system specs
  config.before(:each, type: :system) do
    driven_by :selenium_chrome_headless
  end
end

# Run all tests
# bundle exec rspec

# Run only fast tests (exclude system)
# bundle exec rspec --tag ~type:system

# Run system tests
# bundle exec rspec --tag type:system
```

## Test Speed Tips

1. **Use `build` over `create`** when persistence not needed
2. **Use `let` over `before`** for lazy evaluation
3. **Avoid unnecessary associations** in factories
4. **Use `create_list` sparingly** - often 2-3 records enough
5. **Mock external services** in unit tests
6. **Run system tests separately** from unit tests
