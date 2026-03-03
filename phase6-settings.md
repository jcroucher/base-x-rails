# Phase 6: User Settings & Account Management

## Step 1: Settings Controller

Create `app/controllers/settings_controller.rb` — a tabbed settings page with Profile, Password, Connected Accounts, and Danger Zone sections:

```ruby
class SettingsController < ApplicationController
  before_action :authenticate_user!

  def show
  end

  def update
    if current_user.update(settings_params)
      redirect_to settings_path, notice: "Settings updated."
    else
      render :show, status: :unprocessable_entity
    end
  end

  private

  def settings_params
    params.require(:user).permit(:first_name, :last_name, :time_zone)
  end
end
```

## Step 2: Tabs Stimulus Controller

Create `app/javascript/controllers/tabs_controller.js` for switching settings panels without page reload:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["tab", "panel"]
  static values = { index: { type: Number, default: 0 } }

  connect() {
    this.showTab(this.indexValue)
  }

  select(event) {
    const index = this.tabTargets.indexOf(event.currentTarget)
    this.showTab(index)
  }

  showTab(index) {
    this.tabTargets.forEach((tab, i) => {
      tab.classList.toggle("border-primary-500", i === index)
      tab.classList.toggle("text-primary-600", i === index)
      tab.classList.toggle("border-transparent", i !== index)
      tab.classList.toggle("text-gray-500", i !== index)
    })

    this.panelTargets.forEach((panel, i) => {
      panel.classList.toggle("hidden", i !== index)
    })

    this.indexValue = index
  }
}
```

## Step 3: Timezone Selector

Use `ActiveSupport::TimeZone.all` for a `<select>` dropdown. This is included in the settings profile partial (Step 10).

## Step 4: Passwords Controller

Create `app/controllers/passwords_controller.rb` for authenticated password change:

```ruby
class PasswordsController < ApplicationController
  before_action :authenticate_user!

  def update
    if current_user.valid_password?(params[:user][:current_password])
      if current_user.update(password_params)
        bypass_sign_in(current_user)
        redirect_to settings_path, notice: "Password updated."
      else
        redirect_to settings_path, alert: current_user.errors.full_messages.join(", ")
      end
    else
      redirect_to settings_path, alert: "Current password is incorrect."
    end
  end

  private

  def password_params
    params.require(:user).permit(:password, :password_confirmation)
  end
end
```

## Step 5: Connected Account Model

Create a model for multi-provider OAuth. This extracts `provider`/`uid` from User so users can link multiple OAuth providers.

```bash
rails generate model ConnectedAccount \
  user:references \
  provider:string \
  uid:string \
  access_token:string \
  refresh_token:string \
  expires_at:datetime
```

Add a unique index on `[provider, uid]` in the migration:

```ruby
class CreateConnectedAccounts < ActiveRecord::Migration[8.1]
  def change
    create_table :connected_accounts do |t|
      t.references :user, null: false, foreign_key: true
      t.string :provider, null: false
      t.string :uid, null: false
      t.string :access_token
      t.string :refresh_token
      t.datetime :expires_at

      t.timestamps
    end

    add_index :connected_accounts, [:provider, :uid], unique: true
  end
end
```

```ruby
# app/models/connected_account.rb
class ConnectedAccount < ApplicationRecord
  belongs_to :user

  encrypts :access_token, deterministic: false
  encrypts :refresh_token, deterministic: false

  validates :provider, presence: true
  validates :uid, presence: true, uniqueness: { scope: :provider }

  def self.for_oauth(auth)
    find_or_initialize_by(provider: auth.provider, uid: auth.uid).tap do |account|
      account.access_token = auth.credentials.token
      account.refresh_token = auth.credentials.refresh_token
      account.expires_at = auth.credentials.expires_at ? Time.at(auth.credentials.expires_at) : nil
    end
  end
end
```

Add to User model:

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_many :connected_accounts, dependent: :destroy
  # ... existing code ...
end
```

