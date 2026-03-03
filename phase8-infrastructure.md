# Phase 8: Rate Limiting, CORS & Pagination

## Step 1: Gem Additions

Add to Gemfile:

```ruby
# Rate limiting
gem "rack-attack", "~> 6.7"

# CORS for API
gem "rack-cors", "~> 2.0"

# Pagination
gem "pagy", "~> 9.0"
```

Run `bundle install`.

## Step 2: Rack::Attack Configuration

Create `config/initializers/rack_attack.rb`:

```ruby
class Rack::Attack
  ### Throttle login attempts ###
  # Limit login attempts to 5 per 20 seconds per IP + email
  throttle("logins/ip", limit: 5, period: 20.seconds) do |req|
    if req.path == "/users/sign_in" && req.post?
      req.ip
    end
  end

  throttle("logins/email", limit: 5, period: 20.seconds) do |req|
    if req.path == "/users/sign_in" && req.post?
      # Normalize email to prevent bypassing
      req.params.dig("user", "email")&.downcase&.strip
    end
  end

  ### Throttle password resets ###
  throttle("password_resets/ip", limit: 5, period: 60.seconds) do |req|
    if req.path == "/users/password" && req.post?
      req.ip
    end
  end

  ### Throttle invitation sends ###
  throttle("invitations/ip", limit: 10, period: 60.seconds) do |req|
    if req.path == "/invitations" && req.post?
      req.ip
    end
  end

  ### Throttle API by token ###
  throttle("api/token", limit: 60, period: 60.seconds) do |req|
    if req.path.start_with?("/api/")
      # Use Bearer token as discriminator
      req.env["HTTP_AUTHORIZATION"]&.split(" ")&.last
    end
  end

  ### Throttle API by IP (fallback) ###
  throttle("api/ip", limit: 30, period: 60.seconds) do |req|
    if req.path.start_with?("/api/")
      req.ip
    end
  end

  ### Allow2Ban for repeated login failures ###
  blocklist("logins/allow2ban") do |req|
    if req.path == "/users/sign_in" && req.post?
      # Block IP for 1 hour after 20 failed attempts in 1 hour
      Rack::Attack::Allow2Ban.filter(req.ip, maxretry: 20, findtime: 1.hour, bantime: 1.hour) do
        # Return true for failed login attempts (non-redirect response)
        req.env["warden.options"]&.dig(:action) == "unauthenticated"
      end
    end
  end

  ### Custom throttled response ###
  self.throttled_responder = lambda do |request|
    match_data = request.env["rack.attack.match_data"]
    now = match_data[:epoch_time]
    retry_after = match_data[:period] - (now % match_data[:period])

    headers = {
      "Content-Type" => "application/json",
      "Retry-After" => retry_after.to_s
    }

    body = {
      error: "Rate limit exceeded",
      retry_after: retry_after
    }.to_json

    [429, headers, [body]]
  end
end
```

## Step 3: Enable Rack::Attack Middleware

Add to `config/application.rb`:

```ruby
# Inside Application class
config.middleware.use Rack::Attack
```

## Step 4: Rack::Cors Configuration

Create `config/initializers/cors.rb`:

```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins ENV.fetch("CORS_ORIGINS", "*")

    resource "/api/*",
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      expose: ["X-Total-Count", "X-Page", "X-Per-Page"],
      max_age: 600
  end
end
```

Add `CORS_ORIGINS` to the environment variables documentation. In production, set this to your frontend domain(s), e.g. `https://app.example.com`.

## Step 5: Pagy Configuration

Create `config/initializers/pagy.rb`:

```ruby
# Pagy defaults
Pagy::DEFAULT[:limit] = 25
Pagy::DEFAULT[:overflow] = :last_page
```

## Step 6: Include Pagy in Controllers and Helpers

Update `app/controllers/application_controller.rb` to include `Pagy::Backend`:

```ruby
class ApplicationController < ActionController::Base
  include Pagy::Backend
  include SetCurrentTeam
  include Authentication
  include Authorization
  include SetupGuard
  include TermsGate
end
```

Update `app/helpers/application_helper.rb` to include `Pagy::Frontend`:

```ruby
module ApplicationHelper
  include Pagy::Frontend
end
```

Update `app/controllers/api/base_controller.rb` to include `Pagy::Backend` and add pagination header helpers:

```ruby
module Api
  class BaseController < ActionController::API
    include ActionController::HttpAuthentication::Token::ControllerMethods
    include Pagy::Backend

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

    def set_pagination_headers(pagy)
      response.headers["X-Total-Count"] = pagy.count.to_s
      response.headers["X-Page"] = pagy.page.to_s
      response.headers["X-Per-Page"] = pagy.limit.to_s
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

## Step 7: Apply Pagination to Controllers

Update `app/controllers/memberships_controller.rb` index action:

```ruby
def index
  authorize Membership
  @pagy, @memberships = pagy(current_team.memberships.includes(:user))
  @pending_invitations = current_team.invitations.pending.order(created_at: :desc)
end
```

Update `app/controllers/notifications_controller.rb` index action:

```ruby
def index
  @pagy, @notifications = pagy(
    current_user.notifications
      .includes(:event)
      .order(created_at: :desc)
  )
  @unread_count = current_user.notifications.unread.count
end
```

Update `app/controllers/api_tokens_controller.rb` index action:

```ruby
def index
  @pagy, @api_tokens = pagy(current_team.api_tokens.where(user: current_user))
end
```

## Step 8: Pagination Partial

Create `app/views/shared/_pagination.html.erb`:

```erb
<%== pagy_nav(@pagy) if @pagy.pages > 1 %>
```

To use this partial in views, add it after any paginated list:

```erb
<%= render "shared/pagination" %>
```

### Pagy Tailwind Styling

Add to `config/initializers/pagy.rb` (or create a helper) to get Tailwind-styled pagination:

```ruby
# Add Tailwind styling to pagy nav
Pagy::DEFAULT[:nav_aria_label] = "Pagination"
```

For full Tailwind styling, override the `pagy_nav` helper in `ApplicationHelper`:

```ruby
module ApplicationHelper
  include Pagy::Frontend

  def pagy_tailwind_nav(pagy)
    return "" if pagy.pages <= 1

    link = pagy_link_proc(pagy,
      classes: "relative inline-flex items-center px-4 py-2 text-sm font-medium text-gray-700 bg-white border border-gray-300 hover:bg-gray-50",
      active_classes: "relative inline-flex items-center px-4 py-2 text-sm font-medium text-white bg-primary-600 border border-primary-600"
    )

    html = +%(<nav class="flex items-center justify-between border-t border-gray-200 px-4 sm:px-0 mt-6 pt-4">)
    html << %(<div class="flex flex-1 justify-between sm:justify-end gap-1">)
    html << (pagy.prev ? link.call(pagy.prev, "&larr; Previous", classes: "relative inline-flex items-center rounded-md px-3 py-2 text-sm font-medium text-gray-700 bg-white border border-gray-300 hover:bg-gray-50") : "")
    html << (pagy.next ? link.call(pagy.next, "Next &rarr;", classes: "relative inline-flex items-center rounded-md px-3 py-2 text-sm font-medium text-gray-700 bg-white border border-gray-300 hover:bg-gray-50") : "")
    html << %(</div></nav>)

    html.html_safe
  end
end
```

## Step 9: API Pagination Headers

API controllers should set pagination headers. Example usage in an API controller:

```ruby
# app/controllers/api/v1/teams_controller.rb
module Api
  module V1
    class TeamsController < Api::BaseController
      def show
        render json: current_team.as_json(only: [:id, :name, :created_at])
      end

      # Example paginated endpoint
      def members
        @pagy, @members = pagy(current_team.memberships.includes(:user))
        set_pagination_headers(@pagy)
        render json: @members.map { |m|
          { id: m.id, user: m.user.as_json(only: [:id, :first_name, :last_name, :email]), role: m.admin? ? "admin" : "member" }
        }
      end
    end
  end
end
```

### Response Headers Example

```
X-Total-Count: 42
X-Page: 2
X-Per-Page: 25
```

### Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `CORS_ORIGINS` | No | Allowed CORS origins for API (default: `*`). Set to your frontend domain in production. |
