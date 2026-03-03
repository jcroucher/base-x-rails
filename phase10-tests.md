# Phase 10: Test Suite

This phase adds a comprehensive Minitest test suite covering models, controllers, system tests, and CI configuration.

## Step 1: Test Helper

Update `test/test_helper.rb` with multi-tenant helpers and Devise integration:

```ruby
ENV["RAILS_ENV"] ||= "test"
require_relative "../config/environment"
require "rails/test_help"

class ActiveSupport::TestCase
  fixtures :all

  # Helper to sign in and set team context
  def sign_in_as(user, team: nil)
    sign_in user
    team ||= user.personal_team
    set_current_team(team)
  end

  def set_current_team(team)
    ActsAsTenant.current_tenant = team
  end

  # Reset tenant after each test
  teardown do
    ActsAsTenant.current_tenant = nil
  end
end

class ActionDispatch::IntegrationTest
  include Devise::Test::IntegrationHelpers

  def sign_in_as(user, team: nil)
    sign_in user
    team ||= user.personal_team
    post switch_team_path(team) if team
  end
end
```

## Step 2: Application System Test Case

Create `test/application_system_test_case.rb`:

```ruby
require "test_helper"

class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  driven_by :selenium, using: :headless_chrome, screen_size: [1400, 900]

  include Devise::Test::IntegrationHelpers

  def sign_in_as(user)
    visit new_user_session_path
    fill_in "Email", with: user.email
    fill_in "Password", with: "password123"
    click_button "Log in"
    assert_current_path dashboard_path
  end
end
```

## Step 3: Fixtures

### `test/fixtures/users.yml`

```yaml
admin:
  first_name: Admin
  last_name: User
  email: admin@example.com
  encrypted_password: <%= Devise::Encryptor.digest(User, 'password123') %>
  admin: true
  confirmed_at: <%= Time.current %>
  accepted_terms_at: <%= Time.current %>

member:
  first_name: Regular
  last_name: Member
  email: member@example.com
  encrypted_password: <%= Devise::Encryptor.digest(User, 'password123') %>
  admin: false
  confirmed_at: <%= Time.current %>
  accepted_terms_at: <%= Time.current %>

unconfirmed:
  first_name: Unconfirmed
  last_name: User
  email: unconfirmed@example.com
  encrypted_password: <%= Devise::Encryptor.digest(User, 'password123') %>
  admin: false
  confirmed_at: null
  accepted_terms_at: null
```

### `test/fixtures/teams.yml`

```yaml
acme:
  name: Acme Corp
  personal: false

admin_personal:
  name: "Admin User's Team"
  personal: true

member_personal:
  name: "Regular Member's Team"
  personal: true
```

### `test/fixtures/memberships.yml`

```yaml
admin_acme:
  user: admin
  team: acme
  roles:
    admin: true

member_acme:
  user: member
  team: acme
  roles: {}

admin_personal:
  user: admin
  team: admin_personal
  roles:
    admin: true

member_personal:
  user: member
  team: member_personal
  roles:
    admin: true
```

### `test/fixtures/invitations.yml`

```yaml
pending:
  team: acme
  invited_by: admin
  email: newuser@example.com
  name: New User
  token: pending_invitation_token
  roles: {}
  accepted_at: null

accepted:
  team: acme
  invited_by: admin
  email: accepted@example.com
  name: Accepted User
  token: accepted_invitation_token
  roles: {}
  accepted_at: <%= 1.day.ago %>
```

### `test/fixtures/plans.yml`

```yaml
free:
  name: Free
  stripe_id: free
  price_monthly: 0
  price_yearly: 0
  trial_days: 0
  features:
    max_members: 5
  active: true
  sort_order: 0

pro:
  name: Pro
  stripe_id: pro_monthly
  price_monthly: 2900
  price_yearly: 29000
  trial_days: 14
  features:
    max_members: 25
    priority_support: true
  active: true
  sort_order: 1
```

### `test/fixtures/api_tokens.yml`