## Step 6: Update OmniAuth Callbacks Controller

Update `app/controllers/users/omniauth_callbacks_controller.rb` to use the ConnectedAccount model:

```ruby
class Users::OmniauthCallbacksController < Devise::OmniauthCallbacksController
  def google_oauth2
    handle_oauth("Google")
  end

  def github
    handle_oauth("GitHub")
  end

  private

  def handle_oauth(provider_name)
    auth = request.env["omniauth.auth"]

    # Check if this OAuth account is already linked to a user
    connected_account = ConnectedAccount.find_by(provider: auth.provider, uid: auth.uid)

    if current_user
      # Signed-in user linking a new provider
      if connected_account && connected_account.user != current_user
        redirect_to settings_path, alert: "This #{provider_name} account is linked to another user."
        return
      end

      account = ConnectedAccount.for_oauth(auth)
      account.user = current_user
      account.save!
      redirect_to settings_path, notice: "#{provider_name} account connected."
    elsif connected_account
      # Returning user signing in via OAuth
      connected_account.update!(
        access_token: auth.credentials.token,
        refresh_token: auth.credentials.refresh_token
      )
      sign_in_and_redirect connected_account.user, event: :authentication
      set_flash_message(:notice, :success, kind: provider_name) if is_navigational_format?
    else
      # New user signing up via OAuth
      user = User.find_by(email: auth.info.email)

      if user
        # Existing user, link account
        account = ConnectedAccount.for_oauth(auth)
        account.user = user
        account.save!
        sign_in_and_redirect user, event: :authentication
      else
        # Brand new user
        user = User.new(
          email: auth.info.email,
          first_name: auth.info.first_name || auth.info.name&.split(" ")&.first,
          last_name: auth.info.last_name || auth.info.name&.split(" ")&.last,
          password: Devise.friendly_token[0, 20],
          confirmed_at: Time.current
        )
        user.save!

        account = ConnectedAccount.for_oauth(auth)
        account.user = user
        account.save!

        sign_in_and_redirect user, event: :authentication
        set_flash_message(:notice, :success, kind: provider_name) if is_navigational_format?
      end
    end
  end
end
```

## Step 7: Connected Accounts Controller

Create `app/controllers/connected_accounts_controller.rb` for disconnecting OAuth providers:

```ruby
class ConnectedAccountsController < ApplicationController
  before_action :authenticate_user!

  def destroy
    account = current_user.connected_accounts.find(params[:id])

    if current_user.encrypted_password.blank? && current_user.connected_accounts.count <= 1
      redirect_to settings_path, alert: "You must set a password before disconnecting your only login provider."
      return
    end

    account.destroy
    redirect_to settings_path, notice: "#{account.provider.titleize} account disconnected."
  end
end
```

## Step 8: Terms Gate Concern

Create `app/controllers/concerns/terms_gate.rb` — redirects users to `/terms` if they haven't accepted terms:

```ruby
module TermsGate
  extend ActiveSupport::Concern

  included do
    before_action :require_accepted_terms
  end

  private

  def require_accepted_terms
    return unless user_signed_in?
    return if controller_name.in?(%w[setup terms])
    return if self.class.name.start_with?("Devise::")
    return if request.path.start_with?("/rails/")
    return if request.path.start_with?("/admin")

    if current_user.accepted_terms_at.blank?
      redirect_to terms_path, alert: "Please accept the terms of service to continue."
    end
  end
end
```

Include the concern in `ApplicationController` (add after the existing includes):

```ruby
class ApplicationController < ActionController::Base
  include SetCurrentTeam
  include Authentication
  include Authorization
  include SetupGuard
  include TermsGate
end
```

## Step 9: Terms Controller

Create `app/controllers/terms_controller.rb`:

```ruby
class TermsController < ApplicationController
  before_action :authenticate_user!

  skip_before_action :require_accepted_terms

  def show
  end

  def accept
    current_user.update!(accepted_terms_at: Time.current)
    redirect_to dashboard_path, notice: "Terms accepted. Welcome!"
  end
end
```

