# Views & Frontend Patterns

## Technology Stack

- **Hotwire** - Turbo (page updates) + Stimulus (JavaScript sprinkles)
- **ViewComponent** - Reusable, testable view components
- **Vite Rails** - Modern frontend build tool
- **ERB** - Template engine (not Haml/Slim)

## Hotwire Patterns

### Turbo Drive

Automatic page navigation without full reloads:

```erb
<%# Links work automatically %>
<%= link_to "View Cloud", cloud_path(@cloud) %>

<%# Disable for specific links %>
<%= link_to "External", "https://example.com", data: { turbo: false } %>
```

### Turbo Frames

Partial page updates:

```erb
<%# app/views/clouds/show.html.erb %>
<%= turbo_frame_tag @cloud do %>
  <h1><%= @cloud.title %></h1>
  <%= link_to "Edit", edit_cloud_path(@cloud) %>
<% end %>

<%# app/views/clouds/edit.html.erb %>
<%= turbo_frame_tag @cloud do %>
  <%= form_with model: @cloud do |f| %>
    <%= f.text_field :title %>
    <%= f.submit "Save" %>
  <% end %>
<% end %>
```

### Turbo Streams

Real-time updates:

```erb
<%# app/views/clouds/create.turbo_stream.erb %>
<%= turbo_stream.prepend "clouds", @cloud %>
<%= turbo_stream.update "clouds_count", Cloud.count %>

<%# app/views/clouds/destroy.turbo_stream.erb %>
<%= turbo_stream.remove @cloud %>
```

Broadcasting from models:

```ruby
class Cloud < ApplicationRecord
  after_create_commit -> { broadcast_prepend_to "clouds" }
  after_update_commit -> { broadcast_replace_to "clouds" }
  after_destroy_commit -> { broadcast_remove_to "clouds" }
end
```

### Turbo Stream Actions

```erb
<%# Append/Prepend %>
<%= turbo_stream.append "clouds", @cloud %>
<%= turbo_stream.prepend "clouds", @cloud %>

<%# Replace/Update %>
<%= turbo_stream.replace @cloud %>
<%= turbo_stream.update "flash", partial: "shared/flash" %>

<%# Remove %>
<%= turbo_stream.remove @cloud %>

<%# Custom action %>
<%= turbo_stream.action :highlight, "cloud_#{@cloud.id}" %>
```

## Stimulus Controllers

### Basic Controller

```javascript
// app/frontend/controllers/clipboard_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["source"]
  static values = { successMessage: String }

  copy() {
    navigator.clipboard.writeText(this.sourceTarget.value)
    this.showFeedback()
  }

  showFeedback() {
    const message = this.successMessageValue || "Copied!"
    // Show toast notification
  }
}
```

```erb
<div data-controller="clipboard" data-clipboard-success-message-value="Link copied!">
  <input data-clipboard-target="source" value="<%= @cloud.share_url %>" readonly>
  <button data-action="clipboard#copy">Copy Link</button>
</div>
```

### Common Patterns

```javascript
// Toggle visibility
export default class extends Controller {
  static targets = ["content"]

  toggle() {
    this.contentTarget.classList.toggle("hidden")
  }
}

// Form submission
export default class extends Controller {
  static targets = ["submit"]

  disable() {
    this.submitTarget.disabled = true
    this.submitTarget.textContent = "Saving..."
  }

  enable() {
    this.submitTarget.disabled = false
    this.submitTarget.textContent = "Save"
  }
}

// Auto-submit on change
export default class extends Controller {
  submit() {
    this.element.requestSubmit()
  }
}
```

### Controller Communication

```javascript
// Dispatch custom events
this.dispatch("selected", { detail: { id: this.idValue } })

// Listen in another controller
static targets = ["output"]

handleSelected({ detail: { id } }) {
  this.outputTarget.textContent = id
}
```

```erb
<div data-controller="parent"
     data-action="child:selected->parent#handleSelected">
  <div data-controller="child" data-child-id-value="123">
    <button data-action="child#select">Select</button>
  </div>
  <span data-parent-target="output"></span>
</div>
```

## ViewComponent

### Basic Component

```ruby
# app/views/components/card_component.rb
class CardComponent < ViewComponent::Base
  def initialize(cloud:)
    @cloud = cloud
  end

  private

  attr_reader :cloud
end
```

```erb
<%# app/views/components/card_component.html.erb %>
<div class="card">
  <h3><%= cloud.title %></h3>
  <p><%= cloud.description %></p>
  <%= link_to "View", cloud_path(cloud) %>
</div>
```

Usage:

```erb
<%= render CardComponent.new(cloud: @cloud) %>

<%# Collection %>
<%= render CardComponent.with_collection(@clouds, as: :cloud) %>
```

### Component with Slots

```ruby
# app/views/components/modal_component.rb
class ModalComponent < ViewComponent::Base
  renders_one :header
  renders_one :body
  renders_one :footer

  def initialize(size: :medium)
    @size = size
  end
end
```

```erb
<%# app/views/components/modal_component.html.erb %>
<div class="modal modal--<%= @size %>">
  <% if header? %>
    <div class="modal__header"><%= header %></div>
  <% end %>
  <div class="modal__body"><%= body %></div>
  <% if footer? %>
    <div class="modal__footer"><%= footer %></div>
  <% end %>
</div>
```

Usage:

```erb
<%= render ModalComponent.new(size: :large) do |modal| %>
  <% modal.with_header do %>
    <h2>Confirm Delete</h2>
  <% end %>
  <% modal.with_body do %>
    <p>Are you sure you want to delete this cloud?</p>
  <% end %>
  <% modal.with_footer do %>
    <%= button_to "Delete", cloud_path(@cloud), method: :delete %>
  <% end %>
<% end %>
```

### Testing Components

```ruby
# spec/components/card_component_spec.rb
RSpec.describe CardComponent, type: :component do
  let(:cloud) { create(:cloud, title: "Test Cloud") }

  it "renders the title" do
    render_inline(described_class.new(cloud: cloud))

    expect(page).to have_text("Test Cloud")
  end

  it "links to the cloud" do
    render_inline(described_class.new(cloud: cloud))

    expect(page).to have_link("View", href: cloud_path(cloud))
  end
end
```

## View Best Practices

### Keep Views Simple

```erb
<%# Good - simple logic %>
<% @clouds.each do |cloud| %>
  <%= render CardComponent.new(cloud: cloud) %>
<% end %>

<%# Bad - complex logic in view %>
<% @clouds.select { |c| c.visible? && c.participant.active? }.each do |cloud| %>
  <% if cloud.cards.any? && cloud.cards.first.image.attached? %>
    ...
  <% end %>
<% end %>
```

### Use Scopes and Associations

```ruby
# In model
scope :visible, -> { where(visible: true).joins(:participant).merge(Participant.active) }

# In controller
@clouds = current_participant.clouds.visible.includes(:cards)

# In view - simple iteration
<% @clouds.each do |cloud| %>
```

### Avoid N+1 Queries

```ruby
# Controller
@clouds = Cloud.includes(:participant, :cards).recent

# View - no additional queries
<% @clouds.each do |cloud| %>
  <p>By: <%= cloud.participant.name %></p>
  <p>Cards: <%= cloud.cards.size %></p>
<% end %>
```

### Partials vs Components

Use **partials** for:
- Simple, one-off templates
- Layouts and shared structure
- Turbo Stream responses

Use **ViewComponent** for:
- Reusable UI elements
- Complex components with logic
- Components that need testing
- Components with slots/variants

## Form Patterns

### Basic Form

```erb
<%= form_with model: @cloud do |f| %>
  <div class="field">
    <%= f.label :title %>
    <%= f.text_field :title %>
    <%= render ErrorComponent.new(errors: @cloud.errors[:title]) %>
  </div>

  <div class="field">
    <%= f.label :description %>
    <%= f.text_area :description %>
  </div>

  <%= f.submit %>
<% end %>
```

### Form with Turbo

```erb
<%# Inline form updates %>
<%= form_with model: @cloud, data: { turbo_frame: "_top" } do |f| %>
  ...
<% end %>

<%# Disable Turbo for file uploads %>
<%= form_with model: @cloud, data: { turbo: false } do |f| %>
  <%= f.file_field :image %>
<% end %>
```

### Nested Forms

```erb
<%= form_with model: @participant do |f| %>
  <%= f.text_field :name %>

  <%= f.fields_for :profile do |pf| %>
    <%= pf.text_field :bio %>
    <%= pf.text_field :website %>
  <% end %>

  <%= f.submit %>
<% end %>
```

## Asset Organization

```
app/frontend/
├── entrypoints/
│   └── application.js    # Main entry point
├── controllers/          # Stimulus controllers
│   ├── index.js
│   ├── clipboard_controller.js
│   └── modal_controller.js
├── stylesheets/
│   ├── application.css
│   ├── components/
│   └── utilities/
└── images/
```

### Stimulus Controller Registration

```javascript
// app/frontend/controllers/index.js
import { application } from "./application"
import ClipboardController from "./clipboard_controller"
import ModalController from "./modal_controller"

application.register("clipboard", ClipboardController)
application.register("modal", ModalController)
```
