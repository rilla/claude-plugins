# Rails Best Practices - Claude Code Plugin

> **⚠️ EXPERIMENTAL / WORK IN PROGRESS ⚠️**
>
> This plugin is an **exploration and experiment** in consolidating Rails best practices from multiple sources. It is NOT production-ready, NOT officially endorsed by Evil Martians or 37signals, and should be considered a **test/proof-of-concept**.
>
> **Use at your own risk.** Feedback, contributions, and improvements welcome!

---

> **Comprehensive Rails development standards combining Evil Martians' conventions with 37signals' One Person Framework philosophy.**

A Claude Code plugin that guides AI-assisted Rails development with proven patterns from industry leaders. Write Rails code so clear and simple that one developer can understand the entire system.

## Philosophy

This plugin consolidates two complementary approaches to Rails development:

1. **Evil Martians' Standards** ([palkan's AGENTS.md](https://gist.github.com/palkan/482010bd6ec434685f106779e863d0ef)) - Practical, convention-heavy patterns from a team building complex Rails applications
2. **37signals' One Person Framework** - DHH and team's philosophy of keeping Rails simple enough for one person to understand and maintain

### Core Principles

- **Conceptual Compression** - Hide complexity behind simple, natural APIs
- **Rich Models, Thin Controllers** - Business logic in models, HTTP handling in controllers
- **Model Methods First** - Prefer `card.close` over service objects or form objects
- **Follow Rails Conventions** - Leverage the framework, don't fight it
- **Domain Language** - Name models after business concepts (Participant, Cloud, not User, Image)
- **Vanilla Rails** - Use built-in features before adding gems

### The "One Person" Test

**Can one developer understand this codebase in an afternoon?** If not, simplify.

## What This Plugin Provides

### 1. Decision Trees

Clear guidance on common architectural decisions:
- **State Management** - Enum vs state records (audit trail consideration)
- **Logic Placement** - Model methods vs namespaced classes vs form objects
- **Testing** - RSpec + FactoryBot patterns
- **Technology Stack** - Approved gems and why

### 2. Comprehensive References

Detailed patterns for every layer:
- [`stack.md`](skills/rails-best-practices/references/stack.md) - Technology choices and rationale
- [`models.md`](skills/rails-best-practices/references/models.md) - Model organization, concerns, extraction
- [`state-management.md`](skills/rails-best-practices/references/state-management.md) - **NEW:** Enum vs state records decision tree
- [`controllers.md`](skills/rails-best-practices/references/controllers.md) - Thin controller patterns
- [`database.md`](skills/rails-best-practices/references/database.md) - Schema design, constraints, indexes
- [`jobs.md`](skills/rails-best-practices/references/jobs.md) - Background job orchestration
- [`views.md`](skills/rails-best-practices/references/views.md) - Hotwire, ViewComponent, Stimulus
- [`testing.md`](skills/rails-best-practices/references/testing.md) - RSpec organization, factories
- [`configuration.md`](skills/rails-best-practices/references/configuration.md) - Anyway Config patterns
- [`forms-queries.md`](skills/rails-best-practices/references/forms-queries.md) - When to use form/query objects
- [`naming-conventions.md`](skills/rails-best-practices/references/naming-conventions.md) - **NEW:** Comprehensive naming guide
- [`anti-patterns.md`](skills/rails-best-practices/references/anti-patterns.md) - Common mistakes and solutions

### 3. Code Generation Checklist

Before generating Rails code, the skill verifies:
- Models named after business domain
- Logic in model methods, not controllers
- State management approach matches use case
- Controllers under 10 lines per action
- Complex logic extracted appropriately
- Database normalized with constraints
- Proper scoping for security
- I18n for all user-facing strings

## Key Patterns

### Rich Model Methods

```ruby
# ✅ GOOD: Natural, domain-oriented API
card.close(reason: "Completed")
card.reopen
booking.confirm
recording.incinerate

# ❌ BAD: Service/procedural style
CardClosureService.new(card, user, reason).call
card.set_closed(true)
```

### State as Records (When Audit Trail Matters)

```ruby
# ✅ GOOD: Full audit trail
class Card < ApplicationRecord
  has_one :closure, dependent: :destroy

  def close(reason: nil)
    create_closure!(user: Current.user, reason: reason)
  end
end

# Who closed it? closure.user
# When? closure.created_at
# Why? closure.reason
```

### Concerns with "Has Trait" Semantics

```ruby
# ✅ GOOD: Clear trait
module Card::Closeable
  extend ActiveSupport::Concern

  included do
    has_one :closure, dependent: :destroy
    scope :closed, -> { joins(:closure) }
  end

  def close(reason: nil)
    # Implementation
  end
end
```

### Thin Controllers

```ruby
# ✅ GOOD: 5-10 lines
class CardsController < ApplicationController
  def create
    @card = current_board.cards.create!(card_params)
    redirect_to @card, notice: t(".created")
  end
end
```

### No Service Objects

```ruby
# ❌ BAD: Unnecessary abstraction
class CardClosureService
  def initialize(card, user, reason)
    # ...
  end

  def call
    # ...
  end
end

# ✅ GOOD: Model method
class Card < ApplicationRecord
  def close(reason: nil)
    # Implementation directly in model
  end
end

# ✅ GOOD: Namespaced class for complex logic (15+ lines, external APIs)
class Cloud::CardGenerator
  def initialize(cloud, style: :default)
    @cloud = cloud
    @style = style
  end

  def call
    # Complex generation logic
    # External API calls
  end
end
```

## Technology Stack

### Required

- **Rails** - Latest stable
- **PostgreSQL** - Primary database
- **Hotwire** (Turbo + Stimulus) - Rich UIs without build complexity
- **ViewComponent** - Reusable UI components
- **SolidQueue** - Database-backed jobs
- **ActionPolicy** - Authorization
- **Anyway Config** - Type-safe configuration
- **RSpec + FactoryBot** - Testing
- **Standard** - Ruby linting

### Forbidden

- ❌ Devise (use `has_secure_password`)
- ❌ CanCanCan (use ActionPolicy)
- ❌ Service object gems
- ❌ State machine gems (use enums or state records)
- ❌ ActiveAdmin (build custom admin)

## Installation

### Option 1: Install in Claude Code

```bash
# Clone this repo
git clone <your-repo-url> ~/.claude/plugins/rails-best-practices

# Plugin will be available as a skill
```

### Option 2: Add to Project

```bash
# Add to your Rails project
mkdir -p claude
cp -r path/to/this/plugin/skills/rails-best-practices claude/

# Claude Code will automatically detect it
```

## Usage

### Automatic Activation

The skill activates automatically when you:
- Ask to "generate Rails code"
- Create models, controllers, or migrations
- Discuss Rails conventions or best practices
- Work with ActiveRecord, Hotwire, or RSpec
- Refactor Rails code

### Example Prompts

```
"Create a booking model with confirmation tracking"
→ Generates model with state record for confirmations

"Add a close action to cards"
→ Creates model method, not service object

"Generate a background job for processing uploads"
→ Creates thin job that orchestrates, doesn't execute

"Set up authentication"
→ Uses has_secure_password, not Devise
```

## Customization

### Adjusting Preferences

The plugin is designed to be flexible. To adjust for your team:

1. **Testing Framework** - Edit `references/testing.md` if you prefer Minitest
2. **State Management** - Adjust `references/state-management.md` defaults
3. **Strictness** - Modify `SKILL.md` decision trees

### Adding Your Patterns

Add your team's patterns to `references/` directory:

```bash
# Example: Add your team's deployment patterns
echo "# Deployment Patterns" > skills/rails-best-practices/references/deployment.md
```

Then reference it in `SKILL.md`.

## What Makes This Plugin Different

### vs Pure Evil Martians Approach
- ✅ Adds 37signals' state records pattern (audit trail)
- ✅ Emphasizes "vanilla Rails is plenty" philosophy
- ✅ More flexible on testing framework
- ✅ Stronger anti-service-object stance

### vs Pure 37signals Approach
- ✅ Adds comprehensive reference documentation
- ✅ Includes ActionPolicy for authorization
- ✅ More structured testing patterns (RSpec + FactoryBot)
- ✅ Explicit technology stack guidance

### The Best of Both
- Rich models with natural verb methods (37signals)
- Namespaced classes for complex logic (Evil Martians)
- State records for audit trails (37signals)
- Enums for simple workflows (Evil Martians)
- Form objects as escape hatch, not default (balanced)
- Comprehensive testing standards (Evil Martians)
- One Person Framework philosophy (37signals)

## Contributing

This plugin consolidates patterns from:
- [Evil Martians' AGENTS.md](https://gist.github.com/palkan/482010bd6ec434685f106779e863d0ef) by [@palkan](https://github.com/palkan)
- [37signals' One Person Framework](https://once.com/free-stuff/one-person-framework)
- Rails Best Practices from the community

Found an anti-pattern we missed? Have a better way to explain something? PRs welcome!

## License

MIT License - consolidating open knowledge from the Rails community.

## Credits

- **Evil Martians** - rails-codegen patterns and AGENTS.md
- **@palkan** - Original AGENTS.md standards
- **37signals team** - One Person Framework philosophy
- **Mario Alberto Chávez** - rails-simplifier patterns
- **DHH** - "Vanilla Rails is plenty" philosophy
- **Ruby on Rails community** - Collective wisdom

## Resources

- [Evil Martians' AGENTS.md Gist](https://gist.github.com/palkan/482010bd6ec434685f106779e863d0ef)
- [37signals on Conceptual Compression](https://world.hey.com/dhh/conceptual-compression-means-beginners-don-t-need-to-know-sql-hallelujah-94c81d32)
- [Jorge Manrubia: Vanilla Rails is Plenty](https://dev.37signals.com/vanilla-rails-is-plenty/)
- [Rails Guides](https://guides.rubyonrails.org/)

---

> "The best code is the code you don't write. The second best is the code that's obviously correct."

> "Vanilla Rails is plenty." — Jorge Manrubia, 37signals
