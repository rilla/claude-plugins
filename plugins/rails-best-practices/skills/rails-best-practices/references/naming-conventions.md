# Naming Conventions

Names should reveal intention and read like natural language. Good names make code self-documenting.

## Models: Business Domain Language

Name models after **business concepts**, not technical abstractions.

### ✅ Good Model Names

```ruby
# Business domain
Participant    # Not User (when they participate in something)
Cloud          # Not GeneratedImage
Booking        # Not Appointment or Reservation (match your domain)
Recording      # Not AudioFile
Closure        # The act of closing
Confirmation   # The act of confirming
```

### ❌ Bad Model Names

```ruby
# Technical/generic
User           # Unless you're building a user management system
Data
Info
Manager
Handler
Processor
Service        # Never a model name
```

### Guidelines

- **Use domain language your client uses** — If they say "participant", use Participant
- **Be specific** — Cloud (your domain) not Image (generic)
- **Nouns** — Models are things, not actions
- **Singular** — Rails convention (Card not Cards)

---

## Methods: Verbs for Actions, Predicates for State

### Verb Methods (Actions)

Methods that **do something** should be clear verbs. Read like commands.

```ruby
# ✅ GOOD: Natural, expressive verbs
card.close
card.reopen
card.gild
card.postpone
card.duplicate
board.publish
board.archive
booking.confirm
booking.cancel
recording.incinerate
recording.copy_to(destination)

# ❌ BAD: Procedural, setter-style, unclear
card.set_closed(true)
card.update_status(:closed)
card.mark_as_closed
card.perform_closure
card.close_card           # Redundant (card.close is clearer)
```

### Predicate Methods (State Queries)

Methods that **ask about state** end with `?`. Return boolean.

```ruby
# ✅ GOOD: Clear questions
card.closed?
card.open?
card.golden?
card.postponed?
booking.confirmed?
booking.cancelled?
recording.trashed?

# Derived from state records
def closed?
  closure.present?
end

def confirmed?
  confirmation.present?
end

# Derived from enums
def published?
  status == "published"
end
# Or Rails auto-generates: enum :status, %w[draft published]
```

### Method Naming Patterns

| Pattern | Example | Use For |
|---------|---------|---------|
| **Verb** | `close`, `publish`, `duplicate` | Actions that change state |
| **Verb!** | `close!`, `publish!`, `save!` | Actions that raise on failure |
| **Predicate?** | `closed?`, `published?`, `valid?` | State queries (return boolean) |
| **to_*** | `to_s`, `to_json`, `to_param` | Conversions |
| **as_*** | `as_json`, `as_csv` | Representations |

---

## Scopes: Adverbs and Adjectives

Scopes should read naturally when chained.

### Adverbs (How to Order)

```ruby
scope :chronologically,         -> { order(created_at: :asc) }
scope :reverse_chronologically, -> { order(created_at: :desc) }
scope :alphabetically,          -> { order(name: :asc) }
scope :recently,                -> { order(created_at: :desc).limit(10) }
```

### Adjectives (Which Ones)

```ruby
scope :active,     -> { where(active: true) }
scope :inactive,   -> { where(active: false) }
scope :published,  -> { where(status: :published) }
scope :pending,    -> { where(status: :pending) }
scope :upcoming,   -> { where(starts_at: Time.current..) }
scope :past,       -> { where(starts_at: ...Time.current) }
scope :today,      -> { where(starts_at: Time.current.all_day) }
scope :closed,     -> { joins(:closure) }
scope :open,       -> { where.missing(:closure) }
```

### Scope Chains Should Read Like English

```ruby
# ✅ GOOD: Reads naturally
Card.open.high_priority.chronologically
Booking.today.confirmed.preloaded
Recording.active.alphabetically

# ❌ BAD: Awkward reading
Card.get_open_cards.with_high_priority.ordered_by_created_at
```

### Special Scope: `preloaded`

Common pattern for eager loading associations:

```ruby
scope :preloaded, -> { includes(:creator, :tags, :comments) }

# Usage
@cards = current_board.cards.preloaded.open
```

---

## Concerns: "Has Trait" Semantics

Concerns must represent a clear **trait** or **capability**. Name with adjectives ending in **-able** or **-ible**.

### ✅ Good Concern Names

