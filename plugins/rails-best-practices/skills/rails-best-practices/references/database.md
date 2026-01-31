# Database Patterns

## Schema Design Principles

### Normalize Data

One concern per table. Avoid denormalized columns.

```ruby
# Bad - denormalized
create_table :clouds do |t|
  t.string :title
  t.string :participant_name      # Duplicated from participants
  t.string :participant_email     # Duplicated from participants
  t.integer :cards_count_cached   # Stale data risk
end

# Good - normalized
create_table :clouds do |t|
  t.string :title
  t.references :participant, foreign_key: true, null: false
  t.integer :cards_count, default: 0, null: false  # Counter cache
end
```

### Foreign Key Constraints

Always add foreign key constraints:

```ruby
class CreateClouds < ActiveRecord::Migration[7.1]
  def change
    create_table :clouds do |t|
      t.references :participant, null: false, foreign_key: true
      t.string :title, null: false
      t.timestamps
    end
  end
end
```

### Check Constraints

Use database-level constraints for data integrity:

```ruby
class AddConstraintsToClouds < ActiveRecord::Migration[7.1]
  def change
    # Ensure positive values
    add_check_constraint :clouds, "cards_count >= 0", name: "clouds_cards_count_positive"

    # Ensure non-empty strings
    add_check_constraint :clouds, "char_length(title) > 0", name: "clouds_title_not_empty"

    # Ensure valid state transitions (optional, complex)
    add_check_constraint :clouds,
      "state IN ('uploaded', 'analyzing', 'analyzed', 'generating', 'generated', 'failed')",
      name: "clouds_valid_state"
  end
end
```

### NOT NULL Defaults

Prefer NOT NULL with defaults over nullable columns:

```ruby
# Good
add_column :clouds, :featured, :boolean, default: false, null: false
add_column :clouds, :views_count, :integer, default: 0, null: false
add_column :clouds, :metadata, :jsonb, default: {}, null: false

# Avoid when possible
add_column :clouds, :featured, :boolean  # Nullable, three-state logic
```

## PostgreSQL Features

### Enum Types

Use PostgreSQL enums for fixed value sets:

```ruby
class AddStateEnumToClouds < ActiveRecord::Migration[7.1]
  def up
    create_enum :cloud_state, %w[uploaded analyzing analyzed generating generated failed]
    add_column :clouds, :state, :enum, enum_type: :cloud_state, default: "uploaded", null: false
    add_index :clouds, :state
  end

  def down
    remove_column :clouds, :state
    drop_enum :cloud_state
  end
end
```

### JSONB Columns

For flexible metadata:

```ruby
class AddMetadataToClouds < ActiveRecord::Migration[7.1]
  def change
    add_column :clouds, :metadata, :jsonb, default: {}, null: false

    # Index for querying
    add_index :clouds, :metadata, using: :gin
  end
end

# Usage in model:
class Cloud < ApplicationRecord
  store_accessor :metadata, :width, :height, :format, :analyzed_at
end
```

### Array Columns

```ruby
add_column :clouds, :tags, :string, array: true, default: [], null: false
add_index :clouds, :tags, using: :gin

# Query:
Cloud.where("'landscape' = ANY(tags)")
```

## Indexing Strategy

### Index Foreign Keys

Always index foreign key columns:

```ruby
# Automatic with references
t.references :participant, foreign_key: true  # Adds index

# Manual
add_index :clouds, :participant_id
```

### Index Query Patterns

```ruby
# Single column queries
add_index :clouds, :state
add_index :clouds, :created_at

# Composite queries (column order matters)
add_index :clouds, [:participant_id, :created_at]
add_index :clouds, [:state, :created_at]

# Partial indexes for common queries
add_index :clouds, :created_at, where: "state = 'generated'", name: "index_clouds_generated_by_date"

# Unique constraints
add_index :clouds, [:participant_id, :slug], unique: true
```

### Index for Scopes

Match indexes to your model scopes:

```ruby
# Model scope:
scope :recent_active, -> { where(archived_at: nil).order(created_at: :desc) }

# Supporting index:
add_index :clouds, :created_at, where: "archived_at IS NULL", name: "index_clouds_active_by_date"
```

## Migrations Best Practices

### Reversible Migrations

```ruby
class AddProcessingToClouds < ActiveRecord::Migration[7.1]
  def change
    add_column :clouds, :processed_at, :datetime
    add_column :clouds, :processing_error, :text
  end
end
```

### Non-Reversible Operations

```ruby
class MigrateCloudStates < ActiveRecord::Migration[7.1]
  def up
    Cloud.where(legacy_status: "complete").update_all(state: "generated")
    remove_column :clouds, :legacy_status
  end

  def down
    raise ActiveRecord::IrreversibleMigration
  end
end
```

### Safe Operations for Production

```ruby
# Add column with default (safe in PostgreSQL 11+)
add_column :clouds, :featured, :boolean, default: false, null: false

# Add index concurrently (doesn't lock table)
add_index :clouds, :state, algorithm: :concurrently

# Backfill in batches
Cloud.in_batches.update_all(featured: false)
```

### Separate Structure from Data

```ruby
# Migration 1: Add column (fast, deploy first)
class AddFeaturedToClouds < ActiveRecord::Migration[7.1]
  def change
    add_column :clouds, :featured, :boolean
  end
end

# Migration 2: Backfill data (can run later)
class BackfillFeaturedOnClouds < ActiveRecord::Migration[7.1]
  disable_ddl_transaction!

  def up
    Cloud.in_batches.update_all(featured: false)
  end
end

# Migration 3: Add constraint (after backfill complete)
class AddFeaturedConstraintToClouds < ActiveRecord::Migration[7.1]
  def change
    change_column_null :clouds, :featured, false
    change_column_default :clouds, :featured, false
  end
end
```

## Counter Caches

### Basic Setup

```ruby
# Migration
add_column :participants, :clouds_count, :integer, default: 0, null: false

# Model
belongs_to :participant, counter_cache: true

# Reset if needed
Participant.reset_counters(participant.id, :clouds)
```

### Custom Counter Cache

```ruby
class Cloud < ApplicationRecord
  belongs_to :participant, counter_cache: :active_clouds_count

  # Only count non-archived
  after_save :update_counter_cache
  after_destroy :update_counter_cache

  private

  def update_counter_cache
    participant.update_column(:active_clouds_count, participant.clouds.active.count)
  end
end
```

## Query Optimization

### Eager Loading

```ruby
# N+1 problem
Cloud.all.each { |c| puts c.participant.name }  # Bad

# Eager load
Cloud.includes(:participant).each { |c| puts c.participant.name }  # Good

# Preload vs includes vs eager_load
Cloud.preload(:cards)      # Separate query
Cloud.includes(:cards)     # Rails chooses strategy
Cloud.eager_load(:cards)   # LEFT OUTER JOIN
```

### Select Only Needed Columns

```ruby
# Load full records
Cloud.all  # SELECT * FROM clouds

# Select specific columns
Cloud.select(:id, :title, :state)  # SELECT id, title, state FROM clouds

# Pluck for arrays
Cloud.pluck(:id)  # Returns array of IDs
```

### Batch Processing

```ruby
# Memory efficient
Cloud.find_each(batch_size: 1000) do |cloud|
  cloud.process
end

# With relation
Cloud.where(state: :pending).find_each do |cloud|
  CloudProcessingJob.perform_later(cloud)
end
```

## Schema.rb Hygiene

Keep schema.rb clean:
- Remove commented-out code
- Ensure consistent formatting
- Verify all indexes are intentional
- Check for orphaned foreign keys
