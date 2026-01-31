# Anti-Patterns and How to Fix Them

Common mistakes in Rails development and their solutions.

## Architecture Anti-Patterns

### ❌ Service Objects

**Problem:** Creating `app/services/` directory with procedural classes.

```ruby
# ❌ BAD: Service object
class CardClosureService
  def initialize(card, user, reason = nil)
    @card = card
    @user = user
    @reason = reason
  end

  def call
    @card.transaction do
      @card.create_closure!(user: @user, reason: @reason)
      @card.events.create!(action: :closed, creator: @user)
      notify_watchers
    end
  end

  private
    def notify_watchers
      @card.watchers.each { |w| CardNotificationJob.perform_later(w, @card, :closed) }
    end
end

# Usage (awkward)
CardClosureService.new(card, current_user, reason).call
```

**✅ Solution: Rich Model Method**

```ruby
# app/models/card.rb
class Card < ApplicationRecord
  include Closeable
end

# app/models/concerns/card/closeable.rb
module Card::Closeable
  extend ActiveSupport::Concern

  included do
    has_one :closure, dependent: :destroy
  end

  def close(reason: nil)
    transaction do
      create_closure!(user: Current.user, reason: reason)
      events.create!(action: :closed, creator: Current.user)
    end
    notify_watchers_of_closure
  end

  private
    def notify_watchers_of_closure
      watchers.each { |w| CardNotificationJob.perform_later(w, self, :closed) }
    end
end

# Usage (natural)
card.close(reason: "Completed")
```

**When logic is complex (>15 lines or external API):**

```ruby
# ✅ GOOD: Namespaced model class
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
      { image: @cloud.image_url, style: @style }
    end
end

# Usage from model
class Cloud < ApplicationRecord
  def generate_card(style: :default)
    Cloud::CardGenerator.new(self, style: style).call
  end
end
```

---

### ❌ Custom Controller Actions

**Problem:** Adding non-REST actions to controllers.

```ruby
# ❌ BAD: Custom actions
class CardsController < ApplicationController
  def close
    @card.update!(closed: true)
    redirect_to @card
  end

  def reopen
    @card.update!(closed: false)
    redirect_to @card
  end

  def archive
    @card.update!(archived: true)
    redirect_to @card
  end
end

# routes.rb
resources :cards do
  post :close
  post :reopen
  post :archive
end
```

**✅ Solution 1: Model Method with Standard Actions**

```ruby
# Keep standard CRUD
class CardsController < ApplicationController
  def update
    if card_params[:action] == "close"
      @card.close
    elsif card_params[:action] == "reopen"
      @card.reopen
    else
      @card.update!(card_params)
    end

    redirect_to @card, notice: t(".updated")
  end
end
```

**✅ Solution 2: Nested Resources (37signals Pattern)**

*Note: This is the 37signals preferred approach, but we're not enforcing it by default. Use if it fits your style.*

```ruby
# routes.rb
resources :cards do
  resource :closure, only: [:create, :destroy]   # POST to close, DELETE to reopen
  resource :archive, only: [:create, :destroy]
end

# app/controllers/cards/closures_controller.rb
class Cards::ClosuresController < ApplicationController
  include CardScoped

  def create
    @card.close
    redirect_to @card, notice: t(".created")
  end

  def destroy
    @card.reopen
    redirect_to @card, notice: t(".destroyed")
  end
end
```

---

### ❌ Fat Controllers

**Problem:** Business logic in controllers.

```ruby
# ❌ BAD: Fat controller
class BookingsController < ApplicationController
  def create
    @booking = current_account.bookings.new(booking_params)
    @booking.status = :pending

    if @booking.save
      # Business logic in controller!
      if @booking.requires_confirmation?
        BookingConfirmationJob.perform_later(@booking)
      end

      if @booking.service.notify_staff?
        StaffNotificationJob.perform_later(@booking)
      end

      if @booking.client.first_booking?
        @booking.client.update!(first_booking_at: Time.current)
        WelcomeEmailJob.perform_later(@booking.client)
      end

      redirect_to @booking, notice: "Booking created!"
    else
      render :new, status: :unprocessable_entity
    end
  end
end
```

**✅ Solution: Thin Controller, Model Callbacks**

