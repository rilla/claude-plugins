# Form Objects & Query Objects

## When to Use Form Objects

Use form objects when:
- Operation spans multiple models
- Complex validations beyond single model
- Need transaction boundaries
- Decoupling form from model structure

**Don't use for:**
- Simple single-model CRUD
- Basic validations (use model validations)

## Form Object Pattern

### Base Class

```ruby
# app/forms/application_form.rb
class ApplicationForm
  include ActiveModel::Model
  include ActiveModel::Attributes

  def save
    return false unless valid?

    ApplicationRecord.transaction do
      submit!
    end

    true
  rescue ActiveRecord::RecordInvalid => e
    errors.merge!(e.record.errors)
    false
  end

  private

  def submit!
    raise NotImplementedError
  end
end
```

### Registration Form Example

```ruby
# app/forms/participant_registration_form.rb
class ParticipantRegistrationForm < ApplicationForm
  attribute :full_name, :string
  attribute :email, :string
  attribute :password, :string
  attribute :terms_accepted, :boolean

  validates :full_name, presence: true, length: { minimum: 2 }
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :password, presence: true, length: { minimum: 8 }
  validates :terms_accepted, acceptance: true

  attr_reader :participant

  private

  def submit!
    @participant = Participant.create!(
      name: full_name,
      email: email,
      password: password
    )

    Profile.create!(
      participant: @participant,
      created_via: :registration
    )

    WelcomeMailer.registration(@participant).deliver_later
  end
end
```

### Controller Usage

```ruby
class RegistrationsController < ApplicationController
  def new
    @form = ParticipantRegistrationForm.new
  end

  def create
    @form = ParticipantRegistrationForm.new(registration_params)

    if @form.save
      session[:participant_id] = @form.participant.id
      redirect_to dashboard_path
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def registration_params
    params.require(:registration).permit(:full_name, :email, :password, :terms_accepted)
  end
end
```

### View Usage

```erb
<%= form_with model: @form, url: registrations_path do |f| %>
  <div class="field">
    <%= f.label :full_name %>
    <%= f.text_field :full_name %>
    <%= f.object.errors[:full_name].first %>
  </div>

  <div class="field">
    <%= f.label :email %>
    <%= f.email_field :email %>
  </div>

  <div class="field">
    <%= f.label :password %>
    <%= f.password_field :password %>
  </div>

  <div class="field">
    <%= f.check_box :terms_accepted %>
    <%= f.label :terms_accepted, "I accept the terms" %>
  </div>

  <%= f.submit "Register" %>
<% end %>
```

### Multi-Step Form

```ruby
# app/forms/cloud_creation_form.rb
class CloudCreationForm < ApplicationForm
  # Step 1: Basic info
  attribute :title, :string
  attribute :description, :string

  # Step 2: Upload
  attribute :source_image

  # Step 3: Settings
  attribute :style, :string
  attribute :visibility, :string

  attribute :current_step, :integer, default: 1

  validates :title, presence: true, if: -> { current_step >= 1 }
  validates :source_image, presence: true, if: -> { current_step >= 2 }
  validates :style, presence: true, if: -> { current_step >= 3 }

  def next_step
    self.current_step += 1 if valid?
  end

  def previous_step
    self.current_step -= 1 if current_step > 1
  end

  def final_step?
    current_step == 3
  end

  private

  def submit!
    Cloud.create!(
      title: title,
      description: description,
      source_image: source_image,
      style: style,
      visibility: visibility
    )
  end
end
```

## Query Object Pattern

Use query objects for:
- Complex queries with multiple conditions
- Reusable query logic
- Queries that need testing in isolation
- Queries with dynamic filters

### Base Class

```ruby
# app/queries/application_query.rb
class ApplicationQuery
  def initialize(relation = default_relation)
    @relation = relation
  end

  def call
    raise NotImplementedError
  end

  private

  attr_reader :relation

  def default_relation
    raise NotImplementedError
  end
end
```

### Basic Query Object

```ruby
# app/queries/participant/pending_query.rb
class Participant::PendingQuery < ApplicationQuery
  def call
    relation
      .without_picked_cloud
      .where(blocked: false)
      .order(created_at: :asc)
  end

  private

  def default_relation
    Participant.all
  end
end

# Usage:
Participant::PendingQuery.new.call
Participant::PendingQuery.new(Participant.active).call
```

### Query with Parameters

