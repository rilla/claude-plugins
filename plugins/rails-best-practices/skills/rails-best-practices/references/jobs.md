# Background Job Patterns

## Job Philosophy

Jobs should **orchestrate, not execute**. Complex logic belongs in namespaced model classes.

```ruby
# Good - job orchestrates
class CloudProcessingJob < ApplicationJob
  def perform(cloud)
    cloud.analyzing!
    Cloud::Analyzer.new(cloud).call
    cloud.analyzed!

    cloud.generating!
    Cloud::CardGenerator.new(cloud).call
    cloud.generated!
  rescue => e
    cloud.failed!
    Rails.error.report(e, handled: true)
  end
end

# Bad - job executes everything
class CloudProcessingJob < ApplicationJob
  def perform(cloud)
    # 100 lines of processing logic
    # API calls
    # Image manipulation
    # etc.
  end
end
```

## Basic Job Structure

```ruby
# app/jobs/application_job.rb
class ApplicationJob < ActiveJob::Base
  # Retry on transient failures
  retry_on ActiveRecord::Deadlocked, wait: 5.seconds, attempts: 3
  retry_on Net::OpenTimeout, wait: :polynomially_longer, attempts: 5

  # Don't retry on permanent failures
  discard_on ActiveJob::DeserializationError
end

# app/jobs/cloud_processing_job.rb
class CloudProcessingJob < ApplicationJob
  queue_as :default

  def perform(cloud)
    return if cloud.generated? || cloud.failed?

    process_cloud(cloud)
  rescue StandardError => e
    handle_failure(cloud, e)
    raise  # Re-raise to trigger retry
  end

  private

  def process_cloud(cloud)
    cloud.analyzing!
    Cloud::Analyzer.new(cloud).call
    cloud.analyzed!
  end

  def handle_failure(cloud, error)
    cloud.update!(processing_error: error.message)
    Rails.error.report(error, handled: true, context: { cloud_id: cloud.id })
  end
end
```

## ActiveJob::Continuable for Workflows

For multi-step workflows that can be resumed:

```ruby
class CloudProcessingJob < ApplicationJob
  include ActiveJob::Continuable

  # Define workflow steps
  step :moderate, isolated: true
  step :analyze
  step :generate, unless: -> { cloud.failed? }
  step :notify

  def perform(cloud)
    @cloud = cloud
    run_steps
  end

  private

  attr_reader :cloud

  def moderate
    Cloud::Moderator.new(cloud).call
    cloud.fail! if cloud.flagged?
  end

  def analyze
    cloud.analyzing!
    Cloud::Analyzer.new(cloud).call
    cloud.analyzed!
  end

  def generate
    cloud.generating!
    Cloud::CardGenerator.new(cloud).call
    cloud.generated!
  end

  def notify
    CloudMailer.processing_complete(cloud).deliver_later
  end
end
```

### Step Options

```ruby
# isolated: true - step runs in separate transaction
step :external_api_call, isolated: true

# unless/if - conditional execution
step :generate, unless: -> { cloud.failed? }
step :premium_features, if: -> { cloud.participant.premium? }

# on_error - custom error handling
step :risky_operation, on_error: :handle_risky_error

def handle_risky_error(error)
  Rails.error.report(error, handled: true)
  # Continue to next step instead of failing
end
```

## Queue Configuration

### Queue Priority

```ruby
class CloudProcessingJob < ApplicationJob
  queue_as :default
end

class ReportGenerationJob < ApplicationJob
  queue_as :low
end

class PaymentProcessingJob < ApplicationJob
  queue_as :critical
end
```

### SolidQueue Configuration

```yaml
# config/solid_queue.yml
default: &default
  dispatchers:
    - polling_interval: 1
      batch_size: 500
  workers:
    - queues: [critical]
      threads: 3
      processes: 1
    - queues: [default, low]
      threads: 5
      processes: 2

development:
  <<: *default

production:
  <<: *default
```

## Error Handling

### Retry Strategies

```ruby
class ExternalApiJob < ApplicationJob
  # Exponential backoff
  retry_on Net::OpenTimeout,
           wait: :polynomially_longer,
           attempts: 5

  # Fixed delay
  retry_on RateLimitError,
           wait: 1.minute,
           attempts: 3

  # Custom wait calculation
  retry_on ServiceUnavailable,
           wait: ->(executions) { executions * 30.seconds },
           attempts: 5
end
```