```ruby
# app/controllers/bookings_controller.rb
class BookingsController < ApplicationController
  def create
    @booking = current_account.bookings.create!(booking_params)
    redirect_to @booking, notice: t(".created")
  rescue ActiveRecord::RecordInvalid => e
    @booking = e.record
    render :new, status: :unprocessable_entity
  end
end

# app/models/booking.rb
class Booking < ApplicationRecord
  belongs_to :account
  belongs_to :client
  belongs_to :service

  after_create :send_confirmations
  after_create :handle_first_booking

  private
    def send_confirmations
      BookingConfirmationJob.perform_later(self) if requires_confirmation?
      StaffNotificationJob.perform_later(self) if service.notify_staff?
    end

    def handle_first_booking
      return unless client.first_booking?

      client.update!(first_booking_at: Time.current)
      WelcomeEmailJob.perform_later(client)
    end
end
```

---

### ❌ Boolean Columns for State (When Audit Trail Needed)

**Problem:** Using booleans when you need to track WHO/WHEN/WHY.

```ruby
# ❌ BAD: Lost context
class Card < ApplicationRecord
  # closed: boolean
  # closed_at: datetime
  # closed_by_id: integer
end

# Who closed it? Have to track separately
# Why closed? No place to store reason
# Can't query "cards closed this week by user X"
```

**✅ Solution: State Records**

```ruby
class Card < ApplicationRecord
  has_one :closure, dependent: :destroy

  def closed?
    closure.present?
  end

  def close(reason: nil)
    create_closure!(user: Current.user, reason: reason)
  end

  def reopen
    closure&.destroy
  end
end

class Closure < ApplicationRecord
  belongs_to :card, touch: true
  belongs_to :user, default: -> { Current.user }

  validates :reason, length: { maximum: 500 }
end

# Now you can query:
Card.joins(:closure).where(closures: { user: current_user })
Card.joins(:closure).where(closures: { created_at: 1.week.ago.. })
```

**See `references/state-management.md` for decision tree.**

---

## Code Quality Anti-Patterns

### ❌ Time.now Instead of Time.current

**Problem:** Ignores Rails timezone configuration.

```ruby
# ❌ BAD: System timezone
Time.now
Date.today
Time.parse("2024-01-15")

# Can cause bugs when server timezone != user timezone
```

**✅ Solution: Rails Timezone Methods**

```ruby
# ✅ GOOD: Respects Time.zone
Time.current
Date.current
Time.zone.parse("2024-01-15")
1.week.ago
2.days.from_now
```

---

### ❌ Float for Money

**Problem:** Floating point precision errors.

```ruby
# ❌ BAD: Precision issues
add_column :services, :price, :decimal, precision: 8, scale: 2
add_column :services, :price, :float  # Even worse!

# Calculations can be incorrect
0.1 + 0.2  # => 0.30000000000000004
```

**✅ Solution: Integer Cents**

```ruby
# ✅ GOOD: Exact precision
add_column :services, :price_cents, :integer, default: 0, null: false

class Service < ApplicationRecord
  def price
    price_cents / 100.0
  end

  def price=(value)
    self.price_cents = (value.to_f * 100).round
  end
end
```

---

### ❌ Unscoped Queries (Security Risk)

**Problem:** Not scoping to current tenant/user.

```ruby
# ❌ BAD: Security vulnerability!
@booking = Booking.find(params[:id])
@card = Card.find_by(number: params[:number])

# User can access ANY booking/card by guessing IDs!
```

**✅ Solution: Always Scope**

```ruby
# ✅ GOOD: Scoped to current tenant
@booking = current_account.bookings.find(params[:id])
@card = current_account.cards.find_by!(number: params[:number])

# Or scope to user permissions
@card = current_user.accessible_cards.find(params[:id])
```

---

### ❌ N+1 Queries

**Problem:** Loading associations in a loop.

```ruby
# ❌ BAD: N+1 queries
@bookings = current_account.bookings.upcoming

# In view:
<% @bookings.each do |booking| %>
  <%= booking.client.name %>       # N+1!
  <%= booking.service.title %>     # N+1!
  <%= booking.user.email %>        # N+1!
<% end %>
```

**✅ Solution: Eager Loading**

```ruby
# ✅ GOOD: Single query with joins
@bookings = current_account.bookings
              .includes(:client, :service, :user)
              .upcoming

# Or create a preloaded scope
class Booking < ApplicationRecord
  scope :preloaded, -> { includes(:client, :service, :user, :location) }
end

@bookings = current_account.bookings.preloaded.upcoming
```

---

### ❌ Hardcoded User-Facing Strings

**Problem:** Not using I18n.

```ruby
# ❌ BAD: Hardcoded
redirect_to @booking, notice: "Booking created successfully!"
validates :title, presence: { message: "can't be blank" }
flash[:error] = "Something went wrong"
```

