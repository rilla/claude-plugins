---
name: Rails Best Practices
description: Use when generating Rails code, creating models/controllers, discussing Rails conventions, refactoring Rails apps, or working with ActiveRecord, Hotwire, RSpec, or Ruby on Rails development. Combines Evil Martians' conventions with 37signals' One Person Framework philosophy.
version: 1.0.0
---

# Rails Best Practices

Comprehensive standards for AI-assisted Rails development, combining Evil Martians' proven conventions with 37signals' "One Person Framework" philosophy. The goal: write Rails code so clear and simple that one developer can understand the entire system.

## Core Philosophy

> "Vanilla Rails is plenty." — Jorge Manrubia, 37signals

**Guiding Principles:**

1. **Conceptual Compression** — Hide complexity behind simple, natural APIs
2. **Rich Models, Thin Controllers** — Business logic in models, HTTP handling in controllers
3. **Model Methods First** — Prefer `card.close` over service objects or form objects
4. **Follow Rails Conventions** — Leverage the framework, don't fight it
5. **Domain Language** — Name models after business concepts (Participant, Cloud, not User, Image)
6. **Readable Code** — Self-documenting without comments

### The "One Person" Test

Can one developer understand this codebase in an afternoon? If not, simplify.

## Quick Decision Tree

**Where does this logic belong?**

```
Single model operation?
  → Model method (e.g., card.close)

Multi-model transaction with simple coordination?
  → Model method with transaction

Multi-model with complex validation/conditional logic?
  → Form Object (escape hatch)

External API call?
  → Namespaced model class (e.g., Cloud::CardGenerator)

Async work?
  → Job (orchestrates, doesn't execute)

Complex reusable query?
  → Query object or scope

Authorization?
  → ActionPolicy

Configuration?
  → Anyway Config
```

## File Structure

```
app/
├── models/
│   ├── card.rb
│   ├── card/closeable.rb           # Concerns with "has trait" semantics
│   └── cloud/card_generator.rb     # Namespaced classes for complex logic
├── controllers/
│   ├── cards_controller.rb
│   └── concerns/
│       └── card_scoped.rb          # Shared controller behavior
├── forms/                          # ONLY for genuinely complex multi-model operations
│   └── application_form.rb
├── queries/                        # ONLY for complex, reusable queries
│   └── cloud/search_query.rb
├── policies/
│   └── card_policy.rb              # ActionPolicy
├── jobs/
│   └── cloud_processing_job.rb
├── views/
│   ├── cards/
│   └── components/                 # ViewComponent
├── frontend/                       # Vite Rails
│   ├── controllers/                # Stimulus
│   └── stylesheets/
└── config/
    └── configs/                    # Anyway Config
```

**Critical:** No `app/services/`, `app/contexts/`, or `app/operations/` directories.

## Model Organization

**Standard order within model files:**

1. Gems/DSL extensions
2. Associations (with `counter_cache: true` when appropriate)
3. Enums (for workflow states)
4. Normalizations (Rails 7.1+)
5. Validations
6. Scopes
7. Callbacks
8. Delegations
9. Public methods
10. Private methods

**Example:**

```ruby
class Card < ApplicationRecord
  include Closeable     # Concern with "has trait" semantics
  include Watchable

  # Associations
  belongs_to :board
  has_many :comments, dependent: :destroy, counter_cache: true
  has_many :watchers, dependent: :destroy

  # Enums for workflow states
  enum :priority, %w[low medium high urgent].index_by(&:itself)

  # Validations
  validates :title, presence: true, length: { maximum: 200 }
  validates :priority, presence: true

  # Scopes
  scope :recent, -> { order(created_at: :desc) }
  scope :high_priority, -> { where(priority: %w[high urgent]) }

  # Public methods
  def duplicate
    dup.tap do |duplicate|
      duplicate.title = "Copy of #{title}"
      duplicate.save!
    end
  end
end
```

## State Management: Decision Tree

Choose between **enums** and **state records** based on context:

### Use Enums When:
- Simple workflow states (draft → published → archived)
- No need to track WHO changed it or WHY
- State transitions are straightforward
- PostgreSQL enum is sufficient