```yaml
active:
  user: admin
  team: acme
  name: Active Token
  token: active_api_token_123
  last_used_at: null
  expires_at: null
  revoked_at: null

revoked:
  user: admin
  team: acme
  name: Revoked Token
  token: revoked_api_token_456
  last_used_at: <%= 1.week.ago %>
  expires_at: null
  revoked_at: <%= 1.day.ago %>
```

### `test/fixtures/announcements.yml`

```yaml
published:
  title: New Feature Released
  body: We've added a great new feature!
  announcement_type: new
  published_at: <%= 1.day.ago %>

draft:
  title: Upcoming Maintenance
  body: Scheduled maintenance window next week.
  announcement_type: update
  published_at: null
```

### `test/fixtures/connected_accounts.yml`

```yaml
google:
  user: admin
  provider: google_oauth2
  uid: "123456789"
  access_token: google_access_token
  refresh_token: google_refresh_token
  expires_at: <%= 1.hour.from_now %>

github:
  user: member
  provider: github
  uid: "987654321"
  access_token: github_access_token
  refresh_token: null
  expires_at: null
```

## Step 4: Model Tests

### `test/models/user_test.rb`

```ruby
require "test_helper"

class UserTest < ActiveSupport::TestCase
  test "name returns full name" do
    user = users(:admin)
    assert_equal "Admin User", user.name
  end

  test "name handles missing last name" do
    user = User.new(first_name: "Jane")
    assert_equal "Jane", user.name
  end

  test "creating a user creates a personal team" do
    user = User.create!(
      first_name: "Test",
      last_name: "Person",
      email: "test_personal@example.com",
      password: "password123",
      confirmed_at: Time.current
    )
    assert user.personal_team.present?
    assert user.personal_team.personal?
  end

  test "personal_team returns the personal team" do
    user = users(:admin)
    assert_equal teams(:admin_personal), user.personal_team
  end

  test "soft delete sets discarded_at" do
    user = users(:member)
    user.soft_delete!
    assert user.soft_deleted?
    assert user.discarded_at.present?
  end

  test "restore clears discarded_at" do
    user = users(:member)
    user.soft_delete!
    user.restore!
    refute user.soft_deleted?
    assert_nil user.discarded_at
  end
end
```

### `test/models/team_test.rb`

```ruby
require "test_helper"

class TeamTest < ActiveSupport::TestCase
  test "owner returns the admin user" do
    team = teams(:acme)
    assert_equal users(:admin), team.owner
  end

  test "personal? returns true for personal teams" do
    assert teams(:admin_personal).personal?
  end

  test "personal? returns false for organization teams" do
    refute teams(:acme).personal?
  end

  test "validates name presence" do
    team = Team.new(name: nil)
    refute team.valid?
    assert_includes team.errors[:name], "can't be blank"
  end
end
```

### `test/models/membership_test.rb`

```ruby
require "test_helper"

class MembershipTest < ActiveSupport::TestCase
  test "admin? returns true for admin role" do
    membership = memberships(:admin_acme)
    assert membership.admin?
  end

  test "admin? returns false for regular member" do
    membership = memberships(:member_acme)
    refute membership.admin?
  end

  test "member? returns true for non-admin" do
    membership = memberships(:member_acme)
    assert membership.member?
  end

  test "enforces uniqueness of user per team" do
    duplicate = Membership.new(
      user: users(:admin),
      team: teams(:acme),
      roles: {}
    )
    refute duplicate.valid?
    assert_includes duplicate.errors[:user_id], "has already been taken"
  end
end
```

### `test/models/invitation_test.rb`

```ruby
require "test_helper"

class InvitationTest < ActiveSupport::TestCase
  test "accept! creates membership and sets accepted_at" do
    invitation = invitations(:pending)
    new_user = User.create!(
      first_name: "New",
      last_name: "Person",
      email: "newperson@example.com",
      password: "password123",
      confirmed_at: Time.current
    )

    assert_difference "Membership.count", 1 do
      invitation.accept!(new_user)
    end

    assert invitation.reload.accepted_at.present?
    assert new_user.teams.include?(invitation.team)
  end

  test "has a secure token" do
    invitation = invitations(:pending)
    assert invitation.token.present?
  end

  test "pending scope excludes accepted invitations" do
    pending = Invitation.pending
    assert_includes pending, invitations(:pending)
    refute_includes pending, invitations(:accepted)
  end

  test "validates email presence" do
    invitation = Invitation.new(email: nil, team: teams(:acme), invited_by: users(:admin))
    refute invitation.valid?
    assert_includes invitation.errors[:email], "can't be blank"
  end
end
```