**✅ Solution: I18n Keys**

```ruby
# ✅ GOOD: I18n
redirect_to @booking, notice: t(".created")
validates :title, presence: { message: :blank }  # Uses locale file
flash[:error] = t("errors.generic")

# config/locales/en.yml
en:
  bookings:
    create:
      created: "Booking created successfully!"
  errors:
    generic: "Something went wrong"
```

---

### ❌ Direct ENV Access

**Problem:** No type safety, hard to test, scattered configuration.

```ruby
# ❌ BAD: Direct ENV access
ENV["API_KEY"]
ENV["MAX_RETRIES"].to_i
ENV["FEATURE_ENABLED"] == "true"

# What if ENV var is missing? nil error!
# What if it's the wrong type? Runtime error!
```

**✅ Solution: Anyway Config**

```ruby
# ✅ GOOD: Typed configuration
class ApiConfig < ApplicationConfig
  attr_config :api_key, :max_retries, :feature_enabled

  required :api_key
  coerce_types max_retries: :integer, feature_enabled: :boolean

  on_load :validate_api_key

  private
    def validate_api_key
      raise "Invalid API key" unless api_key&.start_with?("sk_")
    end
end

# Usage
ApiConfig.new.api_key           # Typed, validated
ApiConfig.new.max_retries       # Integer (coerced)
ApiConfig.new.feature_enabled   # Boolean (coerced)
```

**See `references/configuration.md` for details.**

---

## Testing Anti-Patterns

### ❌ Date-Sensitive Tests Without Freezing Time

**Problem:** Tests fail when run near midnight or in parallel.

```ruby
# ❌ BAD: Flaky test
it "includes today's bookings" do
  booking = create(:booking, date: Date.current)
  expect(Booking.today).to include(booking)
  # Fails if Date.current changes between fixture load and test execution!
end
```

**✅ Solution: Freeze Time**

```ruby
# ✅ GOOD: Stable test
it "includes today's bookings" do
  booking = create(:booking, date: Date.current)
  travel_to booking.date.to_time

  expect(Booking.today).to include(booking)
end

# Or freeze at start
it "creates booking for today" do
  freeze_time

  post bookings_path, params: { booking: { date: Date.current } }
  expect(Booking.last.date).to eq(Date.current)
end
```

---

### ❌ Overly Complex Factories

**Problem:** Factories that create 10 associated records.

```ruby
# ❌ BAD: Slow, creates too much
FactoryBot.define do
  factory :booking do
    account
    client { create(:client, :with_full_profile) }   # Creates 5 more records!
    service { create(:service, :with_pricing_tiers) } # Creates 3 more!
    location
    user { create(:user, :admin_with_permissions) }   # Creates permissions!
    # Every booking test now creates 15+ records!
  end
end
```

**✅ Solution: Minimal Factories + Traits**

```ruby
# ✅ GOOD: Minimal default, explicit traits
FactoryBot.define do
  factory :booking do
    account
    client
    service
    date { Date.current }
    time { Time.current }
    status { :pending }

    trait :confirmed do
      after(:create) do |booking|
        create(:confirmation, booking: booking)
      end
    end

    trait :with_full_details do
      location
      notes { "Full details booking" }
    end
  end
end

# Usage: only create what you need
create(:booking)                          # Minimal
create(:booking, :confirmed)              # With confirmation
create(:booking, :with_full_details)      # With extras
```

---

## Database Anti-Patterns

### ❌ Missing Foreign Key Constraints

**Problem:** Database allows orphaned records.

```ruby
# ❌ BAD: No constraint
create_table :closures do |t|
  t.integer :card_id
  t.integer :user_id
end

# Can delete card, leaving orphaned closures!
```

**✅ Solution: Foreign Keys + Cascades**

```ruby
# ✅ GOOD: Database enforced
create_table :closures do |t|
  t.references :card, null: false, foreign_key: true
  t.references :user, null: false, foreign_key: true
end

# Or with cascade
create_table :closures do |t|
  t.references :card, null: false, foreign_key: { on_delete: :cascade }
  t.references :user, null: false, foreign_key: true
end
```

---

### ❌ Missing Indexes

**Problem:** Slow queries on foreign keys and frequently queried columns.

```ruby
# ❌ BAD: No indexes
create_table :bookings do |t|
  t.references :account, foreign_key: true  # Has index ✓
  t.date :date                               # No index ❌
  t.string :status                           # No index ❌
end

# Queries on date/status will be slow!
```