```ruby
class Article < ApplicationRecord
  enum :status, %w[draft published archived].index_by(&:itself)

  def publish!
    update!(status: :published, published_at: Time.current)
  end
end
```

### Use State Records When:
- Need to track WHO made the change
- Need to track WHEN it happened (beyond updated_at)
- Need to track WHY (reason, notes)
- State has associated metadata
- Reversible actions (delete record to undo)

```ruby
class Card < ApplicationRecord
  has_one :closure, dependent: :destroy
  has_one :confirmation, dependent: :destroy

  def closed?
    closure.present?
  end

  def close(reason: nil)
    transaction do
      create_closure!(user: Current.user, reason: reason)
      events.create!(action: :closed, creator: Current.user)
    end
    notify_watchers_of_closure
  end

  def reopen
    closure&.destroy
    events.create!(action: :reopened, creator: Current.user)
  end
end

# app/models/closure.rb
class Closure < ApplicationRecord
  belongs_to :card, touch: true
  belongs_to :user, default: -> { Current.user }

  validates :reason, length: { maximum: 500 }
end
```

**See `references/state-management.md` for detailed decision guide.**

## Controllers: Keep Them Thin

**Target:** 5-10 lines per action. No business logic.

```ruby
class CardsController < ApplicationController
  before_action :set_card, only: %i[show edit update destroy]

  def create
    @card = current_board.cards.create!(card_params)
    CardProcessingJob.perform_later(@card) if @card.requires_processing?

    redirect_to @card, notice: t(".created")
  end

  def update
    @card.update!(card_params)
    redirect_to @card, notice: t(".updated")
  end

  private
    def set_card
      @card = current_board.cards.find(params[:id])
    end

    def card_params
      params.require(:card).permit(:title, :description, :priority)
    end
end
```

**Guard clauses for authorization:**

```ruby
def create
  return head :forbidden unless current_user.can_create_cards?

  @card = current_board.cards.create!(card_params)
  redirect_to @card
end
```

## Concerns: "Has Trait" Semantics

Concerns must represent a clear trait or behavior. Self-contained with associations, scopes, callbacks, and methods.

**Good concern names (adjectives with -able/-ible):**
- `Closeable` — can be closed
- `Publishable` — can be published
- `Watchable` — can be watched
- `Confirmable` — can be confirmed
- `Schedulable` — can be scheduled

**Example:**

```ruby
# app/models/concerns/card/closeable.rb
module Card::Closeable
  extend ActiveSupport::Concern

  included do
    has_one :closure, dependent: :destroy

    scope :closed, -> { joins(:closure) }
    scope :open, -> { where.missing(:closure) }
  end

  def close(reason: nil)
    transaction do
      create_closure!(user: Current.user, reason: reason, closed_at: Time.current)
      events.create!(action: :closed, creator: Current.user)
    end
    notify_watchers_of_closure
  end

  def reopen
    closure&.destroy
    events.create!(action: :reopened, creator: Current.user)
  end

  def closed?
    closure.present?
  end

  def open?
    !closed?
  end

  private
    def notify_watchers_of_closure
      watchers.each { |w| CardNotificationJob.perform_later(w, self, :closed) }
    end
end
```

**What concerns are NOT:**
- Arbitrary containers to split large models
- A replacement for proper extraction to separate classes
- An excuse to avoid creating new models when complexity warrants it

## Naming Conventions

### Verb Methods for Actions

```ruby
# ✅ GOOD: Natural verbs
card.close
card.reopen
card.gild
card.postpone
booking.confirm
booking.cancel

# ❌ BAD: Procedural/setter style
card.set_closed(true)
card.update_status(:closed)
CardCloser.call(card)
```

### Predicate Methods for State

```ruby
card.closed?
card.open?
card.postponed?
booking.confirmed?

# Derived from presence
def closed?
  closure.present?
end
```

### Scope Naming (Adverbs/Adjectives)

```ruby
scope :chronologically,         -> { order(created_at: :asc) }
scope :reverse_chronologically, -> { order(created_at: :desc) }
scope :alphabetically,          -> { order(name: :asc) }
scope :active,                  -> { where(active: true) }
scope :upcoming,                -> { where(starts_at: Time.current..) }
scope :today,                   -> { where(starts_at: Time.current.all_day) }
```