### `test/models/plan_test.rb`

```ruby
require "test_helper"

class PlanTest < ActiveSupport::TestCase
  test "has_feature? returns true when feature exists" do
    plan = plans(:pro)
    assert plan.has_feature?("priority_support")
  end

  test "has_feature? returns false for missing feature" do
    plan = plans(:free)
    refute plan.has_feature?("priority_support")
  end

  test "monthly_amount converts cents to dollars" do
    plan = plans(:pro)
    assert_equal 29.0, plan.monthly_amount
  end

  test "yearly_amount converts cents to dollars" do
    plan = plans(:pro)
    assert_equal 290.0, plan.yearly_amount
  end

  test "active scope returns active plans in sort order" do
    plans = Plan.active
    assert plans.all?(&:active?)
    assert_equal plans.map(&:sort_order), plans.map(&:sort_order).sort
  end
end
```

### `test/models/api_token_test.rb`

```ruby
require "test_helper"

class ApiTokenTest < ActiveSupport::TestCase
  test "active? returns true for valid token" do
    token = api_tokens(:active)
    assert token.active?
  end

  test "active? returns false for revoked token" do
    token = api_tokens(:revoked)
    refute token.active?
  end

  test "revoke! sets revoked_at" do
    token = api_tokens(:active)
    token.revoke!
    assert token.revoked_at.present?
    refute token.active?
  end

  test "active scope excludes revoked tokens" do
    active_tokens = ApiToken.active
    assert_includes active_tokens, api_tokens(:active)
    refute_includes active_tokens, api_tokens(:revoked)
  end

  test "validates name presence" do
    token = ApiToken.new(name: nil, user: users(:admin), team: teams(:acme))
    refute token.valid?
    assert_includes token.errors[:name], "can't be blank"
  end
end
```

### `test/models/announcement_test.rb`

```ruby
require "test_helper"

class AnnouncementTest < ActiveSupport::TestCase
  test "published scope returns only published announcements" do
    published = Announcement.published
    assert_includes published, announcements(:published)
    refute_includes published, announcements(:draft)
  end

  test "published? returns true when published_at is in the past" do
    assert announcements(:published).published?
  end

  test "published? returns false when published_at is nil" do
    refute announcements(:draft).published?
  end

  test "type_color returns correct class for each type" do
    announcement = announcements(:published)
    assert_includes announcement.type_color, "blue"

    announcement.announcement_type = "fix"
    assert_includes announcement.type_color, "red"

    announcement.announcement_type = "improvement"
    assert_includes announcement.type_color, "green"
  end
end
```

## Step 5: Controller / Integration Tests

### `test/controllers/settings_controller_test.rb`

```ruby
require "test_helper"

class SettingsControllerTest < ActionDispatch::IntegrationTest
  setup do
    @user = users(:admin)
    sign_in @user
  end

  test "show renders settings page" do
    get settings_path
    assert_response :success
  end

  test "update changes user profile" do
    patch settings_path, params: {
      user: { first_name: "Updated", last_name: "Name", time_zone: "Pacific Time (US & Canada)" }
    }
    assert_redirected_to settings_path
    @user.reload
    assert_equal "Updated", @user.first_name
    assert_equal "Pacific Time (US & Canada)", @user.time_zone
  end

  test "requires authentication" do
    sign_out @user
    get settings_path
    assert_redirected_to new_user_session_path
  end
end
```

### `test/controllers/passwords_controller_test.rb`

