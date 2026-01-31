# Technology Stack & File Structure

## Required Gems

Only use these approved gems:

### Core Framework
- **Rails** - Latest stable version
- **Puma** - Web server
- **Propshaft** - Asset pipeline (not Sprockets)

### Database
- **PostgreSQL** - Primary database
- **ActiveRecord** - ORM (built-in)

### Frontend
- **Hotwire** - Turbo + Stimulus for interactivity
- **ViewComponent** - Reusable view components
- **Vite Rails** - Frontend build tool

### Background Jobs
- **SolidQueue** - Job backend (Rails 8+ default)

### Authentication & Authorization
- **Built-in Rails auth** - has_secure_password, sessions
- **OmniAuth** - OAuth providers when needed
- **ActionPolicy** - Authorization policies

### Configuration
- **Anyway Config** - Type-safe configuration (NOT Rails credentials)

### Testing
- **RSpec** - Test framework (not Minitest)
- **FactoryBot** - Test data factories
- **Faker** - Fake data generation

### Utilities
- **Standard** - Ruby linting/formatting
- **Nanoid** - Short unique IDs
- **FriendlyID** - Human-readable slugs
- **HTTParty** - HTTP client for APIs

## Forbidden Gems & Patterns

**Never use these:**

| Forbidden | Why | Alternative |
|-----------|-----|-------------|
| Devise | Over-engineered | Built-in has_secure_password |
| CanCanCan | Outdated patterns | ActionPolicy |
| ActiveAdmin | Heavy, inflexible | Custom admin with Hotwire |
| Pundit | Less flexible | ActionPolicy |
| Service object gems | Unnecessary abstraction | Namespaced model classes |
| AASM/state_machine | External dependency | Built-in enums |
| Virtus/dry-types/Literal | Over-typed | Plain Ruby, ActiveModel |
| Interactor/Trailblazer | Context objects | Jobs + model methods |

## File Structure

```
app/
├── models/                    # ActiveRecord models + namespaced classes
│   ├── application_record.rb
│   ├── participant.rb
│   ├── cloud.rb
│   └── cloud/                 # Namespaced operations
│       ├── card_generator.rb
│       └── image_processor.rb
├── controllers/
│   ├── application_controller.rb
│   ├── clouds_controller.rb   # Public endpoints
│   └── participant/           # Auth-scoped namespace
│       ├── application_controller.rb
│       └── clouds_controller.rb
├── jobs/
│   ├── application_job.rb
│   └── cloud_processing_job.rb
├── forms/                     # Multi-model operations only
│   ├── application_form.rb
│   └── participant_registration_form.rb
├── policies/                  # ActionPolicy policies
│   ├── application_policy.rb
│   └── cloud_policy.rb
├── views/
│   ├── layouts/
│   ├── clouds/
│   └── components/            # ViewComponent classes
├── frontend/                  # Vite-managed assets
│   ├── entrypoints/
│   ├── controllers/           # Stimulus controllers
│   └── stylesheets/
└── mailers/

config/
├── configs/                   # Anyway Config classes
│   ├── application_config.rb
│   ├── gemini_config.rb
│   └── stripe_config.rb
└── ...

spec/
├── models/
├── requests/
├── system/
├── factories/
└── support/
```

## Critical Rules

### No Service Directories

**Never create:**
- `app/services/`
- `app/contexts/`
- `app/operations/`
- `app/interactors/`
- `app/use_cases/`

**Instead:**
- Simple operations → Model methods
- Complex operations → Namespaced model classes (`Cloud::CardGenerator`)
- Multi-model transactions → Form objects (`app/forms/`)
- Async operations → Jobs (`app/jobs/`)

### Namespaced Model Classes

When model logic grows complex or calls external APIs:

```ruby
# app/models/cloud/card_generator.rb
class Cloud::CardGenerator
  def initialize(cloud)
    @cloud = cloud
  end

  def call
    # Complex generation logic
    # External API calls
    # Returns generated card
  end
end

# Usage in job:
Cloud::CardGenerator.new(cloud).call
```

### Controller Namespacing

Namespace controllers by authentication context:

```ruby
# app/controllers/participant/application_controller.rb
class Participant::ApplicationController < ApplicationController
  before_action :require_participant

  private

  def current_participant
    @current_participant ||= Participant.find_by(access_token: params[:access_token])
  end

  def require_participant
    head :unauthorized unless current_participant
  end
end

# app/controllers/participant/clouds_controller.rb
class Participant::CloudsController < Participant::ApplicationController
  def index
    @clouds = current_participant.clouds
  end
end
```

### Configuration via Anyway Config

```ruby
# config/configs/application_config.rb
class ApplicationConfig < Anyway::Config
  class << self
    delegate_missing_to :instance

    def instance
      @instance ||= new
    end
  end
end

# config/configs/gemini_config.rb
class GeminiConfig < ApplicationConfig
  attr_config :api_key,
              :model,
              timeout: 30

  required :api_key
end

# Usage:
GeminiConfig.api_key  # reads from GEMINI_API_KEY env var
GeminiConfig.timeout  # defaults to 30
```

**Never use:**
- `Rails.application.credentials`
- Direct `ENV["VAR"]` access
- YAML configuration files for runtime config
