# State Management: Enums vs State Records

One of the key decisions in Rails models is how to represent state. This guide helps you choose between **enums** and **state records**.

## Quick Decision Matrix

| Scenario | Use | Why |
|----------|-----|-----|
| Workflow states without audit needs | **Enum** | Simple, PostgreSQL-backed, perfect for status tracking |
| User-triggered state changes | **State Record** | Track who, when, why |
| Reversible actions | **State Record** | Delete record to undo |
| State with metadata (reason, notes) | **State Record** | Associated model holds context |
| Simple flags (active/inactive) | **Enum or Boolean** | Depends on whether you need history |
| Complex state machines | **Enum + Transitions** | Use enum with validation, no gems |

## Approach 1: Enums

### When to Use Enums

- **Workflow states** (draft → review → published → archived)
- **Status tracking** (pending → processing → completed → failed)
- **Priority levels** (low, medium, high, urgent)
- **Categories** (personal, work, urgent)
- **No audit trail needed** — You don't care WHO changed it or WHY

### Implementation

```ruby
class Article < ApplicationRecord
  # PostgreSQL enum (recommended)
  enum :status, %w[draft reviewing published archived].index_by(&:itself)

  # Transitions with validation
  def publish!
    raise "Cannot publish from #{status}" unless can_publish?

    transaction do
      update!(status: :published, published_at: Time.current)
      notify_subscribers
    end
  end

  def archive!
    raise "Cannot archive unpublished article" unless published?
    update!(status: :archived, archived_at: Time.current)
  end

  private
    def can_publish?
      draft? || reviewing?
    end
end
```

### Migration

```ruby
class AddStatusToArticles < ActiveRecord::Migration[7.1]
  def change
    create_enum :article_status, %w[draft reviewing published archived]

    add_column :articles, :status, :enum, enum_type: :article_status, default: 'draft', null: false
    add_column :articles, :published_at, :datetime
    add_column :articles, :archived_at, :datetime

    add_index :articles, :status
  end
end
```

### Scopes

```ruby
class Article < ApplicationRecord
  scope :draft, -> { where(status: :draft) }
  scope :published, -> { where(status: :published) }
  scope :recent_published, -> { published.where(published_at: 1.week.ago..) }
end
```

### Benefits

✅ **Simple** — Built into Rails and PostgreSQL
✅ **Fast** — Indexed enum column
✅ **Type-safe** — PostgreSQL enforces valid values
✅ **Convenient** — Auto-generated predicate methods (`article.published?`)
✅ **Compact** — One column, not a join

### Drawbacks

❌ **No audit trail** — Can't see who changed it or why
❌ **No history** — Only current state, no transitions log
❌ **Limited metadata** — Can't attach reason/notes to state change

---

## Approach 2: State Records

### When to Use State Records

- **User actions** that need attribution (close, cancel, approve)
- **Audit trail required** — WHO did it, WHEN, WHY
- **Metadata needed** — Cancellation reason, approval notes
- **Reversible** — Delete record to undo (reopen, unarchive)
- **State transitions are events** — You want to log each transition

### Implementation

```ruby
class Card < ApplicationRecord
  include Closeable
  include Archivable

  has_one :closure, dependent: :destroy
  has_one :archive, dependent: :destroy

  def closed?
    closure.present?
  end

  def archived?
    archive.present?
  end
end

# app/models/concerns/card/closeable.rb
module Card::Closeable
  extend ActiveSupport::Concern

  included do
    has_one :closure, dependent: :destroy

    scope :closed, -> { joins(:closure) }
    scope :open, -> { where.missing(:closure) }
  end

  def close(reason: nil)
    return if closed?

    transaction do
      create_closure!(
        user: Current.user,
        reason: reason,
        closed_at: Time.current
      )
      events.create!(action: :closed, creator: Current.user)
    end

    notify_watchers_of_closure
  end

  def reopen
    return unless closed?

    transaction do
      closure.destroy!
      events.create!(action: :reopened, creator: Current.user)
    end
  end

  private
    def notify_watchers_of_closure
      watchers.each { |w| CardNotificationJob.perform_later(w, self, :closed) }
    end
end

# app/models/closure.rb
class Closure < ApplicationRecord
  belongs_to :card, touch: true
  belongs_to :user, default: -> { Current.user }

  validates :reason, length: { maximum: 500 }
end
```

### Migration

```ruby
class CreateClosures < ActiveRecord::Migration[7.1]
  def change
    create_table :closures do |t|
      t.references :card, null: false, foreign_key: true, index: { unique: true }
      t.references :user, null: false, foreign_key: true
      t.datetime :closed_at, null: false
      t.text :reason

      t.timestamps
    end
  end
end
```

### Scopes

```ruby
# In Card model
scope :closed, -> { joins(:closure) }
scope :open, -> { where.missing(:closure) }
scope :closed_by, ->(user) { joins(:closure).where(closures: { user: user }) }
scope :closed_this_week, -> { joins(:closure).where(closures: { closed_at: 1.week.ago.. }) }
```

### Benefits

