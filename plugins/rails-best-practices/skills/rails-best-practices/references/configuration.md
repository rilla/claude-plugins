# Configuration Patterns

## Anyway Config

Use `anyway_config` for all application configuration. Never use Rails credentials or direct `ENV` access.

### Installation

```ruby
# Gemfile
gem "anyway_config"
```

### Base Configuration Class

```ruby
# config/configs/application_config.rb
class ApplicationConfig < Anyway::Config
  class << self
    # Allow direct class method access: GeminiConfig.api_key
    delegate_missing_to :instance

    def instance
      @instance ||= new
    end
  end
end
```

### Service Configurations

```ruby
# config/configs/gemini_config.rb
class GeminiConfig < ApplicationConfig
  attr_config :api_key,
              :model,
              timeout: 30,
              max_retries: 3

  required :api_key

  # Coercion
  coerce_types model: :string, timeout: :integer
end

# config/configs/stripe_config.rb
class StripeConfig < ApplicationConfig
  attr_config :publishable_key,
              :secret_key,
              :webhook_secret,
              endpoint_secret: nil

  required :publishable_key, :secret_key
end

# config/configs/aws_config.rb
class AwsConfig < ApplicationConfig
  attr_config :access_key_id,
              :secret_access_key,
              :region,
              :bucket

  required :access_key_id, :secret_access_key, :bucket

  # Default region
  def region
    super || "us-east-1"
  end
end
```

### Environment Variable Mapping

Anyway Config automatically maps environment variables:

```ruby
# GeminiConfig.api_key reads from:
# - GEMINI_API_KEY environment variable
# - config/gemini.yml
# - config/credentials/gemini.yml.enc

# Nested configs use underscores:
# StripeConfig.webhook_secret reads from:
# - STRIPE_WEBHOOK_SECRET
```

### YAML Configuration (Optional)

```yaml
# config/gemini.yml
default: &default
  model: gemini-pro
  timeout: 30

development:
  <<: *default
  api_key: <%= ENV['GEMINI_API_KEY'] %>

production:
  <<: *default
  timeout: 60
```

### Usage

```ruby
# Direct access
GeminiConfig.api_key
GeminiConfig.timeout  # => 30 (default)

# In service classes
class Cloud::CardGenerator
  def initialize(cloud)
    @cloud = cloud
    @client = Gemini::Client.new(
      api_key: GeminiConfig.api_key,
      model: GeminiConfig.model,
      timeout: GeminiConfig.timeout
    )
  end
end

# In initializers
# config/initializers/stripe.rb
Stripe.api_key = StripeConfig.secret_key

# config/initializers/aws.rb
Aws.config.update(
  access_key_id: AwsConfig.access_key_id,
  secret_access_key: AwsConfig.secret_access_key,
  region: AwsConfig.region
)
```

### Validation

```ruby
class SmtpConfig < ApplicationConfig
  attr_config :host,
              :port,
              :username,
              :password,
              :authentication

  required :host, :port

  # Custom validation
  on_load :validate_port

  private

  def validate_port
    raise ArgumentError, "Port must be between 1-65535" unless port.between?(1, 65535)
  end
end
```

### Environment-Specific Defaults

```ruby
class AppConfig < ApplicationConfig
  attr_config :host,
              :protocol,
              :asset_host

  def host
    super || default_host
  end

  def protocol
    super || (Rails.env.production? ? "https" : "http")
  end

  private

  def default_host
    case Rails.env
    when "production" then "app.example.com"
    when "staging" then "staging.example.com"
    else "localhost:3000"
    end
  end
end
```

## What NOT to Use

### Rails Credentials

```ruby
# Bad - don't use this
Rails.application.credentials.stripe[:secret_key]
Rails.application.credentials.dig(:aws, :access_key_id)

# Why:
# - Requires master key management
# - Hard to override per environment
# - Encryption adds complexity
# - Not type-safe
```

### Direct ENV Access

