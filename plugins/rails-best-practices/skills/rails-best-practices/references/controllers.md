# Controller Patterns

## Target: 5-10 Lines Per Action

Controllers should be thin orchestrators. They handle HTTP concerns only:
- Authentication/authorization checks
- Parameter handling
- Model operations (simple CRUD)
- Job enqueueing
- Response rendering

## Basic Pattern

```ruby
class CloudsController < ApplicationController
  def index
    @clouds = Cloud.recent.limit(20)
  end

  def show
    @cloud = Cloud.find(params[:id])
  end

  def create
    @cloud = Cloud.create!(cloud_params)
    redirect_to @cloud, notice: "Cloud created"
  rescue ActiveRecord::RecordInvalid => e
    @cloud = e.record
    render :new, status: :unprocessable_entity
  end

  def destroy
    @cloud = Cloud.find(params[:id])
    @cloud.destroy!
    redirect_to clouds_path, notice: "Cloud deleted"
  end

  private

  def cloud_params
    params.require(:cloud).permit(:title, :description, :source_image)
  end
end
```

## Guard Clauses

Use guard clauses for early returns:

```ruby
class Participant::CloudsController < Participant::ApplicationController
  def create
    # Guard clause - check permission first
    return head :forbidden unless current_participant.can_create_clouds?

    # Guard clause - check resource limits
    return head :too_many_requests if current_participant.clouds.count >= 100

    @cloud = current_participant.clouds.create!(cloud_params)
    CloudProcessingJob.perform_later(@cloud)
    redirect_to [:participant, @cloud]
  end

  def update
    @cloud = current_participant.clouds.find(params[:id])

    # Guard clause - check state
    return head :conflict if @cloud.processing?

    @cloud.update!(cloud_params)
    redirect_to [:participant, @cloud]
  end
end
```

## Namespacing for Authentication

Namespace controllers by authentication context:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  # Shared behavior for all controllers
end

# app/controllers/participant/application_controller.rb
class Participant::ApplicationController < ApplicationController
  before_action :require_participant

  private

  def current_participant
    @current_participant ||= Participant.find_by(access_token: session[:access_token])
  end
  helper_method :current_participant

  def require_participant
    redirect_to login_path unless current_participant
  end
end

# app/controllers/participant/clouds_controller.rb
class Participant::CloudsController < Participant::ApplicationController
  def index
    # current_participant is guaranteed to exist
    @clouds = current_participant.clouds.recent
  end

  def show
    # Scoped to current participant automatically
    @cloud = current_participant.clouds.find(params[:id])
  end
end

# app/controllers/admin/application_controller.rb
class Admin::ApplicationController < ApplicationController
  before_action :require_admin

  private

  def require_admin
    head :forbidden unless current_user&.admin?
  end
end
```

## Routes with Namespacing

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # Public routes
  resources :clouds, only: [:index, :show]

  # Participant-scoped routes (using access token in URL)
  scope "/p/:access_token", as: :participant, module: :participant do
    resources :clouds
    resource :profile, only: [:show, :edit, :update]
  end

  # Session-based participant routes
  namespace :participant do
    resources :clouds
    resource :settings
  end

  # Admin routes
  namespace :admin do
    resources :participants
    resources :clouds
  end
end
```

## Parameter Handling

### Strong Parameters

```ruby
class CloudsController < ApplicationController
  private

  def cloud_params
    params.require(:cloud).permit(
      :title,
      :description,
      :source_image,
      :style,
      metadata: [:width, :height, :format]
    )
  end
end
```

### Nested Attributes

```ruby
class ParticipantsController < ApplicationController
  private

  def participant_params
    params.require(:participant).permit(
      :name,
      :email,
      profile_attributes: [:bio, :avatar, :website]
    )
  end
end
```

## Response Patterns

### HTML Responses

```ruby
def create
  @cloud = current_participant.clouds.create!(cloud_params)
  redirect_to @cloud, notice: "Created successfully"
rescue ActiveRecord::RecordInvalid => e
  @cloud = e.record
  render :new, status: :unprocessable_entity
end
```

### Turbo Stream Responses

```ruby
def create
  @cloud = current_participant.clouds.create!(cloud_params)

  respond_to do |format|
    format.turbo_stream
    format.html { redirect_to @cloud }
  end
rescue ActiveRecord::RecordInvalid => e
  @cloud = e.record
  render :new, status: :unprocessable_entity
end
```

### JSON API Responses

```ruby
def create
  @cloud = current_participant.clouds.create!(cloud_params)
  render json: @cloud, status: :created
rescue ActiveRecord::RecordInvalid => e
  render json: { errors: e.record.errors }, status: :unprocessable_entity
end
```

## Anti-Patterns to Avoid

### Business Logic in Controllers

```ruby
# Bad - too much logic
def create
  @cloud = Cloud.new(cloud_params)
  @cloud.participant = current_participant
  @cloud.state = :pending

  if @cloud.source_image.attached?
    @cloud.metadata = extract_metadata(@cloud.source_image)
    @cloud.thumbnail = generate_thumbnail(@cloud.source_image)
  end

  if @cloud.save
    notify_participant(@cloud)
    update_participant_stats
    redirect_to @cloud
  else
    render :new
  end
end

# Good - delegate to model/job
def create
  @cloud = current_participant.clouds.create!(cloud_params)
  CloudProcessingJob.perform_later(@cloud)
  redirect_to @cloud
rescue ActiveRecord::RecordInvalid => e
  @cloud = e.record
  render :new, status: :unprocessable_entity
end
```

### Concerns for Business Logic

```ruby
# Bad - concerns hide complexity
class CloudsController < ApplicationController
  include CloudProcessing
  include ParticipantStats
  include Notifications
end

# Good - explicit inheritance, clear responsibilities
class Participant::CloudsController < Participant::ApplicationController
  # Only HTTP concerns here
end
```

### Fat Before Actions

```ruby
# Bad - too many before_actions
class CloudsController < ApplicationController
  before_action :set_cloud
  before_action :authorize_cloud
  before_action :check_processing_state
  before_action :load_related_data
  before_action :set_breadcrumbs
end

# Good - minimal setup, explicit in actions
class CloudsController < ApplicationController
  def show
    @cloud = Cloud.find(params[:id])
    authorize @cloud  # ActionPolicy
  end
end
```

## Error Handling

```ruby
class ApplicationController < ActionController::Base
  rescue_from ActiveRecord::RecordNotFound, with: :not_found
  rescue_from ActionPolicy::Unauthorized, with: :forbidden

  private

  def not_found
    render file: Rails.public_path.join("404.html"), status: :not_found, layout: false
  end

  def forbidden
    head :forbidden
  end
end
```

## Testing Controllers

Test through request specs, not controller specs:

```ruby
# spec/requests/participant/clouds_spec.rb
RSpec.describe "Participant Clouds" do
  let(:participant) { create(:participant) }

  describe "POST /participant/clouds" do
    it "creates a cloud" do
      post participant_clouds_path,
           params: { cloud: { title: "My Cloud" } },
           headers: auth_headers(participant)

      expect(response).to redirect_to(participant_cloud_path(Cloud.last))
      expect(Cloud.last.title).to eq("My Cloud")
    end

    it "rejects invalid clouds" do
      post participant_clouds_path,
           params: { cloud: { title: "" } },
           headers: auth_headers(participant)

      expect(response).to have_http_status(:unprocessable_entity)
    end
  end
end
```
