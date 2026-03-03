# Phase 7: Team Members & Invitations

## Step 1: Memberships Controller

Create `app/controllers/memberships_controller.rb` — list members, change roles, and remove members:

```ruby
class MembershipsController < ApplicationController
  before_action :authenticate_user!

  def index
    authorize Membership
    @memberships = current_team.memberships.includes(:user)
    @pending_invitations = current_team.invitations.pending.order(created_at: :desc)
  end

  def update
    @membership = current_team.memberships.find(params[:id])
    authorize @membership

    if @membership.update(membership_params)
      redirect_to memberships_path, notice: "Role updated."
    else
      redirect_to memberships_path, alert: @membership.errors.full_messages.join(", ")
    end
  end

  def destroy
    @membership = current_team.memberships.find(params[:id])
    authorize @membership

    if @membership.user == current_user
      # Leaving the team
      @membership.destroy
      session.delete(:team_id)
      redirect_to dashboard_path, notice: "You have left the team."
    else
      # Removing a member (admin only)
      @membership.destroy
      redirect_to memberships_path, notice: "Member removed."
    end
  end

  private

  def membership_params
    roles = {}
    roles["admin"] = true if params.dig(:membership, :role) == "admin"
    { roles: roles }
  end
end
```

## Step 2: Invitations Controller

Create `app/controllers/invitations_controller.rb` — send, cancel, and accept invitations:

```ruby
class InvitationsController < ApplicationController
  before_action :authenticate_user!, except: [:show]

  def new
    @invitation = current_team.invitations.build
    authorize @invitation
  end

  def create
    @invitation = current_team.invitations.build(invitation_params)
    @invitation.invited_by = current_user
    authorize @invitation

    if @invitation.save
      InvitationMailer.invite_email(@invitation).deliver_later
      redirect_to memberships_path, notice: "Invitation sent to #{@invitation.email}."
    else
      render :new, status: :unprocessable_entity
    end
  end

  def show
    @invitation = Invitation.pending.find_by!(token: params[:token])

    if user_signed_in?
      accept_invitation(@invitation, current_user)
    else
      # Store token in session so we can accept after sign-in/sign-up
      session[:invitation_token] = @invitation.token
      redirect_to new_user_registration_path, notice: "Create an account or sign in to accept this invitation."
    end
  end

  def destroy
    @invitation = current_team.invitations.find(params[:id])
    authorize @invitation

    @invitation.destroy
    redirect_to memberships_path, notice: "Invitation cancelled."
  end

  private

  def invitation_params
    params.require(:invitation).permit(:email, :name, :role)
  end

  def accept_invitation(invitation, user)
    if invitation.team.users.include?(user)
      redirect_to dashboard_path, notice: "You're already a member of this team."
      return
    end

    roles = {}
    roles["admin"] = true if invitation.roles&.dig("admin")

    invitation.accept!(user)

    session[:team_id] = invitation.team.id
    redirect_to dashboard_path, notice: "Welcome to #{invitation.team.name}!"
  end
end
```

## Step 3: Invitation Acceptance After Sign-Up

Update `ApplicationController` to check for pending invitation tokens after sign-in. Override `after_sign_in_path_for`:

```ruby
# Add to app/controllers/application_controller.rb

def after_sign_in_path_for(resource)
  if session[:invitation_token].present?
    invitation = Invitation.pending.find_by(token: session[:invitation_token])
    if invitation
      session.delete(:invitation_token)
      invitation.accept!(resource)
      session[:team_id] = invitation.team.id
      return dashboard_path
    end
  end

  super
end
```

## Step 4: Invitation Mailer

Create `app/mailers/invitation_mailer.rb`:

```ruby
class InvitationMailer < ApplicationMailer
  def invite_email(invitation)
    @invitation = invitation
    @team = invitation.team
    @inviter = invitation.invited_by
    @accept_url = invitation_accept_url(token: invitation.token)

    mail(
      to: invitation.email,
      subject: "You've been invited to join #{@team.name} on #{BaseX.config.app_name}"
    )
  end
end
```