```ruby
require "test_helper"

class PasswordsControllerTest < ActionDispatch::IntegrationTest
  setup do
    @user = users(:admin)
    sign_in @user
  end

  test "update with correct current password changes password" do
    patch password_path, params: {
      user: {
        current_password: "password123",
        password: "newpassword456",
        password_confirmation: "newpassword456"
      }
    }
    assert_redirected_to settings_path
    assert_equal "Password updated.", flash[:notice]
  end

  test "update with wrong current password fails" do
    patch password_path, params: {
      user: {
        current_password: "wrongpassword",
        password: "newpassword456",
        password_confirmation: "newpassword456"
      }
    }
    assert_redirected_to settings_path
    assert_equal "Current password is incorrect.", flash[:alert]
  end
end
```

### `test/controllers/memberships_controller_test.rb`

```ruby
require "test_helper"

class MembershipsControllerTest < ActionDispatch::IntegrationTest
  setup do
    @admin = users(:admin)
    @member = users(:member)
    @team = teams(:acme)
  end

  test "index shows team members for member" do
    sign_in @member
    get memberships_path
    assert_response :success
  end

  test "update role as admin" do
    sign_in @admin
    membership = memberships(:member_acme)
    patch membership_path(membership), params: {
      membership: { role: "admin" }
    }
    assert_redirected_to memberships_path
    assert membership.reload.admin?
  end

  test "update role as member is forbidden" do
    sign_in @member
    membership = memberships(:admin_acme)
    assert_raises(Pundit::NotAuthorizedError) do
      patch membership_path(membership), params: {
        membership: { role: "member" }
      }
    end
  end

  test "member can leave team" do
    sign_in @member
    membership = memberships(:member_acme)
    assert_difference "Membership.count", -1 do
      delete membership_path(membership)
    end
    assert_redirected_to dashboard_path
  end
end
```

### `test/controllers/invitations_controller_test.rb`

```ruby
require "test_helper"

class InvitationsControllerTest < ActionDispatch::IntegrationTest
  setup do
    @admin = users(:admin)
    @member = users(:member)
    @team = teams(:acme)
  end

  test "admin can create invitation" do
    sign_in @admin
    assert_difference "Invitation.count", 1 do
      post invitations_path, params: {
        invitation: { email: "newinvite@example.com", name: "New Invite" }
      }
    end
    assert_redirected_to memberships_path
  end

  test "member cannot create invitation" do
    sign_in @member
    assert_raises(Pundit::NotAuthorizedError) do
      post invitations_path, params: {
        invitation: { email: "newinvite@example.com", name: "New Invite" }
      }
    end
  end

  test "accept invitation when signed in" do
    sign_in @member
    invitation = invitations(:pending)
    # Remove member from acme first to test fresh acceptance
    memberships(:member_acme).destroy

    get invitation_accept_path(token: invitation.token)
    assert_redirected_to dashboard_path
    assert invitation.reload.accepted_at.present?
  end

  test "accept invitation when not signed in redirects to registration" do
    invitation = invitations(:pending)
    get invitation_accept_path(token: invitation.token)
    assert_redirected_to new_user_registration_path
  end
end
```

### `test/controllers/dashboard_controller_test.rb`

```ruby
require "test_helper"

class DashboardControllerTest < ActionDispatch::IntegrationTest
  test "authenticated user can access dashboard" do
    sign_in users(:admin)
    get dashboard_path
    assert_response :success
  end

  test "unauthenticated user is redirected to sign in" do
    get dashboard_path
    assert_redirected_to new_user_session_path
  end
end
```

### `test/controllers/teams_controller_test.rb`

```ruby
require "test_helper"

class TeamsControllerTest < ActionDispatch::IntegrationTest
  setup do
    @user = users(:admin)
    sign_in @user
  end

  test "create a new team" do
    assert_difference "Team.count", 1 do
      post teams_path, params: { team: { name: "New Team" } }
    end
  end

  test "switch team updates session" do
    team = teams(:acme)
    post switch_team_path(team)
    assert_redirected_to dashboard_path
  end
end
```

### `test/controllers/terms_controller_test.rb`