**✅ Solution: Index Frequently Queried Columns**

```ruby
# ✅ GOOD: Indexed
create_table :bookings do |t|
  t.references :account, foreign_key: true, index: true
  t.date :date, null: false
  t.string :status, null: false, default: "pending"

  t.timestamps
end

add_index :bookings, :date
add_index :bookings, :status
add_index :bookings, [:account_id, :date]  # Composite for common query
add_index :bookings, [:account_id, :status, :date]  # For filtered lists
```

---

### ❌ Denormalized Data Without Purpose

**Problem:** Duplicating data without clear performance need.

```ruby
# ❌ BAD: Unnecessary denormalization
create_table :bookings do |t|
  t.references :service, foreign_key: true
  t.integer :service_price_cents  # Duplicates service.price_cents
  t.string :service_name          # Duplicates service.name
end

# Now you have to keep this in sync manually!
```

**✅ Solution: Normalize, Use Joins**

```ruby
# ✅ GOOD: Normalized
create_table :bookings do |t|
  t.references :service, foreign_key: true
end

# Query with join when needed
Booking.joins(:service).select("bookings.*, services.price_cents as service_price_cents")

# OR denormalize ONLY if proven performance issue + add callback to sync
class Booking < ApplicationRecord
  belongs_to :service

  before_save :cache_service_price, if: -> { service_id_changed? }

  private
    def cache_service_price
      self.service_price_cents = service.price_cents
    end
end
```

---

## Forbidden Gems and Patterns

| Don't Use | Use Instead | Why |
|-----------|-------------|-----|
| **Devise** | `has_secure_password` + sessions | Simpler, ~150 lines vs gem |
| **Pundit / CanCanCan** | ActionPolicy | More explicit, better scoping |
| **ActiveAdmin** | Build custom admin | More control, less magic |
| **AASM / StateMachine gems** | Enums or state records | Simpler, no DSL to learn |
| **dry-types / Virtus** | ActiveModel + plain Ruby | Native Rails patterns |
| **Interactor / Trailblazer** | Model methods | Less abstraction |
| **Sidekiq** (if you don't need it) | SolidQueue | Database-backed, simpler |
| **Redis** (for jobs/cache) | SolidQueue / SolidCache | One less service to run |

---

## Deployment Anti-Patterns

### ❌ No Database Constraints

**Problem:** Relying only on ActiveRecord validations.

```ruby
# ❌ BAD: Only model validation
class User < ApplicationRecord
  validates :email, presence: true, uniqueness: true
end

# Race condition: two requests can create duplicate emails!
```

**✅ Solution: Database Constraints + Model Validations**

```ruby
# Migration
add_index :users, :email, unique: true
change_column_null :users, :email, false

# Model (keep validation for error messages)
class User < ApplicationRecord
  validates :email, presence: true, uniqueness: true
end
```

---

### ❌ No Rollback Plan

**Problem:** Deploying migrations without down method.

```ruby
# ❌ BAD: Can't rollback
class AddStatusToBookings < ActiveRecord::Migration[7.1]
  def change
    add_column :bookings, :status, :string
    Booking.update_all(status: "pending")
  end
end
```

**✅ Solution: Reversible Migrations**

```ruby
# ✅ GOOD: Can rollback
class AddStatusToBookings < ActiveRecord::Migration[7.1]
  def up
    add_column :bookings, :status, :string
    Booking.update_all(status: "pending")
  end

  def down
    remove_column :bookings, :status
  end
end
```

---

## Quick Anti-Pattern Checklist

Before deploying, check for:

- [ ] No `app/services/` directory
- [ ] No service objects (use model methods + concerns)
- [ ] No custom controller actions (or justified)
- [ ] No fat controllers (logic in models)
- [ ] No booleans for stateful actions (use state records when audit needed)
- [ ] No `Time.now` (use `Time.current`)
- [ ] No float for money (use integer cents)
- [ ] No unscoped queries (always scope to tenant/user)
- [ ] No N+1 queries (use `includes`)
- [ ] No hardcoded strings (use I18n)
- [ ] No direct ENV access (use Anyway Config)
- [ ] No date tests without `travel_to`
- [ ] Foreign keys on all references
- [ ] Indexes on frequently queried columns
- [ ] Database constraints match validations
- [ ] All migrations reversible

---

**Remember:** Anti-patterns exist to solve problems. If you have a genuine need, document it. But 95% of the time, the Rails Way is simpler and better.
