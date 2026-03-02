# Phase 2: Authentication & Billing

## Step 1: Devise Setup

Add to Gemfile:

```ruby
gem "devise"
gem "omniauth"
gem "omniauth-google-oauth2"
gem "omniauth-github"
gem "omniauth-rails_csrf_protection"
gem "rotp"           # Time-based OTP for 2FA
gem "rqrcode"        # QR code generation for 2FA
gem "pretender"      # Admin impersonation
gem "pundit"         # Authorization
```

Run `bundle install`, then:

```bash
rails generate devise:install
rails generate devise User
```

Since User already exists, modify the generated migration to only add Devise columns:

```ruby
class AddDeviseToUsers < ActiveRecord::Migration[8.1]
  def change
    change_table :users, bulk: true do |t|
      ## Devise core
      t.string :encrypted_password, null: false, default: ""
      t.string :reset_password_token
      t.datetime :reset_password_sent_at
      t.datetime :remember_created_at
      t.integer :sign_in_count, default: 0, null: false
      t.datetime :current_sign_in_at
      t.datetime :last_sign_in_at
      t.string :current_sign_in_ip
      t.string :last_sign_in_ip

      ## Confirmable
      t.string :confirmation_token
      t.datetime :confirmed_at
      t.datetime :confirmation_sent_at
      t.string :unconfirmed_email

      ## Lockable
      t.integer :failed_attempts, default: 0, null: false
      t.string :unlock_token
      t.datetime :locked_at

      ## Two-Factor
      t.string :otp_secret
      t.boolean :otp_required_for_login, default: false
      t.string :otp_backup_codes, array: true

      ## OmniAuth
      t.string :provider
      t.string :uid
      t.string :avatar_url
    end

    add_index :users, :email, unique: true
    add_index :users, :reset_password_token, unique: true
    add_index :users, :confirmation_token, unique: true
    add_index :users, :unlock_token, unique: true
  end
end
```

Update the User model to enable Devise modules:

```ruby
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable,
         :confirmable, :lockable, :trackable,
         :omniauthable, omniauth_providers: [:google_oauth2, :github]

  # ... existing associations and methods ...
end
```

## Step 2: Two-Factor Authentication

Add 2FA methods to User model:

```ruby
# In user.rb
def enable_two_factor!
  self.otp_secret = ROTP::Base32.random
  self.otp_backup_codes = generate_backup_codes
  save!
end

def disable_two_factor!
  update!(
    otp_secret: nil,
    otp_required_for_login: false,
    otp_backup_codes: nil
  )
end

def verify_otp(code)
  totp = ROTP::TOTP.new(otp_secret)
  totp.verify(code, drift_behind: 15, drift_ahead: 15)
end

def otp_provisioning_uri
  issuer = BaseX.config.app_name
  ROTP::TOTP.new(otp_secret).provisioning_uri(email, issuer: issuer)
end

def otp_qr_code
  RQRCode::QRCode.new(otp_provisioning_uri)
end

private

def generate_backup_codes
  10.times.map { SecureRandom.hex(4) }
end
```

Create `app/controllers/two_factor_controller.rb`:

```ruby
class TwoFactorController < ApplicationController
  before_action :authenticate_user!

  def show
    # Display 2FA setup page
  end

  def create
    current_user.enable_two_factor!
    @qr_code = current_user.otp_qr_code
    @backup_codes = current_user.otp_backup_codes
    render :setup
  end

  def verify
    if current_user.verify_otp(params[:otp_code])
      current_user.update!(otp_required_for_login: true)
      redirect_to settings_path, notice: "Two-factor authentication enabled."
    else
      flash.now[:alert] = "Invalid code. Please try again."
      render :setup
    end
  end

  def destroy
    if current_user.verify_otp(params[:otp_code])
      current_user.disable_two_factor!
      redirect_to settings_path, notice: "Two-factor authentication disabled."
    else
      redirect_to two_factor_path, alert: "Invalid code."
    end
  end
end
```

Create a custom Devise strategy for 2FA in `config/initializers/devise.rb` — after Devise signs the user in, check if 2FA is required and redirect to the OTP verification page if so.

## Step 3: OmniAuth Social Login

Create `app/controllers/users/omniauth_callbacks_controller.rb`:

```ruby
class Users::OmniauthCallbacksController < Devise::OmniauthCallbacksController
  def google_oauth2
    handle_auth("Google")
  end

  def github
    handle_auth("GitHub")
  end

  private

  def handle_auth(kind)
    user = User.from_omniauth(request.env["omniauth.auth"])

    if user.persisted?
      sign_in_and_redirect user, event: :authentication
      set_flash_message(:notice, :success, kind: kind) if is_navigational_format?
    else
      session["devise.omniauth_data"] = request.env["omniauth.auth"].except(:extra)
      redirect_to new_user_registration_url, alert: user.errors.full_messages.join("\n")
    end
  end
end
```

Add to User model:

```ruby
def self.from_omniauth(auth)
  where(provider: auth.provider, uid: auth.uid).first_or_create do |user|
    user.email = auth.info.email
    user.password = Devise.friendly_token[0, 20]
    user.first_name = auth.info.first_name || auth.info.name&.split(" ")&.first
    user.last_name = auth.info.last_name || auth.info.name&.split(" ")&.last
    user.avatar_url = auth.info.image
    user.confirmed_at = Time.current  # Trust OAuth provider
  end
end
```

