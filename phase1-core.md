# Phase 1: Project Setup & Core Models

## Step 1: Create the Rails App

```bash
rails new APP_NAME \
  --database=postgresql \
  --css=tailwind \
  --skip-jbuilder \
  --skip-test
```

Then add to Gemfile and bundle:

```ruby
# Gemfile additions for Phase 1
gem "acts_as_tenant"
gem "solid_queue"
gem "solid_cache"
gem "dotenv-rails", groups: [:development, :test]
```

The user should have already created a `.env` file before this phase begins (see SKILL.md Pre-Build section). Do NOT attempt to create or write to `.env` or `.env.example` files — Claude Code cannot write to these files. If the `.env` file doesn't exist, stop and ask the user to create it.

## Step 2: Generate Core Models

### User Model

The User model handles authentication (Devise is added in Phase 2) and has global attributes.

```bash
rails generate model User \
  first_name:string \
  last_name:string \
  email:string \
  time_zone:string \
  admin:boolean \
  accepted_terms_at:datetime \
  accepted_privacy_at:datetime
```

Add these to the User model:

```ruby
class User < ApplicationRecord
  has_many :memberships, dependent: :destroy
  has_many :teams, through: :memberships

  after_create :create_personal_team

  def personal_team
    teams.find_by(personal: true)
  end

  def name
    "#{first_name} #{last_name}".strip
  end

  private

  def create_personal_team
    team = teams.create!(name: "#{name}'s Team", personal: true)
    memberships.find_by(team: team).update!(roles: { "admin" => true })
  end
end
```

### Team Model

The primary tenant. Everything in the app belongs to a Team.

```bash
rails generate model Team \
  name:string \
  personal:boolean \
  domain:string \
  subdomain:string \
  extra_billing_info:text
```

```ruby
class Team < ApplicationRecord
  has_many :memberships, dependent: :destroy
  has_many :users, through: :memberships
  has_many :invitations, dependent: :destroy

  validates :name, presence: true

  def owner
    memberships.find_by("roles @> ?", '{"admin": true}')&.user
  end

  def personal?
    personal == true
  end
end
```

### Membership Model

Join table between User and Team. Stores roles as JSONB.

```bash
rails generate model Membership \
  user:references \
  team:references \
  roles:jsonb
```

Migration should set `roles` default to `{}`:

```ruby
t.jsonb :roles, default: {}, null: false
```

```ruby
class Membership < ApplicationRecord
  belongs_to :user
  belongs_to :team

  validates :user_id, uniqueness: { scope: :team_id }

  def admin?
    roles.dig("admin") == true
  end

  def member?
    !admin?
  end
end
```

### Invitation Model

For inviting users to teams via email with a secure token.

```bash
rails generate model Invitation \
  team:references \
  invited_by:references \
  email:string \
  name:string \
  token:string \
  roles:jsonb \
  accepted_at:datetime
```

```ruby
class Invitation < ApplicationRecord
  belongs_to :team
  belongs_to :invited_by, class_name: "User"

  has_secure_token :token

  validates :email, presence: true
  validates :email, uniqueness: { scope: :team_id }

  scope :pending, -> { where(accepted_at: nil) }

  def accept!(user)
    transaction do
      team.memberships.create!(user: user, roles: roles || {})
      update!(accepted_at: Time.current)
    end
  end
end
```

### Plan Model

Defines subscription tiers and their features.

```bash
rails generate model Plan \
  name:string \
  description:text \
  stripe_id:string \
  paddle_id:string \
  lemon_squeezy_id:string \
  price_monthly:integer \
  price_yearly:integer \
  trial_days:integer \
  features:jsonb \
  active:boolean \
  sort_order:integer
```

```ruby
class Plan < ApplicationRecord
  validates :name, presence: true
  scope :active, -> { where(active: true).order(:sort_order) }
  scope :visible, -> { active }

  def monthly_amount
    price_monthly.to_f / 100
  end

  def yearly_amount
    price_yearly.to_f / 100
  end

  def has_feature?(feature_name)
    features&.dig(feature_name.to_s) == true
  end
end
```

## Step 3: Multi-Tenant Middleware

Create `lib/middleware/tenant_middleware.rb`. This middleware resolves the current tenant before the request reaches Rails.

```ruby
# lib/middleware/tenant_middleware.rb
class TenantMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    request = ActionDispatch::Request.new(env)

    team = resolve_tenant(request)
    env["basex.current_team"] = team

    @app.call(env)
  end

  private

  def resolve_tenant(request)
    case BaseX.config.tenant_mode
    when :subdomain
      resolve_from_subdomain(request)
    when :path
      resolve_from_path(request)
    when :session
      nil # Resolved in ApplicationController from session
    end
  end

  def resolve_from_subdomain(request)
    subdomain = request.subdomain
    return nil if subdomain.blank? || subdomain == "www"
    Team.find_by(subdomain: subdomain)
  end

  def resolve_from_path(request)
    # Extract team ID from first path segment: /123/dashboard
    match = request.path.match(%r{^/(\d+)(/|$)})
    return nil unless match
    Team.find_by(id: match[1])
  end
end
```

Register the middleware in `config/application.rb`:

```ruby
config.middleware.use TenantMiddleware
```

## Step 4: BaseX Configuration

Create `config/initializers/base_x.rb`:

```ruby
module BaseX
  class Configuration
    attr_accessor :tenant_mode, :app_name, :support_email,
                  :require_personal_team, :default_plan

    def initialize
      @tenant_mode = :session  # :session, :subdomain, or :path
      @app_name = Rails.application.class.module_parent_name
      @support_email = "support@example.com"
      @require_personal_team = true
      @default_plan = nil
    end
  end

  class << self
    def config
      @config ||= Configuration.new
    end

    def configure
      yield(config)
    end
  end
end
```

## Step 5: Application Controller

Set up the tenant-aware base controller.

```ruby
class ApplicationController < ActionController::Base
  include SetCurrentTeam
  include Authentication
  include Authorization
end
```

Create `app/controllers/concerns/set_current_team.rb`:

```ruby
module SetCurrentTeam
  extend ActiveSupport::Concern

  included do
    helper_method :current_team
    before_action :set_current_team
  end

  def current_team
    @current_team
  end

  def set_current_team
    @current_team = resolve_team
    ActsAsTenant.current_tenant = @current_team if @current_team
  end

  private

  def resolve_team
    # Check middleware first (subdomain/path modes)
    return request.env["basex.current_team"] if request.env["basex.current_team"]

    # Session mode: use stored team_id
    return nil unless user_signed_in?
    team_id = session[:team_id] || current_user.personal_team&.id
    current_user.teams.find_by(id: team_id)
  end
end
```

Create `app/controllers/concerns/authorization.rb`:

```ruby
module Authorization
  extend ActiveSupport::Concern

  included do
    helper_method :current_membership
  end

  def current_membership
    return nil unless current_user && current_team
    @current_membership ||= current_user.memberships.find_by(team: current_team)
  end
end
```

## Step 6: Seeds

Create a seeds file that sets up a default admin user and a couple of plans.

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
  end
  puts "Development login: admin@example.com / password123"
end
```

## Step 7: Database Configuration

Ensure `config/database.yml` uses env vars:

```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  url: <%= ENV["DATABASE_URL"] %>

development:
  <<: *default
  database: <%= ENV.fetch("DB_NAME", "APP_NAME_development") %>

test:
  <<: *default
  database: <%= ENV.fetch("DB_NAME_TEST", "APP_NAME_test") %>

production:
  <<: *default
```

Replace `APP_NAME` with the actual app name throughout.