```ruby
require "test_helper"

class TermsControllerTest < ActionDispatch::IntegrationTest
  test "user without accepted terms is redirected to terms page" do
    user = users(:unconfirmed)
    user.update!(confirmed_at: Time.current, accepted_terms_at: nil)
    sign_in user
    get dashboard_path
    assert_redirected_to terms_path
  end

  test "terms page renders successfully" do
    user = users(:unconfirmed)
    user.update!(confirmed_at: Time.current, accepted_terms_at: nil)
    sign_in user
    get terms_path
    assert_response :success
    assert_select "h1", "Terms of Service"
  end

  test "accepting terms updates the user" do
    user = users(:unconfirmed)
    user.update!(confirmed_at: Time.current, accepted_terms_at: nil)
    sign_in user
    post accept_terms_path
    assert_redirected_to dashboard_path
    assert user.reload.accepted_terms_at.present?
  end

  test "user with accepted terms can access dashboard" do
    sign_in users(:admin)
    get dashboard_path
    assert_response :success
  end
end
```

### `test/controllers/api/v1/teams_controller_test.rb`

```ruby
require "test_helper"

class Api::V1::TeamsControllerTest < ActionDispatch::IntegrationTest
  test "valid token returns team data" do
    token = api_tokens(:active)
    get api_v1_team_path, headers: {
      "Authorization" => "Bearer #{token.token}"
    }
    assert_response :success
    json = JSON.parse(response.body)
    assert_equal token.team.name, json["name"]
  end

  test "invalid token returns unauthorized" do
    get api_v1_team_path, headers: {
      "Authorization" => "Bearer invalid_token"
    }
    assert_response :unauthorized
  end

  test "revoked token returns unauthorized" do
    token = api_tokens(:revoked)
    get api_v1_team_path, headers: {
      "Authorization" => "Bearer #{token.token}"
    }
    assert_response :unauthorized
  end

  test "missing token returns unauthorized" do
    get api_v1_team_path
    assert_response :unauthorized
  end
end
```

## Step 6: System Tests

### `test/system/registration_test.rb`

```ruby
require "application_system_test_case"

class RegistrationTest < ApplicationSystemTestCase
  test "user can register" do
    visit new_user_registration_path

    fill_in "First name", with: "Test"
    fill_in "Last name", with: "Person"
    fill_in "Email", with: "newregistration@example.com"
    fill_in "Password", with: "password123"
    fill_in "Password confirmation", with: "password123"

    click_button "Sign up"

    # User should be redirected (either to confirmation notice or dashboard)
    assert_no_current_path new_user_registration_path
  end
end
```

### `test/system/team_management_test.rb`

```ruby
require "application_system_test_case"

class TeamManagementTest < ApplicationSystemTestCase
  setup do
    @admin = users(:admin)
  end

  test "admin can create a new team" do
    sign_in_as @admin

    visit new_team_path
    fill_in "Name", with: "Test Team"
    click_button "Create Team"

    assert_text "Test Team"
  end

  test "admin can view team members" do
    sign_in_as @admin
    visit memberships_path

    assert_text "Team Members"
    assert_text @admin.name
  end
end
```

### `test/system/settings_test.rb`

```ruby
require "application_system_test_case"

class SettingsTest < ApplicationSystemTestCase
  setup do
    @user = users(:admin)
  end

  test "user can update profile" do
    sign_in_as @user

    visit settings_path
    fill_in "First name", with: "Updated"
    fill_in "Last name", with: "Name"
    click_button "Save Changes"

    assert_text "Settings updated"
  end
end
```

## Step 7: CI Configuration

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: app_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    env:
      RAILS_ENV: test
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/app_test

    steps:
      - uses: actions/checkout@v4

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: .ruby-version
          bundler-cache: true

      - name: Set up database
        run: |
          bin/rails db:create db:migrate

      - name: Run tests
        run: bin/rails test

      - name: Run system tests
        run: bin/rails test:system

  lint:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: .ruby-version
          bundler-cache: true

      - name: Run RuboCop
        run: bundle exec rubocop --parallel
```

### CI Summary

The CI workflow runs on every push to `main` and every pull request. It:

1. Starts a PostgreSQL 16 service container
2. Sets up Ruby from `.ruby-version`
3. Installs gems with bundler caching
4. Creates and migrates the test database
5. Runs the full Minitest suite (`rails test`)
6. Runs system tests with headless Chrome (`rails test:system`)
7. Runs RuboCop lint checks in parallel