✅ **Full audit trail** — Who closed it, when, why
✅ **Metadata** — Reason, notes, additional context
✅ **Reversible** — Delete record to reopen/undo
✅ **Query flexibility** — Join on state records for complex queries
✅ **Event history** — Can keep all transitions if needed (has_many instead of has_one)
✅ **Natural scopes** — `joins(:closure)` reads clearly

### Drawbacks

❌ **More complex** — Extra model, migration, concern
❌ **Slower queries** — Requires join (though indexed FK is fast)
❌ **More code** — Concern + state model vs single enum

---

## Hybrid Approach

Sometimes you need both! Use enum for workflow + state records for user actions.

### Example: Booking System

```ruby
class Booking < ApplicationRecord
  # Enum for workflow status
  enum :status, %w[pending confirmed completed cancelled].index_by(&:itself)

  # State records for user actions
  has_one :confirmation, dependent: :destroy
  has_one :cancellation, dependent: :destroy

  def confirmed?
    confirmation.present?
  end

  def cancelled?
    cancellation.present?
  end

  def confirm!
    return if confirmed?

    transaction do
      create_confirmation!(user: Current.user, confirmed_at: Time.current)
      update!(status: :confirmed)
      BookingConfirmationJob.perform_later(self)
    end
  end

  def cancel!(reason:)
    return if cancelled?

    transaction do
      create_cancellation!(
        user: Current.user,
        reason: reason,
        cancelled_at: Time.current
      )
      update!(status: :cancelled)
      BookingCancellationJob.perform_later(self)
    end
  end
end

# app/models/confirmation.rb
class Confirmation < ApplicationRecord
  belongs_to :booking, touch: true
  belongs_to :user, default: -> { Current.user }
end

# app/models/cancellation.rb
class Cancellation < ApplicationRecord
  belongs_to :booking, touch: true
  belongs_to :user, default: -> { Current.user }

  validates :reason, presence: true, length: { maximum: 500 }
end
```

### Why Hybrid?

- **Enum for status** — Fast queries, simple workflow tracking
- **State records for user actions** — Audit trail for critical transitions
- **Best of both** — Speed + context where it matters

---

## Decision Flowchart

```
Does the state change need to track WHO did it?
  └─ YES → State Record
  └─ NO → Continue

Does the state have associated metadata (reason, notes)?
  └─ YES → State Record
  └─ NO → Continue

Is the action reversible by deleting the state?
  └─ YES → State Record
  └─ NO → Continue

Is this a simple workflow status?
  └─ YES → Enum
  └─ NO → State Record (when in doubt, audit trail wins)
```

---

## Migration Path

### From Boolean to State Record

```ruby
# Before
class Card < ApplicationRecord
  # closed: boolean
  # closed_at: datetime
end

# After
class Card < ApplicationRecord
  has_one :closure, dependent: :destroy

  def closed?
    closure.present?
  end
end

# Migration
class MigrateBooleanToStateRecord < ActiveRecord::Migration[7.1]
  def up
    create_table :closures do |t|
      t.references :card, null: false, foreign_key: true, index: { unique: true }
      t.references :user, null: true, foreign_key: true  # null during migration
      t.datetime :closed_at, null: false
      t.timestamps
    end

    # Migrate existing closed cards
    Card.where(closed: true).find_each do |card|
      Closure.create!(
        card: card,
        closed_at: card.closed_at || card.updated_at,
        user: nil  # Unknown for old data
      )
    end

    remove_column :cards, :closed
    remove_column :cards, :closed_at
  end

  def down
    add_column :cards, :closed, :boolean, default: false
    add_column :cards, :closed_at, :datetime

    Closure.find_each do |closure|
      closure.card.update!(closed: true, closed_at: closure.closed_at)
    end

    drop_table :closures
  end
end
```

### From Enum to State Record

Only if you realize you need audit trail. Usually, start with enum and migrate if needed.

---

## Examples from Popular Gems/Apps

### Basecamp (37signals)
- Uses state records extensively (Recording::Trash, Recording::Archive)
- `recording.trash!` creates Trash record
- `recording.restore!` deletes Trash record

### GitHub
- Pull request states: enum for status (open/closed/merged)
- But: `reviews`, `approvals` are separate models for audit trail

### Shopify
- Order status: enum
- But: `refunds`, `cancellations` are models with reasons

---

## Summary

| Factor | Enum | State Record | Hybrid |
|--------|------|--------------|--------|
| **Simplicity** | ✅ Very simple | ❌ More complex | ⚠️ Moderate |
| **Audit trail** | ❌ No | ✅ Full | ✅ For user actions |
| **Metadata** | ❌ Limited | ✅ Yes | ✅ For user actions |
| **Performance** | ✅ Fast | ⚠️ Join required | ⚠️ Join when needed |
| **Reversibility** | ❌ Manual | ✅ Delete record | ✅ Delete record |
| **Use case** | Workflow status | User actions | Complex workflows |

**Default recommendation:**
- Start with **enum** for simple workflow states
- Use **state records** for user-triggered actions that need context
- Use **hybrid** when you need both workflow tracking AND audit trail

**When in doubt:** If someone might ask "Who closed this and why?", use a state record.