## When to Extract Logic

### Model Methods → Namespaced Classes

Extract when:
- Method > 15 lines
- Calls external API
- Complex algorithm that would clutter model

```ruby
# ✅ GOOD: Extracted to namespaced class
class Cloud::CardGenerator
  def initialize(cloud, style: :default)
    @cloud = cloud
    @style = style
  end

  def call
    response = HTTParty.post(api_endpoint, body: payload)
    @cloud.update!(card_url: response["url"])
  end

  private
    attr_reader :cloud, :style

    def api_endpoint
      ENV.fetch("CARD_API_URL")
    end

    def payload
      { image: cloud.image_url, style: style }
    end
end

# Usage in model
class Cloud < ApplicationRecord
  def generate_card(style: :default)
    Cloud::CardGenerator.new(self, style: style).call
  end
end
```

### Model Methods → Form Objects

**Only extract to Form Object when:**
- Coordinating 2+ models with complex validation
- Conditional logic based on user input
- Operation has form-specific validation rules

```ruby
# app/forms/booking_with_new_client_form.rb
class BookingWithNewClientForm < ApplicationForm
  attribute :account
  attribute :service_id, :integer
  attribute :date, :date
  attribute :time, :time

  # New client attributes
  attribute :client_name, :string
  attribute :client_email, :string
  attribute :client_phone, :string

  validates :service_id, :date, :time, presence: true
  validates :client_name, :client_email, :client_phone, presence: true
  validate :email_not_already_registered

  def save
    return false unless valid?

    transaction do
      client = create_client!
      booking = create_booking!(client)
      send_notifications(booking)
      @booking = booking
    end
    true
  end

  attr_reader :booking

  private
    def create_client!
      account.clients.create!(
        name: client_name,
        email: client_email,
        phone: client_phone
      )
    end

    def create_booking!(client)
      account.bookings.create!(
        client: client,
        service_id: service_id,
        date: date,
        time: time
      )
    end

    def send_notifications(booking)
      BookingConfirmationJob.perform_later(booking)
    end

    def email_not_already_registered
      if account.clients.exists?(email: client_email)
        errors.add(:client_email, :taken)
      end
    end
end
```

**Default to model methods. Form Objects are the escape hatch, not the default.**

## Technology Stack

### Required Gems

| Component | Choice | Why |
|-----------|--------|-----|
| **Web Server** | Puma | Default, battle-tested |
| **Assets** | Propshaft + Vite Rails | Modern, fast |
| **Database** | PostgreSQL | Full-featured, reliable |
| **Frontend** | Hotwire (Turbo + Stimulus) | Rich UIs without build complexity |
| **Components** | ViewComponent | Reusable, testable UI |
| **Jobs** | SolidQueue | Database-backed, no Redis |
| **Authorization** | ActionPolicy | Explicit, testable policies |
| **Config** | Anyway Config | Type-safe environment vars |
| **Testing** | RSpec + FactoryBot | Comprehensive, readable tests |
| **Linting** | Standard | Zero-config Ruby style |
| **IDs** | Nanoid + FriendlyID | Short, readable URLs |
| **HTTP** | HTTParty | Simple, clear API calls |

### Forbidden

- ❌ Devise (use `has_secure_password` + sessions)
- ❌ CanCanCan (use ActionPolicy)
- ❌ ActiveAdmin (build admin interfaces in Rails)
- ❌ Service object gems
- ❌ State machine gems (use enums or state records)
- ❌ dry-types, Virtus, Literal (use plain Ruby + ActiveModel)

## Code Quality Patterns

### Time Handling

```ruby
# ✅ GOOD
Time.current                    # Respects Rails timezone
Date.current                    # Respects Rails timezone
booking.starts_at.in_time_zone(account.timezone)

# ❌ BAD
Time.now                        # System timezone
Date.today                      # System timezone
```

### Money Handling

```ruby
# ✅ GOOD: Integer cents
add_column :services, :price_cents, :integer, default: 0, null: false

def price
  price_cents / 100.0
end

def price=(value)
  self.price_cents = (value.to_f * 100).round
end
```

### Query Scoping (Security)