Create `app/views/terms/show.html.erb`:

```erb
<% content_for(:title, "Terms of Service") %>

<div class="max-w-3xl mx-auto py-12 px-4 sm:px-6 lg:px-8">
  <h1 class="text-3xl font-bold text-gray-900 mb-6">Terms of Service</h1>

  <div class="prose prose-gray max-w-none mb-8">
    <p>
      By using <%= BaseX.config.app_name %>, you agree to the following terms and conditions.
      Please read them carefully before continuing.
    </p>

    <h2>1. Acceptance of Terms</h2>
    <p>
      By accessing or using this application, you agree to be bound by these Terms of Service
      and all applicable laws and regulations.
    </p>

    <h2>2. Use of Service</h2>
    <p>
      You agree to use the service only for lawful purposes and in accordance with these Terms.
      You are responsible for maintaining the confidentiality of your account credentials.
    </p>

    <h2>3. Privacy</h2>
    <p>
      Your use of the service is also governed by our Privacy Policy. Please review our
      Privacy Policy to understand our practices.
    </p>

    <h2>4. Termination</h2>
    <p>
      We may terminate or suspend your account at any time, without prior notice or liability,
      for any reason, including breach of these Terms.
    </p>
  </div>

  <div class="border-t border-gray-200 pt-6">
    <%= button_to "I Accept the Terms of Service",
        accept_terms_path,
        method: :post,
        class: "inline-flex items-center px-6 py-3 border border-transparent text-base font-medium rounded-md shadow-sm text-white bg-primary-600 hover:bg-primary-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-primary-500" %>
  </div>
</div>
```

## Step 10: Account Deletions Controller

Create `app/controllers/account_deletions_controller.rb`:

```ruby
class AccountDeletionsController < ApplicationController
  before_action :authenticate_user!

  def create
    unless current_user.valid_password?(params[:password])
      redirect_to settings_path, alert: "Incorrect password. Account not deleted."
      return
    end

    current_user.soft_delete!
    sign_out(current_user)
    redirect_to root_path, notice: "Your account has been deleted."
  end
end
```

## Step 11: Settings Views

Create `app/views/settings/show.html.erb`:

```erb
<% content_for(:title, "Settings") %>

<div class="max-w-4xl mx-auto py-8 px-4 sm:px-6 lg:px-8">
  <h1 class="text-2xl font-bold text-gray-900 mb-6">Settings</h1>

  <div data-controller="tabs">
    <!-- Tab Navigation -->
    <div class="border-b border-gray-200">
      <nav class="-mb-px flex space-x-8" aria-label="Settings tabs">
        <button type="button" data-tabs-target="tab" data-action="click->tabs#select"
                class="border-primary-500 text-primary-600 whitespace-nowrap border-b-2 py-4 px-1 text-sm font-medium">
          Profile
        </button>
        <button type="button" data-tabs-target="tab" data-action="click->tabs#select"
                class="border-transparent text-gray-500 hover:border-gray-300 hover:text-gray-700 whitespace-nowrap border-b-2 py-4 px-1 text-sm font-medium">
          Password
        </button>
        <button type="button" data-tabs-target="tab" data-action="click->tabs#select"
                class="border-transparent text-gray-500 hover:border-gray-300 hover:text-gray-700 whitespace-nowrap border-b-2 py-4 px-1 text-sm font-medium">
          Connected Accounts
        </button>
        <button type="button" data-tabs-target="tab" data-action="click->tabs#select"
                class="border-transparent text-gray-500 hover:border-gray-300 hover:text-gray-700 whitespace-nowrap border-b-2 py-4 px-1 text-sm font-medium">
          Danger Zone
        </button>
      </nav>
    </div>

    <!-- Tab Panels -->
    <div class="mt-6">
      <div data-tabs-target="panel">
        <%= render "settings/profile" %>
      </div>

      <div data-tabs-target="panel" class="hidden">
        <%= render "settings/password" %>
      </div>

      <div data-tabs-target="panel" class="hidden">
        <%= render "settings/connected_accounts" %>
      </div>

      <div data-tabs-target="panel" class="hidden">
        <%= render "settings/danger_zone" %>
      </div>
    </div>
  </div>
</div>
```