Configure providers in `config/initializers/devise.rb`:

```ruby
config.omniauth :google_oauth2,
  ENV["GOOGLE_CLIENT_ID"],
  ENV["GOOGLE_CLIENT_SECRET"],
  scope: "email,profile"

config.omniauth :github,
  ENV["GITHUB_CLIENT_ID"],
  ENV["GITHUB_CLIENT_SECRET"],
  scope: "user:email"
```

## Step 4: Impersonation

Add `pretender` to enable admins to impersonate users for troubleshooting.

In `ApplicationController`:

```ruby
class ApplicationController < ActionController::Base
  impersonates :user

  # Add a before_action to stop non-admins from impersonating
  def authorize_impersonation!
    redirect_to root_path, alert: "Not authorized" unless true_user&.admin?
  end
end
```

Create `app/controllers/admin/impersonations_controller.rb`:

```ruby
module Admin
  class ImpersonationsController < Admin::BaseController
    def create
      user = User.find(params[:user_id])
      impersonate_user(user)
      redirect_to root_path, notice: "Now impersonating #{user.name}"
    end

    def destroy
      stop_impersonating_user
      redirect_to admin_root_path, notice: "Stopped impersonating"
    end
  end
end
```

Add an impersonation banner partial in `app/views/shared/_impersonation_banner.html.erb`:

```erb
<% if current_user != true_user %>
  <div class="bg-yellow-400 text-yellow-900 text-center py-2 text-sm font-medium">
    You are impersonating <%= current_user.name %>.
    <%= link_to "Stop", admin_impersonation_path, method: :delete,
        class: "underline font-bold ml-2" %>
  </div>
<% end %>
```

## Step 5: Pundit Authorization

Run `rails generate pundit:install` to create `app/policies/application_policy.rb`.

Create team-aware base policy:

```ruby
class ApplicationPolicy
  attr_reader :user, :record

  def initialize(user, record)
    @user = user
    @record = record
  end

  def membership
    @membership ||= user.memberships.find_by(team: ActsAsTenant.current_tenant)
  end

  def admin?
    membership&.admin? || user.admin?
  end

  def member?
    membership.present?
  end

  def owner?
    admin?
  end
end
```

Example `TeamPolicy`:

```ruby
class TeamPolicy < ApplicationPolicy
  def show?
    member?
  end

  def update?
    admin?
  end

  def destroy?
    admin? && !record.personal?
  end
end
```

Example `MembershipPolicy`:

```ruby
class MembershipPolicy < ApplicationPolicy
  def destroy?
    admin? || record.user == user
  end

  def update?
    admin?
  end
end
```

## Step 6: Pay Gem (Billing)

Add to Gemfile:

```ruby
gem "pay", "~> 7.0"
gem "stripe", "~> 12.0"
```

Run:

```bash
rails generate pay:install
rails db:migrate
```

The Pay gem adds `pay_customers`, `pay_subscriptions`, `pay_charges`, etc.

Make Team the billable model:

```ruby
class Team < ApplicationRecord
  pay_customer

  # ... existing code ...

  def on_trial?
    pay_subscriptions.where("trial_ends_at > ?", Time.current).exists?
  end

  def subscribed?
    pay_subscriptions.active.exists?
  end

  def subscription
    pay_subscriptions.active.last
  end

  def plan
    return nil unless subscription
    Plan.find_by(stripe_id: subscription.processor_plan)
  end
end
```

Create `config/initializers/pay.rb`:

```ruby
Pay.setup do |config|
  config.application_name = BaseX.config.app_name
  config.business_name = BaseX.config.app_name
  config.business_address = "123 Main St"
  config.support_email = BaseX.config.support_email
end
```

Mount the Pay webhook engine in `config/routes.rb`:

```ruby
mount Pay::Engine, at: "/pay"
```

Create `app/controllers/billings_controller.rb`:

```ruby
class BillingsController < ApplicationController
  before_action :authenticate_user!

  def show
    @team = current_team
    @plans = Plan.visible
    @subscription = @team.subscription
  end

  def create
    plan = Plan.find(params[:plan_id])
    checkout = current_team.payment_processor.checkout(
      mode: "subscription",
      line_items: plan.stripe_id,
      success_url: billing_url(session_id: "{CHECKOUT_SESSION_ID}"),
      cancel_url: billing_url
    )
    redirect_to checkout.url, allow_other_host: true
  end

  def portal
    portal_session = current_team.payment_processor.billing_portal(
      return_url: billing_url
    )
    redirect_to portal_session.url, allow_other_host: true
  end
end
```

## Routes for Phase 2

Add to `config/routes.rb`:

```ruby
Rails.application.routes.draw do
  devise_for :users, controllers: {
    omniauth_callbacks: "users/omniauth_callbacks"
  }

  resource :two_factor, only: [:show, :create, :destroy] do
    post :verify
  end

  resource :billing, only: [:show, :create] do
    post :portal
  end

  mount Pay::Engine, at: "/pay"
end
```

## Environment Variables

The user's `.env` file should already contain the Stripe and OAuth keys (set up in the Pre-Build step). Do NOT attempt to write to `.env` or `.env.example`. If keys are missing, ask the user to add them manually.