```ruby
# Clear "has trait" semantics
Closeable      # Can be closed
Publishable    # Can be published
Watchable      # Can be watched
Confirmable    # Can be confirmed
Cancellable    # Can be cancelled
Archivable     # Can be archived
Trashable      # Can be trashed
Schedulable    # Can be scheduled
Commentable    # Can have comments
Taggable       # Can have tags
```

### ❌ Bad Concern Names

```ruby
# Too vague, no clear trait
CardMethods
Helpers
Utils
CommonBehavior
SharedLogic

# Action-oriented (use verbs in methods, not concerns)
Closer
Publisher
Validator
```

### Concern Structure

Namespaced under model when specific to one model:

```ruby
# app/models/concerns/card/closeable.rb
module Card::Closeable
  extend ActiveSupport::Concern

  included do
    has_one :closure, dependent: :destroy
    scope :closed, -> { joins(:closure) }
  end

  def close
    # Implementation
  end
end
```

Shared concerns at top level:

```ruby
# app/models/concerns/commentable.rb
module Commentable
  extend ActiveSupport::Concern

  included do
    has_many :comments, as: :commentable, dependent: :destroy
  end

  def commented_by?(user)
    comments.exists?(user: user)
  end
end
```

---

## Controllers

### Standard CRUD Controllers

```ruby
CardsController
BookingsController
RecordingsController
```

### Namespaced Controllers

For scoping or organization:

```ruby
# Scoped by authentication
Participant::CardsController
Admin::UsersController

# Scoped by resource
Cards::ClosuresController      # Nested resource
Cards::CommentsController
```

### Controller Actions

Stick to Rails conventions:

```ruby
# Standard CRUD
index
show
new
create
edit
update
destroy

# Avoid custom actions
# ❌ BAD
def close
def archive
def publish

# ✅ GOOD: Use nested resources instead
# POST /cards/:id/closure (Cards::ClosuresController#create)
# DELETE /cards/:id/closure (Cards::ClosuresController#destroy)
```

---

## Jobs

Jobs orchestrate async work. Name with `Job` suffix.

### ✅ Good Job Names

```ruby
CardProcessingJob
BookingConfirmationJob
BookingReminderJob
RecordingTranscriptionJob
CloudGenerationJob
EmailDeliveryJob
```

### ❌ Bad Job Names

```ruby
ProcessCard              # Ambiguous
CardProcessor            # Sounds like a service object
HandleBookingConfirmation  # Verbose
DoStuff                  # Useless
```

### Job Method

Always `perform`:

```ruby
class BookingConfirmationJob < ApplicationJob
  def perform(booking)
    # Send confirmation
  end
end
```

---

## Policies (ActionPolicy)

Policies handle authorization. Name after the model + `Policy`.

```ruby
CardPolicy
BookingPolicy
RecordingPolicy
```

### Methods

```ruby
class CardPolicy < ApplicationPolicy
  def create?
    user.can_create_cards?
  end

  def update?
    record.board.owned_by?(user)
  end

  def destroy?
    user.admin? || record.created_by?(user)
  end

  def close?
    update? && record.open?
  end
end
```

---

## Forms (When Used)

Form objects handle complex multi-model operations. Name with `Form` suffix.

```ruby
# ✅ GOOD: Descriptive
BookingWithNewClientForm
ParticipantRegistrationForm
CloudGenerationRequestForm

# ❌ BAD: Too generic
BookingForm              # What about it?
FormObject
NewClientForm            # For what context?
```

---

## Queries (When Used)

Query objects for complex, reusable queries. Name with `Query` suffix.

```ruby
# ✅ GOOD: Specific
Cloud::SearchQuery
Booking::AvailabilityQuery
Recording::AdvancedFilterQuery

# ❌ BAD
CloudQuery               # Too generic
Search                   # Not descriptive
Finder                   # Vague
```

---

## Variables and Parameters

### Local Variables

```ruby
# ✅ GOOD: Descriptive
user = User.find(id)
booking = Booking.new(params)
confirmed_bookings = bookings.confirmed
upcoming_count = bookings.upcoming.count

# ❌ BAD: Too short, unclear
u = User.find(id)
b = Booking.new(params)
arr = bookings.confirmed
tmp = bookings.upcoming.count
```

### Instance Variables (Controllers)