```ruby
# ✅ GOOD: Always scope to tenant
@bookings = current_account.bookings.upcoming
@booking = current_account.bookings.find(params[:id])

# ❌ BAD: Unscoped (security risk!)
@booking = Booking.find(params[:id])
```

### N+1 Prevention

```ruby
# ✅ GOOD: Eager load
@bookings = current_account.bookings
              .includes(:client, :service, :user)
              .upcoming

# Add a preloaded scope for common patterns
scope :preloaded, -> { includes(:creator, :tags, :board) }
```

### I18n

```ruby
# ✅ GOOD: Always use I18n
redirect_to @booking, notice: t(".created")
validates :starts_at, presence: { message: :blank }

# ❌ BAD: Hardcoded strings
redirect_to @booking, notice: "Booking created!"
```

## Testing with RSpec

### File Organization

```
spec/
├── models/
│   ├── card_spec.rb
│   └── concerns/
│       └── closeable_spec.rb
├── requests/
│   └── cards_spec.rb
├── system/
│   └── cards_spec.rb
├── jobs/
│   └── card_processing_job_spec.rb
├── policies/
│   └── card_policy_spec.rb
└── factories/
    ├── cards.rb
    ├── users.rb
    └── closures.rb
```

### Factory Pattern

```ruby
# spec/factories/cards.rb
FactoryBot.define do
  factory :card do
    board
    title { "Card #{sequence(:card_number)}" }
    description { "Description for #{title}" }
    priority { :medium }

    trait :high_priority do
      priority { :high }
    end

    trait :closed do
      after(:create) do |card|
        create(:closure, card: card)
      end
    end

    trait :with_comments do
      after(:create) do |card|
        create_list(:comment, 3, card: card)
      end
    end
  end
end
```

### Time-Sensitive Tests

```ruby
# ✅ GOOD: Freeze time
it "includes today's bookings" do
  booking = create(:booking, date: Date.current)
  travel_to booking.date.to_time

  expect(Booking.today).to include(booking)
end
```

**See `references/testing.md` for comprehensive RSpec patterns.**

## Building Block References

Consult these reference files for detailed patterns:

| Reference | Content |
|-----------|---------|
| `references/stack.md` | Complete gem list, rationale, alternatives |
| `references/models.md` | Model organization, extraction patterns, concerns |
| `references/state-management.md` | **NEW:** Enum vs state record decision tree |
| `references/controllers.md` | Thin controllers, guard clauses, concerns |
| `references/database.md` | Schema design, constraints, indexes, migrations |
| `references/jobs.md` | Background job patterns, orchestration |
| `references/views.md` | Hotwire, ViewComponent, Stimulus |
| `references/testing.md` | RSpec organization, factory patterns, what to test |
| `references/configuration.md` | Anyway Config patterns |
| `references/naming-conventions.md` | **NEW:** Comprehensive naming guide |
| `references/anti-patterns.md` | Common mistakes with solutions |

## Code Generation Checklist

Before generating Rails code, verify:

- [ ] Models named after business domain concepts?
- [ ] Model follows organization order?
- [ ] Logic in model methods, not controller?
- [ ] States use enums OR state records (decision tree applied)?
- [ ] Controllers under 10 lines per action?
- [ ] Complex logic extracted to namespaced classes (if > 15 lines)?
- [ ] Concerns have "has trait" semantics?
- [ ] Form Objects only for genuinely complex multi-model operations?
- [ ] Database normalized with FK constraints?
- [ ] Counter caches on has_many associations?
- [ ] Always scoped to current_account/current_user?
- [ ] `Time.current` not `Time.now`?
- [ ] I18n for all user-facing strings?
- [ ] N+1 queries prevented with `includes`?
- [ ] Tests organized by type and comprehensive?

## Usage

When generating Rails code:

1. **Start simple** — Model method first, extract later if needed
2. **Check `references/stack.md`** — Verify gem choices
3. **Check `references/state-management.md`** — Enum or state record?
4. **Follow patterns** in the relevant building block reference
5. **Consult `references/anti-patterns.md`** — Avoid common mistakes
6. **Run through checklist** before finalizing

---

> "The best code is the code you don't write. The second best is the code that's obviously correct."

> "Vanilla Rails is plenty." — Jorge Manrubia, 37signals
