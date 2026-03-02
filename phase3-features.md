# Phase 3: Notifications, Admin & API

## Step 1: Noticed Notifications

Add to Gemfile:

```ruby
gem "noticed", "~> 2.0"
```

Run:

```bash
rails generate noticed:install
rails db:migrate
```

Create a base notification class and an example notification:

```ruby
# app/notifications/base_notification.rb
class BaseNotification < Noticed::Event
  deliver_by :database
  deliver_by :email do |config|
    config.mailer = "NotificationMailer"
    config.method = :notification_email
  end

  # Override in subclasses to customize
  def message
    raise NotImplementedError
  end

  def url
    nil
  end
end
```

Example notification for team invitations:

```ruby
# app/notifications/invitation_notification.rb
class InvitationNotification < BaseNotification
  deliver_by :database
  deliver_by :email do |config|
    config.mailer = "InvitationMailer"
    config.method = :invite_email
  end

  notification_methods do
    def message
      "You've been invited to join #{params[:team].name}"
    end

    def url
      accept_invitation_url(params[:invitation].token)
    end
  end
end
```

Example notification for new member joined:

```ruby
# app/notifications/member_joined_notification.rb
class MemberJoinedNotification < BaseNotification
  notification_methods do
    def message
      "#{params[:user].name} joined #{params[:team].name}"
    end

    def url
      team_memberships_url(params[:team])
    end
  end
end
```

Create `app/controllers/notifications_controller.rb`:

```ruby
class NotificationsController < ApplicationController
  before_action :authenticate_user!

  def index
    @notifications = current_user.notifications
                      .includes(:event)
                      .order(created_at: :desc)
                      .limit(50)
    @unread_count = current_user.notifications.unread.count
  end

  def mark_as_read
    notification = current_user.notifications.find(params[:id])
    notification.mark_as_read!
    redirect_to notification.event.url || notifications_path
  end

  def mark_all_as_read
    current_user.notifications.unread.mark_as_read!
    redirect_to notifications_path, notice: "All notifications marked as read."
  end
end
```

Add a notification bell helper partial at `app/views/shared/_notification_bell.html.erb`:

```erb
<% unread = current_user.notifications.unread.count %>
<%= link_to notifications_path, class: "relative" do %>
  <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
          d="M15 17h5l-1.405-1.405A2.032 2.032 0 0118 14.158V11a6.002 6.002 0 00-4-5.659V5a2 2 0 10-4 0v.341C7.67 6.165 6 8.388 6 11v3.159c0 .538-.214 1.055-.595 1.436L4 17h5m6 0v1a3 3 0 11-6 0v-1m6 0H9" />
  </svg>
  <% if unread > 0 %>
    <span class="absolute -top-1 -right-1 bg-red-500 text-white text-xs rounded-full w-5 h-5 flex items-center justify-center">
      <%= unread > 9 ? "9+" : unread %>
    </span>
  <% end %>
<% end %>
```

## Step 2: Admin Panel

Add to Gemfile (choose one):

```ruby
# Option A: Avo (recommended — modern, rich UI)
gem "avo"

# Option B: Administrate (simpler, more customizable)
gem "administrate"
```

For Avo, run:

```bash
rails generate avo:install
```

Create admin resources:

```ruby
# app/avo/resources/user.rb
class Avo::Resources::User < Avo::BaseResource
  self.title = :name
  self.includes = [:teams, :memberships]

  def fields
    field :id, as: :id
    field :first_name, as: :text
    field :last_name, as: :text
    field :email, as: :text
    field :admin, as: :boolean
    field :sign_in_count, as: :number, readonly: true
    field :current_sign_in_at, as: :date_time, readonly: true
    field :teams, as: :has_many, through: :memberships
    field :created_at, as: :date_time, readonly: true
  end

  def actions
    action Avo::Actions::ImpersonateUser
  end
end

# app/avo/resources/team.rb
class Avo::Resources::Team < Avo::BaseResource
  self.title = :name
  self.includes = [:users, :memberships]

  def fields
    field :id, as: :id
    field :name, as: :text
    field :personal, as: :boolean
    field :subdomain, as: :text
    field :users, as: :has_many, through: :memberships
    field :created_at, as: :date_time, readonly: true
  end
end

# app/avo/resources/plan.rb
class Avo::Resources::Plan < Avo::BaseResource
  self.title = :name

  def fields
    field :id, as: :id
    field :name, as: :text
    field :price_monthly, as: :number
    field :price_yearly, as: :number
    field :trial_days, as: :number
    field :active, as: :boolean
    field :sort_order, as: :number
    field :features, as: :code, language: "json"
  end
end
```