```ruby
# ✅ GOOD: Match resource
@card = Card.find(params[:id])
@booking = Booking.new
@confirmed_bookings = current_account.bookings.confirmed

# ❌ BAD
@c = Card.find(params[:id])
@item = Booking.new
@list = current_account.bookings.confirmed
```

### Private Methods

```ruby
# ✅ GOOD: Descriptive verbs/nouns
def set_card
def card_params
def authorize_access
def send_notifications

# ❌ BAD
def do_stuff
def helper
def setup
```

---

## Database Tables and Columns

### Table Names

```ruby
# ✅ GOOD: Plural, snake_case
cards
bookings
closures
confirmations
booking_confirmations  # Join table

# ❌ BAD
card                   # Singular
Bookings               # Not snake_case
booking_confirmation   # Ambiguous for join tables
```

### Column Names

```ruby
# ✅ GOOD: Descriptive, snake_case
title
description
created_at
updated_at
closed_at
confirmed_at
starts_at
price_cents
user_id

# ❌ BAD
t                      # Too short
desc                   # Ambiguous
time                   # Which time?
price                  # Use integer cents, not float
```

### Foreign Keys

```ruby
# ✅ GOOD: model_id
user_id
board_id
booking_id

# Polymorphic
commentable_id
commentable_type
```

### Boolean Columns

```ruby
# ✅ GOOD: Predicate form
active
published
visible
confirmed

# ❌ BAD: Redundant prefix
is_active
is_published
has_confirmed
```

**Note:** Prefer state records over booleans when you need audit trail!

---

## Files and Directories

### Model Files

```ruby
app/models/card.rb
app/models/cloud.rb
app/models/cloud/card_generator.rb         # Namespaced class
app/models/concerns/closeable.rb           # Shared concern
app/models/concerns/card/closeable.rb      # Model-specific concern
```

### Controller Files

```ruby
app/controllers/cards_controller.rb
app/controllers/participant/cards_controller.rb
app/controllers/cards/closures_controller.rb
app/controllers/concerns/card_scoped.rb
```

### Test Files

```ruby
spec/models/card_spec.rb
spec/models/concerns/closeable_spec.rb
spec/requests/cards_spec.rb
spec/system/cards_spec.rb
spec/jobs/card_processing_job_spec.rb
spec/factories/cards.rb
```

---

## Constants

```ruby
# ✅ GOOD: SCREAMING_SNAKE_CASE
MAX_RETRIES = 3
DEFAULT_TIMEOUT = 30
API_ENDPOINT = "https://api.example.com"

# ❌ BAD
MaxRetries = 3         # Not screaming case
max_retries = 3        # Not a constant
```

---

## Summary Table

| Element | Convention | Example |
|---------|-----------|---------|
| **Model** | Singular, CamelCase, domain noun | `Card`, `Participant`, `Booking` |
| **Table** | Plural, snake_case | `cards`, `participants`, `bookings` |
| **Controller** | Plural + Controller | `CardsController` |
| **Job** | Noun + Job | `CardProcessingJob` |
| **Policy** | Model + Policy | `CardPolicy` |
| **Concern** | Adjective (-able/-ible) | `Closeable`, `Publishable` |
| **Form** | Descriptive + Form | `BookingWithNewClientForm` |
| **Query** | Descriptive + Query | `Cloud::SearchQuery` |
| **Scope** | Adverb or adjective | `chronologically`, `active`, `upcoming` |
| **Action method** | Verb | `close`, `publish`, `duplicate` |
| **Predicate method** | Adjective + ? | `closed?`, `published?`, `active?` |
| **Instance var** | @snake_case | `@card`, `@bookings` |
| **Local var** | snake_case | `confirmed_bookings`, `user` |
| **Constant** | SCREAMING_SNAKE_CASE | `MAX_RETRIES`, `API_KEY` |

---

## The Readability Test

Good names pass the **readability test** — code reads like English:

```ruby
# ✅ GOOD: Reads like a sentence
card.close
booking.confirm
recording.incinerate
Card.open.high_priority.chronologically
user.bookings.upcoming.today

# ❌ BAD: Awkward, unclear
card.set_status_to_closed
booking.mark_as_confirmed
recording.perform_incineration
Card.get_open.filter_by_priority(:high).order_chronologically
user.get_bookings.filter_upcoming.for_today
```

**When naming:** If you have to stop and think what something means, the name isn't good enough.
