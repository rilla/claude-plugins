# Model Patterns

## Naming Conventions

Use domain language for model names:

| Generic | Domain-Specific |
|---------|-----------------|
| User | Participant, Member, Author |
| Image | Cloud, Artwork, Asset |
| Post | Article, Story, Update |
| Comment | Response, Feedback, Note |

Choose names that reflect what the entity represents in the business domain, not technical implementation.

## Organization Order

Organize model code in this exact order:

```ruby
class Cloud < ApplicationRecord
  # 1. Gems/DSL extensions
  include Turbo::Broadcastable
  has_paper_trail

  # 2. Associations
  belongs_to :participant, counter_cache: true
  has_many :cards, dependent: :destroy
  has_one_attached :source_image

  # 3. Enums (for state machines)
  enum :state, %w[uploaded analyzing analyzed generating generated failed].index_by(&:itself)

  # 4. Normalizations (Rails 7.1+)
  normalizes :title, with: ->(title) { title.strip.titleize }
  normalizes :email, with: ->(email) { email.strip.downcase }

  # 5. Validations
  validates :title, presence: true, length: { maximum: 100 }
  validates :state, presence: true

  # 6. Scopes
  scope :recent, -> { order(created_at: :desc) }
  scope :completed, -> { where(state: %w[analyzed generated]) }
  scope :pending, -> { where(state: %w[uploaded analyzing generating]) }

  # 7. Callbacks
  after_create_commit :schedule_processing

  # 8. Delegations
  delegate :email, to: :participant, prefix: true

  # 9. Public methods
  def display_name
    title.presence || "Cloud ##{id}"
  end

  def processing?
    %w[analyzing generating].include?(state)
  end

  # 10. Private methods
  private

  def schedule_processing
    CloudProcessingJob.perform_later(self)
  end
end
```

## Enums for State

Always use enums for state management instead of state machine gems:

```ruby
# Define enum with explicit mapping
enum :state, %w[draft pending active archived].index_by(&:itself)

# PostgreSQL enum type (recommended for large tables)
# In migration:
create_enum :cloud_state, %w[uploaded analyzing analyzed generating generated failed]

add_column :clouds, :state, :enum, enum_type: :cloud_state, default: "uploaded", null: false

# Query methods are auto-generated:
cloud.uploaded?     # => true/false
cloud.analyzing!    # transitions state
Cloud.uploaded      # => scope
```

## Associations

### Counter Caches

Every `has_many` association should include counter cache:

```ruby
# app/models/participant.rb
class Participant < ApplicationRecord
  has_many :clouds, dependent: :destroy
end

# app/models/cloud.rb
class Cloud < ApplicationRecord
  belongs_to :participant, counter_cache: true
end

# Migration:
add_column :participants, :clouds_count, :integer, default: 0, null: false

# Reset counters if needed:
Participant.reset_counters(participant.id, :clouds)
```

### Dependent Options

Choose appropriate dependent option:

```ruby
has_many :clouds, dependent: :destroy      # destroy children, run callbacks
has_many :logs, dependent: :delete_all     # fast delete, skip callbacks
has_many :comments, dependent: :nullify    # keep records, remove FK
has_one :profile, dependent: :destroy
```

## Normalizations (Rails 7.1+)

Use `normalizes` for data cleaning:

```ruby
class Participant < ApplicationRecord
  normalizes :email, with: ->(email) { email.strip.downcase }
  normalizes :phone, with: ->(phone) { phone.gsub(/\D/, "") }
  normalizes :name, with: ->(name) { name.strip.squeeze(" ") }
end

# Applied automatically on assignment and query:
participant.email = "  JOHN@EXAMPLE.COM  "
participant.email  # => "john@example.com"

Participant.find_by(email: "JOHN@example.com")  # finds the record
```

## Scopes

Keep scopes simple and composable:

```ruby
class Cloud < ApplicationRecord
  # Simple conditions
  scope :recent, -> { order(created_at: :desc) }
  scope :active, -> { where(archived_at: nil) }
  scope :by_participant, ->(participant) { where(participant: participant) }

  # Composable
  scope :recent_active, -> { active.recent }

  # With joins (use sparingly)
  scope :with_participant, -> { includes(:participant) }
end

# Usage:
Cloud.active.recent.limit(10)
Cloud.by_participant(current_participant).recent
```

For complex queries, use Query Objects instead (see `forms-queries.md`).

## Validations

```ruby
class Cloud < ApplicationRecord
  # Presence
  validates :title, presence: true

  # Length
  validates :description, length: { maximum: 1000 }

  # Uniqueness (with scope)
  validates :slug, uniqueness: { scope: :participant_id }

  # Format
  validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }

  # Custom
  validate :source_image_attached

  private

  def source_image_attached
    errors.add(:source_image, "must be attached") unless source_image.attached?
  end
end
```

## Extracting Complex Logic

### When to Extract

Extract to namespaced class when:
- Method exceeds 15 lines
- Logic calls external APIs
- Logic involves complex algorithms
- Logic is reusable across contexts

### Namespaced Model Classes

```ruby
# app/models/cloud/card_generator.rb
class Cloud::CardGenerator
  def initialize(cloud, options = {})
    @cloud = cloud
    @style = options.fetch(:style, :default)
  end

  def call
    validate_source!
    image = process_image
    card = create_card(image)
    notify_completion(card)
    card
  end

  private

  attr_reader :cloud, :style

  def validate_source!
    raise ArgumentError, "Source image required" unless cloud.source_image.attached?
  end

  def process_image
    # Complex image processing
  end

  def create_card(image)
    cloud.cards.create!(
      image: image,
      style: style
    )
  end

  def notify_completion(card)
    CloudMailer.card_ready(card).deliver_later
  end
end

# Usage:
Cloud::CardGenerator.new(cloud, style: :vintage).call
```

### Keep Models Focused

```ruby
# Good - model delegates to specialized class
class Cloud < ApplicationRecord
  def generate_card(style: :default)
    Cloud::CardGenerator.new(self, style: style).call
  end
end

# Bad - too much logic in model
class Cloud < ApplicationRecord
  def generate_card(style: :default)
    validate_source!
    image = ImageProcessor.new(source_image).process
    card = cards.create!(...)
    # 50 more lines...
  end
end
```

## Callbacks

Use callbacks sparingly:

```ruby
class Cloud < ApplicationRecord
  # Good - scheduling async work
  after_create_commit :schedule_processing

  # Good - cleanup
  after_destroy :cleanup_external_resources

  # Bad - business logic in callback
  # after_save :recalculate_participant_stats  # Move to job

  private

  def schedule_processing
    CloudProcessingJob.perform_later(self)
  end
end
```

**Avoid:**
- Complex logic in callbacks
- Callbacks that call external services synchronously
- Deeply nested callback chains
- Callbacks that modify other records (use jobs instead)