Secure admin access in `config/initializers/avo.rb`:

```ruby
Avo.configure do |config|
  config.root_path = "/admin"
  config.current_user_method = :current_user
  config.authenticate_with do
    redirect_to root_path unless current_user&.admin?
  end
end
```

## Step 3: Announcements

System-wide announcements that admins can broadcast to all users.

```bash
rails generate model Announcement \
  title:string \
  body:text \
  announcement_type:string \
  published_at:datetime
```

```ruby
class Announcement < ApplicationRecord
  validates :title, :body, :announcement_type, presence: true

  TYPES = %w[new fix improvement update].freeze

  scope :published, -> { where("published_at <= ?", Time.current) }
  scope :recent, -> { published.order(published_at: :desc).limit(10) }

  def published?
    published_at.present? && published_at <= Time.current
  end

  def type_color
    case announcement_type
    when "new" then "bg-blue-100 text-blue-800"
    when "fix" then "bg-red-100 text-red-800"
    when "improvement" then "bg-green-100 text-green-800"
    when "update" then "bg-purple-100 text-purple-800"
    else "bg-gray-100 text-gray-800"
    end
  end
end
```

Track which users have seen announcements:

```bash
rails generate model AnnouncementRead \
  user:references \
  announcement:references
```

```ruby
class AnnouncementRead < ApplicationRecord
  belongs_to :user
  belongs_to :announcement
end

# Add to User model:
def unread_announcements
  Announcement.published
    .where.not(id: AnnouncementRead.where(user: self).select(:announcement_id))
    .order(published_at: :desc)
end
```

## Step 4: API Tokens

```bash
rails generate model ApiToken \
  user:references \
  team:references \
  name:string \
  token:string \
  last_used_at:datetime \
  metadata:jsonb \
  expires_at:datetime \
  revoked_at:datetime
```

```ruby
class ApiToken < ApplicationRecord
  belongs_to :user
  belongs_to :team

  has_secure_token :token

  scope :active, -> { where(revoked_at: nil).where("expires_at IS NULL OR expires_at > ?", Time.current) }

  validates :name, presence: true

  def active?
    revoked_at.nil? && (expires_at.nil? || expires_at > Time.current)
  end

  def revoke!
    update!(revoked_at: Time.current)
  end

  def touch_last_used!
    update_column(:last_used_at, Time.current)
  end
end
```

Create `app/controllers/api/base_controller.rb`:

```ruby
module Api
  class BaseController < ActionController::API
    include ActionController::HttpAuthentication::Token::ControllerMethods

    before_action :authenticate_api_token!

    private

    def authenticate_api_token!
      authenticate_or_request_with_http_token do |token, _options|
        @api_token = ApiToken.active.find_by(token: token)
        if @api_token
          @api_token.touch_last_used!
          @current_user = @api_token.user
          @current_team = @api_token.team
          ActsAsTenant.current_tenant = @current_team
          true
        end
      end
    end

    def current_user
      @current_user
    end

    def current_team
      @current_team
    end
  end
end
```

Example versioned API controller:

```ruby
# app/controllers/api/v1/teams_controller.rb
module Api
  module V1
    class TeamsController < Api::BaseController
      def show
        render json: current_team.as_json(only: [:id, :name, :created_at])
      end
    end
  end
end
```

Create `app/controllers/api_tokens_controller.rb` for managing tokens in the UI:

```ruby
class ApiTokensController < ApplicationController
  before_action :authenticate_user!

  def index
    @api_tokens = current_team.api_tokens.where(user: current_user)
  end

  def create
    @token = current_team.api_tokens.build(
      user: current_user,
      name: params[:api_token][:name]
    )
    if @token.save
      # Show the token value only once
      flash[:token_value] = @token.token
      redirect_to api_tokens_path, notice: "Token created. Copy it now — it won't be shown again."
    else
      redirect_to api_tokens_path, alert: @token.errors.full_messages.join(", ")
    end
  end

  def destroy
    token = current_team.api_tokens.find(params[:id])
    token.revoke!
    redirect_to api_tokens_path, notice: "Token revoked."
  end
end
```

## Routes for Phase 3

```ruby
# Add to config/routes.rb
resources :notifications, only: [:index] do
  member do
    post :mark_as_read
  end
  collection do
    post :mark_all_as_read
  end
end

resources :api_tokens, only: [:index, :create, :destroy]
resources :announcements, only: [:index]

namespace :api do
  namespace :v1 do
    resource :team, only: [:show]
    # Add more API resources here
  end
end
```