Create `app/views/invitation_mailer/invite_email.html.erb`:

```erb
<h2>You're invited!</h2>

<p>
  <strong><%= @inviter.name %></strong> has invited you to join
  <strong><%= @team.name %></strong> on <%= BaseX.config.app_name %>.
</p>

<p>
  <%= link_to "Accept Invitation", @accept_url,
      style: "display: inline-block; padding: 12px 24px; background-color: #4F46E5; color: #ffffff; text-decoration: none; border-radius: 6px; font-weight: 600;" %>
</p>

<p style="color: #6B7280; font-size: 14px;">
  If you weren't expecting this invitation, you can safely ignore this email.
</p>
```

Create `app/views/invitation_mailer/invite_email.text.erb`:

```erb
You're invited!

<%= @inviter.name %> has invited you to join <%= @team.name %> on <%= BaseX.config.app_name %>.

Accept the invitation by visiting:
<%= @accept_url %>

If you weren't expecting this invitation, you can safely ignore this email.
```

## Step 5: Invitation Policy

Create `app/policies/invitation_policy.rb`:

```ruby
class InvitationPolicy < ApplicationPolicy
  def new?
    admin?
  end

  def create?
    admin?
  end

  def destroy?
    admin?
  end

  private

  def admin?
    user_membership&.admin?
  end

  def user_membership
    record.team.memberships.find_by(user: user)
  end
end
```

## Step 6: Update Membership Policy

Update `app/policies/membership_policy.rb` to add `index?`:

```ruby
class MembershipPolicy < ApplicationPolicy
  def index?
    member?
  end

  def update?
    admin? && record.user != user
  end

  def destroy?
    admin? || record.user == user
  end

  class Scope < Scope
    def resolve
      scope.where(team: user.teams)
    end
  end

  private

  def admin?
    current_membership&.admin?
  end

  def member?
    current_membership.present?
  end

  def current_membership
    @current_membership ||= record.respond_to?(:team) ?
      record.team.memberships.find_by(user: user) :
      user.memberships.find_by(team: ActsAsTenant.current_tenant)
  end
end
```

## Step 7: Team Settings Controller

Create `app/controllers/team_settings_controller.rb` — admin-only team name and billing info:

```ruby
class TeamSettingsController < ApplicationController
  before_action :authenticate_user!
  before_action :require_admin!

  def show
  end

  def update
    if current_team.update(team_settings_params)
      redirect_to team_settings_path, notice: "Team settings updated."
    else
      render :show, status: :unprocessable_entity
    end
  end

  private

  def team_settings_params
    params.require(:team).permit(:name, :extra_billing_info)
  end

  def require_admin!
    unless current_membership&.admin?
      redirect_to dashboard_path, alert: "Only admins can manage team settings."
    end
  end
end
```

## Step 8: Memberships Index View

Create `app/views/memberships/index.html.erb`:

```erb
<% content_for(:title, "Team Members") %>

<div class="max-w-4xl mx-auto py-8 px-4 sm:px-6 lg:px-8">
  <div class="flex items-center justify-between mb-6">
    <h1 class="text-2xl font-bold text-gray-900">Team Members</h1>
    <% if current_membership&.admin? %>
      <%= link_to "Invite Member", new_invitation_path,
          class: "inline-flex items-center px-4 py-2 border border-transparent text-sm font-medium rounded-md shadow-sm text-white bg-primary-600 hover:bg-primary-700" %>
    <% end %>
  </div>

  <!-- Current Members -->
  <div class="bg-white shadow rounded-lg overflow-hidden">
    <table class="min-w-full divide-y divide-gray-200">
      <thead class="bg-gray-50">
        <tr>
          <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Member</th>
          <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Role</th>
          <th class="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">Actions</th>
        </tr>
      </thead>
      <tbody class="bg-white divide-y divide-gray-200">
        <% @memberships.each do |membership| %>
          <tr>
            <td class="px-6 py-4 whitespace-nowrap">
              <div class="flex items-center">
                <div class="flex-shrink-0 h-10 w-10 rounded-full bg-primary-100 flex items-center justify-center">
                  <span class="text-primary-700 font-medium text-sm">
                    <%= membership.user.first_name&.first %><%= membership.user.last_name&.first %>
                  </span>
                </div>
                <div class="ml-4">
                  <div class="text-sm font-medium text-gray-900"><%= membership.user.name %></div>
                  <div class="text-sm text-gray-500"><%= membership.user.email %></div>
                </div>
              </div>
            </td>
            <td class="px-6 py-4 whitespace-nowrap">
              <% if current_membership&.admin? && membership.user != current_user %>
                <%= form_with(model: membership, url: membership_path(membership), method: :patch, class: "inline") do |f| %>
                  <%= f.select :role,
                      [["Member", "member"], ["Admin", "admin"]],
                      { selected: membership.admin? ? "admin" : "member" },
                      class: "text-sm rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500",
                      onchange: "this.form.requestSubmit()" %>
                <% end %>
              <% else %>
                <span class="inline-flex items-center rounded-full px-2.5 py-0.5 text-xs font-medium <%= membership.admin? ? 'bg-purple-100 text-purple-800' : 'bg-gray-100 text-gray-800' %>">
                  <%= membership.admin? ? "Admin" : "Member" %>
                </span>
              <% end %>
            </td>
            <td class="px-6 py-4 whitespace-nowrap text-right text-sm font-medium">
              <% if membership.user == current_user && !current_team.personal? %>
                <%= button_to "Leave",
                    membership_path(membership),
                    method: :delete,
                    class: "text-red-600 hover:text-red-900",
                    data: { turbo_confirm: "Are you sure you want to leave this team?" } %>
              <% elsif current_membership&.admin? && membership.user != current_user %>
                <%= button_to "Remove",
                    membership_path(membership),
                    method: :delete,
                    class: "text-red-600 hover:text-red-900",
                    data: { turbo_confirm: "Are you sure you want to remove #{membership.user.name}?" } %>
              <% end %>
            </td>
          </tr>
        <% end %>
      </tbody>
    </table>
  </div>

  <!-- Pending Invitations -->
  <% if @pending_invitations.any? %>
    <h2 class="text-lg font-medium text-gray-900 mt-8 mb-4">Pending Invitations</h2>
    <div class="bg-white shadow rounded-lg overflow-hidden">
      <table class="min-w-full divide-y divide-gray-200">
        <thead class="bg-gray-50">
          <tr>
            <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Email</th>
            <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Invited</th>
            <th class="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">Actions</th>
          </tr>
        </thead>
        <tbody class="bg-white divide-y divide-gray-200">
          <% @pending_invitations.each do |invitation| %>
            <tr>
              <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-900"><%= invitation.email %></td>
              <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500"><%= time_ago_in_words(invitation.created_at) %> ago</td>
              <td class="px-6 py-4 whitespace-nowrap text-right text-sm font-medium">
                <% if current_membership&.admin? %>
                  <%= button_to "Cancel",
                      invitation_path(invitation),
                      method: :delete,
                      class: "text-red-600 hover:text-red-900" %>
                <% end %>
              </td>
            </tr>
          <% end %>
        </tbody>
      </table>
    </div>
  <% end %>
</div>
```

## Step 9: Invitation New View

Create `app/views/invitations/new.html.erb`:

```erb
<% content_for(:title, "Invite Member") %>

<div class="max-w-lg mx-auto py-8 px-4 sm:px-6 lg:px-8">
  <h1 class="text-2xl font-bold text-gray-900 mb-6">Invite a Team Member</h1>

  <div class="bg-white shadow rounded-lg p-6">
    <%= form_with(model: @invitation, url: invitations_path, class: "space-y-4") do |f| %>
      <% if @invitation.errors.any? %>
        <div class="rounded-md bg-red-50 border border-red-200 p-4">
          <ul class="list-disc list-inside text-sm text-red-700">
            <% @invitation.errors.full_messages.each do |message| %>
              <li><%= message %></li>
            <% end %>
          </ul>
        </div>
      <% end %>

      <div>
        <%= f.label :email, class: "block text-sm font-medium text-gray-700" %>
        <%= f.email_field :email, required: true, autocomplete: "email",
            class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500 sm:text-sm",
            placeholder: "colleague@example.com" %>
      </div>

      <div>
        <%= f.label :name, class: "block text-sm font-medium text-gray-700" %>
        <%= f.text_field :name,
            class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500 sm:text-sm",
            placeholder: "Their name (optional)" %>
      </div>

      <div>
        <%= f.label :role, class: "block text-sm font-medium text-gray-700" %>
        <%= f.select :role,
            [["Member", "member"], ["Admin", "admin"]],
            {},
            class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500 sm:text-sm" %>
      </div>

      <div class="pt-2">
        <%= f.submit "Send Invitation",
            class: "w-full flex justify-center py-2.5 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white bg-primary-600 hover:bg-primary-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-primary-500 cursor-pointer" %>
      </div>
    <% end %>
  </div>
</div>
```

## Step 10: Team Settings View

Create `app/views/team_settings/show.html.erb`:

```erb
<% content_for(:title, "Team Settings") %>

<div class="max-w-2xl mx-auto py-8 px-4 sm:px-6 lg:px-8">
  <h1 class="text-2xl font-bold text-gray-900 mb-6">Team Settings</h1>

  <div class="bg-white shadow rounded-lg p-6">
    <%= form_with(model: current_team, url: team_settings_path, method: :patch, class: "space-y-4") do |f| %>
      <% if current_team.errors.any? %>
        <div class="rounded-md bg-red-50 border border-red-200 p-4">
          <ul class="list-disc list-inside text-sm text-red-700">
            <% current_team.errors.full_messages.each do |message| %>
              <li><%= message %></li>
            <% end %>
          </ul>
        </div>
      <% end %>

      <div>
        <%= f.label :name, "Team Name", class: "block text-sm font-medium text-gray-700" %>
        <%= f.text_field :name, required: true,
            class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500 sm:text-sm" %>
      </div>

      <div>
        <%= f.label :extra_billing_info, "Billing Info", class: "block text-sm font-medium text-gray-700" %>
        <p class="text-xs text-gray-500 mt-1">This will appear on your invoices (e.g. VAT number, company address).</p>
        <%= f.text_area :extra_billing_info, rows: 4,
            class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500 sm:text-sm" %>
      </div>

      <div class="pt-2">
        <%= f.submit "Save Settings",
            class: "inline-flex items-center px-4 py-2 border border-transparent text-sm font-medium rounded-md shadow-sm text-white bg-primary-600 hover:bg-primary-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-primary-500 cursor-pointer" %>
      </div>
    <% end %>
  </div>
</div>
```

## Step 11: Routes

Add the following routes to `config/routes.rb`:

```ruby
# Team management
resources :memberships, only: [:index, :update, :destroy]
resources :invitations, only: [:new, :create, :destroy]
resource :team_settings, only: [:show, :update], controller: "team_settings"

# Public invitation acceptance (no auth required)
get "invitations/:token", to: "invitations#show", as: :invitation_accept
```

### Route Summary

| Path | Method | Purpose | Access |
|------|--------|---------|--------|
| `/memberships` | GET | List team members | Authenticated (member) |
| `/memberships/:id` | PATCH | Change member role | Authenticated (admin) |
| `/memberships/:id` | DELETE | Remove member / leave team | Authenticated (admin or self) |
| `/invitations/new` | GET | Invite form | Authenticated (admin) |
| `/invitations` | POST | Send invitation | Authenticated (admin) |
| `/invitations/:id` | DELETE | Cancel invitation | Authenticated (admin) |
| `/invitations/:token` | GET | Accept invitation | Public |
| `/team_settings` | GET | Team settings form | Authenticated (admin) |
| `/team_settings` | PATCH | Update team settings | Authenticated (admin) |