```ruby
# app/queries/cloud/search_query.rb
class Cloud::SearchQuery < ApplicationQuery
  def initialize(relation = Cloud.all, params = {})
    super(relation)
    @params = params
  end

  def call
    result = relation
    result = filter_by_state(result)
    result = filter_by_participant(result)
    result = filter_by_date_range(result)
    result = search_by_title(result)
    result = apply_sorting(result)
    result
  end

  private

  attr_reader :params

  def default_relation
    Cloud.all
  end

  def filter_by_state(rel)
    return rel if params[:state].blank?

    rel.where(state: params[:state])
  end

  def filter_by_participant(rel)
    return rel if params[:participant_id].blank?

    rel.where(participant_id: params[:participant_id])
  end

  def filter_by_date_range(rel)
    return rel if params[:start_date].blank? && params[:end_date].blank?

    rel = rel.where("created_at >= ?", params[:start_date]) if params[:start_date]
    rel = rel.where("created_at <= ?", params[:end_date]) if params[:end_date]
    rel
  end

  def search_by_title(rel)
    return rel if params[:q].blank?

    rel.where("title ILIKE ?", "%#{params[:q]}%")
  end

  def apply_sorting(rel)
    case params[:sort]
    when "oldest" then rel.order(created_at: :asc)
    when "title" then rel.order(title: :asc)
    else rel.order(created_at: :desc)
    end
  end
end

# Usage:
Cloud::SearchQuery.new(Cloud.all, state: "generated", q: "sunset").call
```

### Controller Usage

```ruby
class CloudsController < ApplicationController
  def index
    @clouds = Cloud::SearchQuery.new(
      current_participant.clouds,
      search_params
    ).call.page(params[:page])
  end

  private

  def search_params
    params.permit(:state, :q, :sort, :start_date, :end_date)
  end
end
```

### Composable Queries

```ruby
# Chain multiple query objects
class Dashboard::StatsQuery
  def initialize(participant)
    @participant = participant
  end

  def call
    {
      pending_clouds: pending_clouds_query.call.count,
      completed_clouds: completed_clouds_query.call.count,
      recent_activity: recent_activity_query.call
    }
  end

  private

  def pending_clouds_query
    Cloud::ByStateQuery.new(base_relation, state: :pending)
  end

  def completed_clouds_query
    Cloud::ByStateQuery.new(base_relation, state: :generated)
  end

  def recent_activity_query
    Cloud::RecentQuery.new(base_relation, limit: 5)
  end

  def base_relation
    @participant.clouds
  end
end
```

## Testing

### Form Object Tests

```ruby
# spec/forms/participant_registration_form_spec.rb
RSpec.describe ParticipantRegistrationForm do
  describe "validations" do
    it "requires full_name" do
      form = described_class.new(full_name: "")
      expect(form).not_to be_valid
      expect(form.errors[:full_name]).to include("can't be blank")
    end

    it "requires valid email" do
      form = described_class.new(email: "invalid")
      expect(form).not_to be_valid
    end
  end

  describe "#save" do
    let(:valid_params) do
      {
        full_name: "John Doe",
        email: "john@example.com",
        password: "password123",
        terms_accepted: true
      }
    end

    it "creates participant and profile" do
      form = described_class.new(valid_params)

      expect { form.save }.to change(Participant, :count).by(1)
        .and change(Profile, :count).by(1)
    end

    it "returns participant" do
      form = described_class.new(valid_params)
      form.save

      expect(form.participant).to be_persisted
    end
  end
end
```

### Query Object Tests

```ruby
# spec/queries/cloud/search_query_spec.rb
RSpec.describe Cloud::SearchQuery do
  let!(:cloud1) { create(:cloud, title: "Sunset Beach", state: :generated) }
  let!(:cloud2) { create(:cloud, title: "Mountain View", state: :pending) }
  let!(:cloud3) { create(:cloud, title: "Sunset Hills", state: :generated) }

  describe "#call" do
    it "filters by state" do
      result = described_class.new(Cloud.all, state: "generated").call

      expect(result).to include(cloud1, cloud3)
      expect(result).not_to include(cloud2)
    end

    it "searches by title" do
      result = described_class.new(Cloud.all, q: "Sunset").call

      expect(result).to include(cloud1, cloud3)
      expect(result).not_to include(cloud2)
    end

    it "combines filters" do
      result = described_class.new(Cloud.all, state: "generated", q: "Beach").call

      expect(result).to eq([cloud1])
    end
  end
end
```

## Comparison: Scopes vs Query Objects

### Use Scopes When

- Simple, single-condition filters
- Frequently chained with other scopes
- Part of model's core identity

```ruby
class Cloud < ApplicationRecord
  scope :recent, -> { order(created_at: :desc) }
  scope :by_state, ->(state) { where(state: state) }
  scope :visible, -> { where(visibility: :public) }
end
```

### Use Query Objects When

- Multiple interacting conditions
- Complex business logic
- Dynamic filter combinations
- Need isolated testing

```ruby
Cloud::SearchQuery.new(relation, params).call
```