### Discard vs Retry

```ruby
class ImportJob < ApplicationJob
  # Retry transient errors
  retry_on ActiveRecord::Deadlocked
  retry_on Redis::ConnectionError

  # Discard permanent failures
  discard_on ActiveJob::DeserializationError  # Record deleted
  discard_on InvalidDataError                  # Bad input, won't fix itself
end
```

### Error Reporting

```ruby
class CloudProcessingJob < ApplicationJob
  def perform(cloud)
    process(cloud)
  rescue StandardError => e
    # Report but continue (handled error)
    Rails.error.report(e, handled: true, context: {
      cloud_id: cloud.id,
      participant_id: cloud.participant_id
    })

    # Update record state
    cloud.update!(
      state: :failed,
      processing_error: e.message
    )

    # Don't re-raise - job is "successful" (we handled it)
  end
end
```

## Job Patterns

### Idempotency

Jobs should be safe to run multiple times:

```ruby
class CloudProcessingJob < ApplicationJob
  def perform(cloud)
    # Guard clause - already processed
    return if cloud.generated?

    # Guard clause - already in progress (optional)
    return if cloud.processing? && cloud.updated_at > 5.minutes.ago

    process(cloud)
  end
end
```

### Batch Processing

```ruby
class BulkExportJob < ApplicationJob
  def perform(participant)
    participant.clouds.find_each(batch_size: 100) do |cloud|
      ExportCloudJob.perform_later(cloud)
    end
  end
end
```

### Job Chaining

```ruby
class CloudWorkflowJob < ApplicationJob
  def perform(cloud)
    # Process sequentially
    Cloud::Analyzer.new(cloud).call
    Cloud::CardGenerator.new(cloud).call

    # Enqueue next job
    CloudNotificationJob.perform_later(cloud)
  end
end
```

### Scheduled Jobs

```ruby
class DailyReportJob < ApplicationJob
  def perform
    Report::DailyGenerator.new(Date.yesterday).call
  end
end

# Schedule with SolidQueue
# config/recurring.yml
daily_report:
  class: DailyReportJob
  schedule: "0 6 * * *"  # 6 AM daily
```

## Testing Jobs

```ruby
# spec/jobs/cloud_processing_job_spec.rb
RSpec.describe CloudProcessingJob do
  let(:cloud) { create(:cloud, state: :uploaded) }

  describe "#perform" do
    it "processes the cloud" do
      expect { described_class.perform_now(cloud) }
        .to change { cloud.reload.state }.from("uploaded").to("generated")
    end

    it "handles failures gracefully" do
      allow(Cloud::Analyzer).to receive(:new).and_raise(StandardError)

      expect { described_class.perform_now(cloud) }.not_to raise_error
      expect(cloud.reload.state).to eq("failed")
    end
  end
end

# Test job is enqueued
it "enqueues processing job" do
  expect { create(:cloud) }
    .to have_enqueued_job(CloudProcessingJob)
end
```

## Anti-Patterns

### Long-Running Synchronous Work

```ruby
# Bad - blocks for too long
class ReportJob < ApplicationJob
  def perform
    Report.all.each do |report|
      generate_pdf(report)  # 30 seconds each
      send_email(report)
    end
  end
end

# Good - fan out to individual jobs
class ReportBatchJob < ApplicationJob
  def perform
    Report.pending.find_each do |report|
      ReportGenerationJob.perform_later(report)
    end
  end
end
```

### Missing Idempotency

```ruby
# Bad - sends duplicate emails on retry
class WelcomeJob < ApplicationJob
  def perform(user)
    UserMailer.welcome(user).deliver_now
    user.update!(welcomed_at: Time.current)
  end
end

# Good - check before sending
class WelcomeJob < ApplicationJob
  def perform(user)
    return if user.welcomed_at.present?

    UserMailer.welcome(user).deliver_now
    user.update!(welcomed_at: Time.current)
  end
end
```

### Business Logic in Jobs

```ruby
# Bad - job contains business logic
class ProcessOrderJob < ApplicationJob
  def perform(order)
    order.calculate_total
    order.apply_discounts
    order.charge_payment
    order.update_inventory
    order.send_confirmation
    # 50 more lines...
  end
end

# Good - job orchestrates
class ProcessOrderJob < ApplicationJob
  def perform(order)
    Order::Processor.new(order).call
  end
end
```
