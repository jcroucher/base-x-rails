# Phase 5: Developer Experience & App Setup

## Step 1: Gem Additions

Add to Gemfile:

```ruby
# Development email preview
gem "letter_opener_web", "~> 3.0", group: :development

# Background job dashboard
gem "mission_control-jobs"

# Soft deletes
gem "discard", "~> 1.4"
```

Run `bundle install`.

## Step 2: Development Email Configuration

Update `config/environments/development.rb` to capture emails in the browser instead of sending them:

```ruby
# config/environments/development.rb
Rails.application.configure do
  # ... existing config ...

  # Use letter_opener_web to capture emails in the browser
  config.action_mailer.delivery_method = :letter_opener_web
  config.action_mailer.default_url_options = { host: "localhost", port: 3000 }
  config.action_mailer.perform_deliveries = true
end
```

Emails sent in development will now appear at `/letter_opener` instead of being delivered. This is critical for Devise's `:confirmable` module — without it, new users can't verify their email and can't log in.

## Step 3: Fix Seeds

Update `db/seeds.rb` to set `confirmed_at` on the admin user so they can log in immediately in development:

```ruby
# db/seeds.rb
puts "Creating plans..."
Plan.find_or_create_by!(name: "Free") do |plan|
  plan.stripe_id = "free"
  plan.price_monthly = 0
  plan.price_yearly = 0
  plan.trial_days = 0
  plan.features = { "max_members" => 5 }
  plan.active = true
  plan.sort_order = 0
end

Plan.find_or_create_by!(name: "Pro") do |plan|
  plan.stripe_id = "pro_monthly"
  plan.price_monthly = 2900
  plan.price_yearly = 29000
  plan.trial_days = 14
  plan.features = { "max_members" => 25, "priority_support" => true }
  plan.active = true
  plan.sort_order = 1
end

Plan.find_or_create_by!(name: "Enterprise") do |plan|
  plan.stripe_id = "enterprise_monthly"
  plan.price_monthly = 9900
  plan.price_yearly = 99000
  plan.trial_days = 14
  plan.features = { "max_members" => -1, "priority_support" => true, "sso" => true }
  plan.active = true
  plan.sort_order = 2
end

if Rails.env.development?
  puts "Creating development user..."
  user = User.find_or_create_by!(email: "admin@example.com") do |u|
    u.first_name = "Admin"
    u.last_name = "User"
    u.password = "password123"
    u.admin = true
    u.confirmed_at = Time.current
  end

  # Fix existing unconfirmed admin users (re-running seeds after Phase 2)
  if user.confirmed_at.nil?
    user.update!(confirmed_at: Time.current)
  end

  puts "Development login: admin@example.com / password123"
end
```

## Step 4: First-Run Setup Wizard

When a fresh app is deployed, there are no users yet. The setup wizard lets you create the first admin account without needing seeds, email confirmation, or console access.

### Setup Guard Concern

Create `app/controllers/concerns/setup_guard.rb`:

```ruby
module SetupGuard
  extend ActiveSupport::Concern

  included do
    before_action :redirect_to_setup_if_needed
  end

  private

  def redirect_to_setup_if_needed
    return if controller_name == "setup"
    return if self.class.name.start_with?("Devise::")
    return if request.path.start_with?("/rails/")

    unless User.where(admin: true).where.not(confirmed_at: nil).exists?
      redirect_to setup_path
    end
  end
end
```

Include the concern in `ApplicationController`:

```ruby
class ApplicationController < ActionController::Base
  include SetCurrentTeam
  include Authentication
  include Authorization
  include SetupGuard
end
```

### Setup Controller

Create `app/controllers/setup_controller.rb`:

```ruby
class SetupController < ApplicationController
  layout "marketing"

  # Skip all auth — this runs before any user exists
  skip_before_action :authenticate_user!, raise: false
  skip_before_action :set_current_team, raise: false
  skip_before_action :redirect_to_setup_if_needed

  before_action :require_no_admin!

  def show
    @user = User.new
  end

  def create
    @user = User.new(setup_params)
    @user.admin = true
    @user.confirmed_at = Time.current

    if @user.save
      sign_in(@user)
      redirect_to dashboard_path, notice: "Welcome! Your admin account has been created."
    else
      render :show, status: :unprocessable_entity
    end
  end

  private

  def setup_params
    params.require(:user).permit(:first_name, :last_name, :email, :password, :password_confirmation)
  end

  def require_no_admin!
    if User.where(admin: true).where.not(confirmed_at: nil).exists?
      redirect_to root_path, alert: "Setup has already been completed."
    end
  end
end
```

### Setup View

Create `app/views/setup/show.html.erb`:

```erb
<% content_for(:title, "Setup — #{BaseX.config.app_name}") %>

<div class="flex min-h-full items-center justify-center py-12 px-4 sm:px-6 lg:px-8">
  <div class="w-full max-w-md space-y-8">
    <div class="text-center">
      <h1 class="text-3xl font-bold tracking-tight text-gray-900">
        Welcome to <%= BaseX.config.app_name %>
      </h1>
      <p class="mt-2 text-sm text-gray-600">
        Create your admin account to get started.
      </p>
    </div>

    <%= form_with(model: @user, url: setup_path, class: "mt-8 space-y-6") do |f| %>
      <% if @user.errors.any? %>
        <div class="rounded-md bg-red-50 border border-red-200 p-4">
          <ul class="list-disc list-inside text-sm text-red-700">
            <% @user.errors.full_messages.each do |message| %>
              <li><%= message %></li>
            <% end %>
          </ul>
        </div>
      <% end %>

      <div class="grid grid-cols-2 gap-4">
        <div>
          <%= f.label :first_name, class: "block text-sm font-medium text-gray-700" %>
          <%= f.text_field :first_name, required: true,
              class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500 sm:text-sm",
              placeholder: "Jane" %>
        </div>
        <div>
          <%= f.label :last_name, class: "block text-sm font-medium text-gray-700" %>
          <%= f.text_field :last_name, required: true,
              class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500 sm:text-sm",
              placeholder: "Smith" %>
        </div>
      </div>

      <div>
        <%= f.label :email, class: "block text-sm font-medium text-gray-700" %>
        <%= f.email_field :email, required: true, autocomplete: "email",
            class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500 sm:text-sm",
            placeholder: "you@example.com" %>
      </div>

      <div>
        <%= f.label :password, class: "block text-sm font-medium text-gray-700" %>
        <%= f.password_field :password, required: true, autocomplete: "new-password",
            class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500 sm:text-sm",
            placeholder: "Minimum 6 characters" %>
      </div>

      <div>
        <%= f.label :password_confirmation, class: "block text-sm font-medium text-gray-700" %>
        <%= f.password_field :password_confirmation, required: true, autocomplete: "new-password",
            class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500 sm:text-sm" %>
      </div>

      <%= f.submit "Create Admin Account",
          class: "w-full flex justify-center py-2.5 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white bg-primary-600 hover:bg-primary-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-primary-500 cursor-pointer" %>
    <% end %>
  </div>
</div>
```

## Step 5: Soft Deletes with Discard

Instead of permanently destroying records, soft-delete critical models so data can be recovered.

### Migrations

Generate a migration for each model:

```bash
rails generate migration AddDiscardedAtToUsers discarded_at:datetime:index
rails generate migration AddDiscardedAtToTeams discarded_at:datetime:index
rails generate migration AddDiscardedAtToMemberships discarded_at:datetime:index
rails generate migration AddDiscardedAtToInvitations discarded_at:datetime:index
rails generate migration AddDiscardedAtToApiTokens discarded_at:datetime:index
```

Each migration will look like:

```ruby
class AddDiscardedAtToUsers < ActiveRecord::Migration[8.1]
  def change
    add_column :users, :discarded_at, :datetime
    add_index :users, :discarded_at
  end
end
```

Repeat the same pattern for Teams, Memberships, Invitations, and ApiTokens.

Run `rails db:migrate`.

### SoftDeletable Concern

Create `app/models/concerns/soft_deletable.rb`:

```ruby
module SoftDeletable
  extend ActiveSupport::Concern

  included do
    include Discard::Model

    default_scope -> { kept }
  end

  # Aliases for a more natural API
  def soft_delete!
    discard!
  end

  def restore!
    undiscard!
  end

  def soft_deleted?
    discarded?
  end
end
```

### Include in Models

Add `include SoftDeletable` to each model:

```ruby
# app/models/user.rb
class User < ApplicationRecord
  include SoftDeletable
  # ... existing code ...
end

# app/models/team.rb
class Team < ApplicationRecord
  include SoftDeletable
  # ... existing code ...
end

# app/models/membership.rb
class Membership < ApplicationRecord
  include SoftDeletable
  # ... existing code ...
end

# app/models/invitation.rb
class Invitation < ApplicationRecord
  include SoftDeletable
  # ... existing code ...
end

# app/models/api_token.rb
class ApiToken < ApplicationRecord
  include SoftDeletable
  # ... existing code ...
end
```

### Usage Guide

```ruby
# Default scope only returns kept (non-discarded) records
User.all                    # Only kept users
User.kept                   # Explicit: only non-discarded
User.discarded              # Only soft-deleted records
User.with_discarded         # All records, including discarded

# Soft delete a record
user.discard!               # Sets discarded_at to Time.current
user.soft_delete!           # Alias for discard!

# Restore a soft-deleted record
user.undiscard!             # Clears discarded_at
user.restore!               # Alias for undiscard!

# Check status
user.discarded?             # true if soft-deleted
user.soft_deleted?          # Alias for discarded?
user.kept?                  # true if NOT soft-deleted
```

**Note:** Update Avo resources to show the `discarded_at` field so admins can see and manage soft-deleted records:

```ruby
# In each Avo resource, add:
field :discarded_at, as: :date_time, readonly: true, hide_on: [:index]
```

## Step 6: Mission Control Jobs Dashboard

Mission Control provides a web UI for monitoring Solid Queue jobs.

Create `config/initializers/mission_control.rb`:

```ruby
Rails.application.config.to_prepare do
  MissionControl::Jobs.base_controller_class = "Admin::BaseController"
  MissionControl::Jobs.http_basic_auth_enabled = false
end
```

This ensures the jobs dashboard inherits admin authentication from `Admin::BaseController` — only admin users can access it.

## Step 7: Routes

Add the following routes to `config/routes.rb`:

```ruby
Rails.application.routes.draw do
  # ... existing routes ...

  # Setup wizard (first-run only)
  resource :setup, only: [:show, :create], controller: "setup"

  # Development email preview
  if Rails.env.development?
    mount LetterOpenerWeb::Engine, at: "/letter_opener"
  end

  # Admin: Solid Queue job dashboard
  constraints ->(request) { request.env["warden"].user&.admin? } do
    mount MissionControl::Jobs::Engine, at: "/admin/jobs"
  end

  # ... rest of routes ...
end
```

### Route Summary

| Path | Purpose | Access |
|------|---------|--------|
| `/setup` | First-run setup wizard | Public (disabled after first admin exists) |
| `/letter_opener` | Email preview (dev only) | Development environment only |
| `/admin/jobs` | Solid Queue job dashboard | Admin users only |