Create `app/views/settings/_profile.html.erb`:

```erb
<div class="bg-white shadow rounded-lg p-6">
  <h2 class="text-lg font-medium text-gray-900 mb-4">Profile</h2>

  <%= form_with(model: current_user, url: settings_path, method: :patch, class: "space-y-4") do |f| %>
    <% if current_user.errors.any? %>
      <div class="rounded-md bg-red-50 border border-red-200 p-4">
        <ul class="list-disc list-inside text-sm text-red-700">
          <% current_user.errors.full_messages.each do |message| %>
            <li><%= message %></li>
          <% end %>
        </ul>
      </div>
    <% end %>

    <div class="grid grid-cols-1 gap-4 sm:grid-cols-2">
      <div>
        <%= f.label :first_name, class: "block text-sm font-medium text-gray-700" %>
        <%= f.text_field :first_name,
            class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500 sm:text-sm" %>
      </div>
      <div>
        <%= f.label :last_name, class: "block text-sm font-medium text-gray-700" %>
        <%= f.text_field :last_name,
            class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500 sm:text-sm" %>
      </div>
    </div>

    <div>
      <%= f.label :time_zone, class: "block text-sm font-medium text-gray-700" %>
      <%= f.time_zone_select :time_zone,
          ActiveSupport::TimeZone.all,
          { default: current_user.time_zone || "UTC" },
          class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500 sm:text-sm" %>
    </div>

    <div class="pt-2">
      <%= f.submit "Save Changes",
          class: "inline-flex items-center px-4 py-2 border border-transparent text-sm font-medium rounded-md shadow-sm text-white bg-primary-600 hover:bg-primary-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-primary-500 cursor-pointer" %>
    </div>
  <% end %>
</div>
```

Create `app/views/settings/_password.html.erb`:

```erb
<div class="bg-white shadow rounded-lg p-6">
  <h2 class="text-lg font-medium text-gray-900 mb-4">Change Password</h2>

  <%= form_with(url: password_path, method: :patch, class: "space-y-4") do |f| %>
    <div>
      <%= f.label :current_password, class: "block text-sm font-medium text-gray-700" %>
      <%= f.password_field :current_password, name: "user[current_password]", required: true, autocomplete: "current-password",
          class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500 sm:text-sm" %>
    </div>

    <div>
      <%= f.label :password, "New Password", class: "block text-sm font-medium text-gray-700" %>
      <%= f.password_field :password, name: "user[password]", required: true, autocomplete: "new-password",
          class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500 sm:text-sm" %>
    </div>

    <div>
      <%= f.label :password_confirmation, class: "block text-sm font-medium text-gray-700" %>
      <%= f.password_field :password_confirmation, name: "user[password_confirmation]", required: true, autocomplete: "new-password",
          class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500 sm:text-sm" %>
    </div>

    <div class="pt-2">
      <%= f.submit "Update Password",
          class: "inline-flex items-center px-4 py-2 border border-transparent text-sm font-medium rounded-md shadow-sm text-white bg-primary-600 hover:bg-primary-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-primary-500 cursor-pointer" %>
    </div>
  <% end %>
</div>
```

Create `app/views/settings/_connected_accounts.html.erb`:

```erb
<div class="bg-white shadow rounded-lg p-6">
  <h2 class="text-lg font-medium text-gray-900 mb-4">Connected Accounts</h2>

  <% if current_user.connected_accounts.any? %>
    <div class="divide-y divide-gray-200">
      <% current_user.connected_accounts.each do |account| %>
        <div class="flex items-center justify-between py-4">
          <div class="flex items-center">
            <div class="flex-shrink-0">
              <% case account.provider %>
              <% when "google_oauth2" %>
                <span class="inline-flex items-center justify-center w-10 h-10 rounded-full bg-red-100 text-red-600 font-bold">G</span>
              <% when "github" %>
                <span class="inline-flex items-center justify-center w-10 h-10 rounded-full bg-gray-800 text-white font-bold">GH</span>
              <% else %>
                <span class="inline-flex items-center justify-center w-10 h-10 rounded-full bg-gray-100 text-gray-600 font-bold"><%= account.provider.first.upcase %></span>
              <% end %>
            </div>
            <div class="ml-4">
              <p class="text-sm font-medium text-gray-900"><%= account.provider.titleize.gsub("Oauth2", "").strip %></p>
              <p class="text-sm text-gray-500">Connected <%= time_ago_in_words(account.created_at) %> ago</p>
            </div>
          </div>

          <%= button_to "Disconnect",
              connected_account_path(account),
              method: :delete,
              class: "text-sm text-red-600 hover:text-red-800 font-medium",
              data: { turbo_confirm: "Are you sure you want to disconnect this account?" } %>
        </div>
      <% end %>
    </div>
  <% else %>
    <p class="text-sm text-gray-500">No connected accounts. Sign in with a social provider to link it to your account.</p>
  <% end %>
</div>
```

Create `app/views/settings/_danger_zone.html.erb`:

```erb
<div class="bg-white shadow rounded-lg border border-red-200 p-6">
  <h2 class="text-lg font-medium text-red-900 mb-2">Danger Zone</h2>
  <p class="text-sm text-gray-600 mb-6">
    Once you delete your account, there is no going back. Please be certain.
  </p>

  <details class="group">
    <summary class="cursor-pointer text-sm font-medium text-red-600 hover:text-red-800">
      Delete my account...
    </summary>

    <div class="mt-4 p-4 bg-red-50 rounded-md">
      <%= form_with(url: account_deletion_path, method: :post, class: "space-y-4") do |f| %>
        <p class="text-sm text-red-700">
          Enter your password to confirm account deletion. This action is irreversible.
        </p>

        <div>
          <%= f.label :password, "Confirm Password", class: "block text-sm font-medium text-red-700" %>
          <%= f.password_field :password, required: true,
              class: "mt-1 block w-full rounded-md border-red-300 shadow-sm focus:border-red-500 focus:ring-red-500 sm:text-sm" %>
        </div>

        <%= f.submit "Permanently Delete Account",
            class: "inline-flex items-center px-4 py-2 border border-transparent text-sm font-medium rounded-md shadow-sm text-white bg-red-600 hover:bg-red-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-red-500 cursor-pointer",
            data: { turbo_confirm: "This will permanently delete your account. Are you absolutely sure?" } %>
      <% end %>
    </div>
  </details>
</div>
```

## Step 12: Routes

Add the following routes to `config/routes.rb`:

```ruby
# Settings & Account
resource :settings, only: [:show, :update], controller: "settings"
resource :password, only: [:update], controller: "passwords"
resources :connected_accounts, only: [:destroy]
resource :terms, only: [:show], controller: "terms" do
  post :accept, on: :collection
end
resource :account_deletion, only: [:create], controller: "account_deletions"
```

### Route Summary

| Path | Method | Purpose | Access |
|------|--------|---------|--------|
| `/settings` | GET | Show settings page | Authenticated |
| `/settings` | PATCH | Update profile | Authenticated |
| `/password` | PATCH | Change password | Authenticated |
| `/connected_accounts/:id` | DELETE | Disconnect OAuth | Authenticated |
| `/terms` | GET | Show terms of service | Authenticated |
| `/terms/accept` | POST | Accept terms | Authenticated |
| `/account_deletion` | POST | Delete account (soft) | Authenticated |