```ruby
# Bad - scattered ENV access
class PaymentService
  def initialize
    @api_key = ENV["STRIPE_SECRET_KEY"]
    @timeout = ENV["STRIPE_TIMEOUT"]&.to_i || 30
  end
end

# Why:
# - No validation
# - No defaults management
# - Type coercion scattered
# - Hard to test
# - No documentation of required vars
```

### Config Gems Comparison

| Feature | Anyway Config | Rails Credentials | ENV |
|---------|--------------|-------------------|-----|
| Type coercion | Yes | No | No |
| Defaults | Yes | Limited | No |
| Validation | Yes | No | No |
| Testing | Easy | Complex | Easy |
| Env override | Automatic | Manual | Direct |
| Documentation | In class | None | None |

## Application Settings

### Feature Flags

```ruby
# config/configs/features_config.rb
class FeaturesConfig < ApplicationConfig
  attr_config dark_mode: false,
              new_dashboard: false,
              api_v2: false,
              beta_features: false

  def enabled?(feature)
    public_send(feature)
  rescue NoMethodError
    false
  end
end

# Usage:
if FeaturesConfig.dark_mode
  # render dark mode
end

if FeaturesConfig.enabled?(:new_dashboard)
  # render new dashboard
end
```

### Rate Limits

```ruby
# config/configs/rate_limits_config.rb
class RateLimitsConfig < ApplicationConfig
  attr_config clouds_per_day: 10,
              api_requests_per_minute: 60,
              uploads_per_hour: 20

  coerce_types clouds_per_day: :integer,
               api_requests_per_minute: :integer,
               uploads_per_hour: :integer
end
```

### Email Settings

```ruby
# config/configs/email_config.rb
class EmailConfig < ApplicationConfig
  attr_config :from_address,
              :reply_to,
              :support_email,
              delivery_method: :smtp

  required :from_address

  def from_address
    super || "noreply@#{AppConfig.host}"
  end
end

# config/initializers/action_mailer.rb
Rails.application.configure do
  config.action_mailer.default_options = {
    from: EmailConfig.from_address,
    reply_to: EmailConfig.reply_to
  }
end
```

## Testing Configurations

```ruby
# spec/support/config_helper.rb
module ConfigHelper
  def with_config(config_class, **overrides)
    original_values = overrides.keys.to_h { |k| [k, config_class.send(k)] }

    overrides.each do |key, value|
      allow(config_class).to receive(key).and_return(value)
    end

    yield
  ensure
    original_values.each do |key, value|
      allow(config_class).to receive(key).and_return(value)
    end
  end
end

RSpec.configure do |config|
  config.include ConfigHelper
end

# Usage in tests:
it "uses configured timeout" do
  with_config(GeminiConfig, timeout: 60) do
    expect(GeminiConfig.timeout).to eq(60)
  end
end
```

## Development Setup

### .env Files (Development Only)

```bash
# .env.development (gitignored)
GEMINI_API_KEY=your-dev-key
STRIPE_PUBLISHABLE_KEY=pk_test_xxx
STRIPE_SECRET_KEY=sk_test_xxx

# Load with dotenv gem in development
# Gemfile
group :development, :test do
  gem "dotenv-rails"
end
```

### Required Environment Variables Documentation

```ruby
# config/configs/README.md or in each config class

# Required environment variables:
#
# GEMINI_API_KEY          - Gemini API key for AI features
# STRIPE_PUBLISHABLE_KEY  - Stripe publishable key
# STRIPE_SECRET_KEY       - Stripe secret key
# AWS_ACCESS_KEY_ID       - AWS access key for S3
# AWS_SECRET_ACCESS_KEY   - AWS secret key for S3
# AWS_BUCKET              - S3 bucket name
```

### Validation on Boot

```ruby
# config/initializers/verify_config.rb
Rails.application.config.after_initialize do
  # Trigger loading and validation of required configs
  [GeminiConfig, StripeConfig, AwsConfig].each do |config|
    config.instance  # Loads and validates
  rescue Anyway::Config::ValidationError => e
    raise "Configuration error: #{e.message}"
  end
end
```
