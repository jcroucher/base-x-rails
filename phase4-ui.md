# Phase 4: UI & Frontend

## Step 1: Tailwind CSS 4.0 Configuration

Rails 8.1 with `--css=tailwind` sets up Tailwind automatically. Ensure the config supports the component library:

Create `app/assets/stylesheets/application.css` with Tailwind directives and custom base styles:

```css
@import "tailwindcss";

/* Custom color theme — adjust to match branding */
@theme {
  --color-primary-50: #eff6ff;
  --color-primary-100: #dbeafe;
  --color-primary-200: #bfdbfe;
  --color-primary-300: #93c5fd;
  --color-primary-400: #60a5fa;
  --color-primary-500: #3b82f6;
  --color-primary-600: #2563eb;
  --color-primary-700: #1d4ed8;
  --color-primary-800: #1e40af;
  --color-primary-900: #1e3a8a;
}
```

## Step 2: Layout System

Create three layout files for the different sections of the app.

### Authenticated Layout (`app/views/layouts/application.html.erb`)

This is the main app layout for signed-in users.

```erb
<!DOCTYPE html>
<html lang="en" class="h-full bg-gray-50">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title><%= content_for(:title) || BaseX.config.app_name %></title>
  <meta name="turbo-cache-control" content="no-preview">
  <%= csrf_meta_tags %>
  <%= csp_meta_tag %>
  <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
  <%= javascript_importmap_tags %>
</head>
<body class="h-full">
  <%= render "shared/impersonation_banner" if current_user != true_user rescue nil %>
  <%= render "shared/announcements_bar" %>

  <div class="min-h-full">
    <%= render "shared/navbar" %>

    <main class="py-10">
      <div class="mx-auto max-w-7xl px-4 sm:px-6 lg:px-8">
        <%= render "shared/flash" %>
        <%= yield %>
      </div>
    </main>
  </div>
</body>
</html>
```

### Marketing Layout (`app/views/layouts/marketing.html.erb`)

Public-facing pages (landing, pricing, about).

```erb
<!DOCTYPE html>
<html lang="en" class="h-full">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title><%= content_for(:title) || BaseX.config.app_name %></title>
  <%= csrf_meta_tags %>
  <%= csp_meta_tag %>
  <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
  <%= javascript_importmap_tags %>
</head>
<body class="h-full bg-white">
  <%= render "shared/marketing_nav" %>

  <main>
    <%= yield %>
  </main>

  <%= render "shared/footer" %>
</body>
</html>
```

### Admin Layout (`app/views/layouts/admin.html.erb`)

If not using Avo's built-in layout, create a simple admin shell:

```erb
<!DOCTYPE html>
<html lang="en" class="h-full bg-gray-100">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Admin — <%= BaseX.config.app_name %></title>
  <%= csrf_meta_tags %>
  <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
  <%= javascript_importmap_tags %>
</head>
<body class="h-full">
  <div class="flex h-full">
    <%= render "admin/sidebar" %>
    <main class="flex-1 overflow-y-auto p-8">
      <%= render "shared/flash" %>
      <%= yield %>
    </main>
  </div>
</body>
</html>
```

## Step 3: Shared Partials

### Navbar (`app/views/shared/_navbar.html.erb`)

```erb
<nav class="border-b border-gray-200 bg-white">
  <div class="mx-auto max-w-7xl px-4 sm:px-6 lg:px-8">
    <div class="flex h-16 justify-between">
      <div class="flex items-center">
        <%= link_to root_path, class: "text-xl font-bold text-gray-900" do %>
          <%= BaseX.config.app_name %>
        <% end %>
        <div class="ml-10 flex items-center space-x-4">
          <%= link_to "Dashboard", dashboard_path, class: "text-sm text-gray-600 hover:text-gray-900" %>
        </div>
      </div>

      <div class="flex items-center space-x-4">
        <%= render "shared/team_switcher" %>
        <%= render "shared/notification_bell" %>
        <%= render "shared/user_menu" %>
      </div>
    </div>
  </div>
</nav>
```

### Team Switcher (`app/views/shared/_team_switcher.html.erb`)

```erb
<div data-controller="dropdown" class="relative">
  <button data-action="click->dropdown#toggle" class="flex items-center text-sm text-gray-700 hover:text-gray-900">
    <span class="mr-1"><%= current_team&.name || "Select Team" %></span>
    <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"/>
    </svg>
  </button>

  <div data-dropdown-target="menu" class="hidden absolute right-0 mt-2 w-56 rounded-md bg-white shadow-lg ring-1 ring-black ring-opacity-5 z-50">
    <div class="py-1">
      <% current_user.teams.each do |team| %>
        <%= link_to team.name,
            switch_team_path(team),
            method: :patch,
            class: "block px-4 py-2 text-sm #{'font-bold bg-gray-50' if team == current_team} text-gray-700 hover:bg-gray-100" %>
      <% end %>
      <hr class="my-1">
      <%= link_to "Create New Team", new_team_path, class: "block px-4 py-2 text-sm text-primary-600 hover:bg-gray-100" %>
    </div>
  </div>
</div>
```

### Flash Messages (`app/views/shared/_flash.html.erb`)

```erb
<% flash.each do |type, message| %>
  <%
    colors = case type
             when "notice", "success" then "bg-green-50 text-green-800 border-green-200"
             when "alert", "error"    then "bg-red-50 text-red-800 border-red-200"
             when "warning"           then "bg-yellow-50 text-yellow-800 border-yellow-200"
             else "bg-blue-50 text-blue-800 border-blue-200"
             end
  %>
  <div class="mb-4 rounded-md border p-4 <%= colors %>"
       data-controller="dismissible"
       data-action="click->dismissible#dismiss">
    <div class="flex justify-between items-center">
      <p class="text-sm"><%= message %></p>
      <button class="text-current opacity-50 hover:opacity-100">&times;</button>
    </div>
  </div>
<% end %>
```

## Step 4: Stimulus Controllers

Create these standard Stimulus controllers.

### Dropdown (`app/javascript/controllers/dropdown_controller.js`)

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["menu"]

  toggle() {
    this.menuTarget.classList.toggle("hidden")
  }

  close(event) {
    if (!this.element.contains(event.target)) {
      this.menuTarget.classList.add("hidden")
    }
  }

  connect() {
    this.boundClose = this.close.bind(this)
    document.addEventListener("click", this.boundClose)
  }

  disconnect() {
    document.removeEventListener("click", this.boundClose)
  }
}
```

### Dismissible (`app/javascript/controllers/dismissible_controller.js`)

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  dismiss() {
    this.element.remove()
  }
}
```

### Clipboard (`app/javascript/controllers/clipboard_controller.js`)

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["source"]

  copy() {
    navigator.clipboard.writeText(this.sourceTarget.value || this.sourceTarget.textContent)
    const originalText = this.element.querySelector("[data-clipboard-label]")
    if (originalText) {
      const original = originalText.textContent
      originalText.textContent = "Copied!"
      setTimeout(() => { originalText.textContent = original }, 2000)
    }
  }
}
```

### Modal (`app/javascript/controllers/modal_controller.js`)

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["dialog"]

  open() {
    this.dialogTarget.showModal()
  }

  close() {
    this.dialogTarget.close()
  }

  clickOutside(event) {
    if (event.target === this.dialogTarget) {
      this.close()
    }
  }
}
```

### Auto-Submit (`app/javascript/controllers/auto_submit_controller.js`)

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { delay: { type: Number, default: 300 } }

  submit() {
    clearTimeout(this.timeout)
    this.timeout = setTimeout(() => {
      this.element.requestSubmit()
    }, this.delayValue)
  }
}
```

## Step 5: Core Routes

Here's the full routes file bringing everything together:

```ruby
Rails.application.routes.draw do
  # Devise
  devise_for :users, controllers: {
    omniauth_callbacks: "users/omniauth_callbacks",
    registrations: "users/registrations"
  }

  # Public / Marketing
  root "marketing#index"
  get "pricing", to: "marketing#pricing"

  # Authenticated
  get "dashboard", to: "dashboard#show"

  # Teams
  resources :teams do
    member do
      patch :switch
    end
    resources :memberships, only: [:index, :update, :destroy]
    resources :invitations, only: [:new, :create, :destroy]
  end

  # Accept invitation (public token-based route)
  get "invitations/:token", to: "invitations#accept", as: :accept_invitation

  # Settings
  resource :settings, only: [:show, :update]
  resource :two_factor, only: [:show, :create, :destroy] do
    post :verify
  end

  # Billing
  resource :billing, only: [:show, :create] do
    post :portal
  end

  # API Tokens
  resources :api_tokens, only: [:index, :create, :destroy]

  # Notifications
  resources :notifications, only: [:index] do
    member { post :mark_as_read }
    collection { post :mark_all_as_read }
  end

  # Announcements
  resources :announcements, only: [:index]

  # API
  namespace :api do
    namespace :v1 do
      resource :team, only: [:show]
    end
  end

  # Pay webhooks
  mount Pay::Engine, at: "/pay"

  # Health check
  get "up", to: "rails/health#show", as: :rails_health_check
end
```

## Step 6: Team Switching Controller

```ruby
class TeamsController < ApplicationController
  before_action :authenticate_user!

  def switch
    team = current_user.teams.find(params[:id])
    session[:team_id] = team.id
    redirect_to dashboard_path, notice: "Switched to #{team.name}"
  end

  def new
    @team = Team.new
  end

  def create
    @team = Team.new(team_params)
    if @team.save
      @team.memberships.create!(user: current_user, roles: { "admin" => true })
      session[:team_id] = @team.id
      redirect_to dashboard_path, notice: "Team created!"
    else
      render :new, status: :unprocessable_entity
    end
  end

  def show
    @team = current_team
    authorize @team
  end

  def update
    @team = current_team
    authorize @team
    if @team.update(team_params)
      redirect_to team_path(@team), notice: "Team updated."
    else
      render :show, status: :unprocessable_entity
    end
  end

  def destroy
    @team = current_team
    authorize @team
    @team.destroy
    session.delete(:team_id)
    redirect_to dashboard_path, notice: "Team deleted."
  end

  private

  def team_params
    params.require(:team).permit(:name, :domain, :subdomain)
  end
end
```

## Step 7: Dashboard

Create a simple dashboard controller and view:

```ruby
# app/controllers/dashboard_controller.rb
class DashboardController < ApplicationController
  before_action :authenticate_user!

  def show
    @team = current_team
    @members = @team.memberships.includes(:user).limit(5)
    @announcements = current_user.unread_announcements.limit(3)
  end
end
```

```erb
<%# app/views/dashboard/show.html.erb %>
<% content_for(:title, "Dashboard") %>

<div class="space-y-8">
  <div>
    <h1 class="text-2xl font-bold text-gray-900">Welcome back, <%= current_user.first_name %>!</h1>
    <p class="text-gray-600 mt-1">You're working in <strong><%= current_team.name %></strong></p>
  </div>

  <%# Quick stats cards %>
  <div class="grid grid-cols-1 sm:grid-cols-3 gap-6">
    <div class="bg-white rounded-lg shadow p-6">
      <p class="text-sm text-gray-500">Team Members</p>
      <p class="text-3xl font-bold text-gray-900"><%= current_team.memberships.count %></p>
    </div>
    <div class="bg-white rounded-lg shadow p-6">
      <p class="text-sm text-gray-500">Plan</p>
      <p class="text-3xl font-bold text-gray-900"><%= current_team.plan&.name || "Free" %></p>
    </div>
    <div class="bg-white rounded-lg shadow p-6">
      <p class="text-sm text-gray-500">Notifications</p>
      <p class="text-3xl font-bold text-gray-900"><%= current_user.notifications.unread.count %></p>
    </div>
  </div>

  <%# This is where the user will add their app-specific content %>
  <div class="bg-white rounded-lg shadow p-6">
    <h2 class="text-lg font-semibold text-gray-900 mb-4">Getting Started</h2>
    <p class="text-gray-600">
      Your Base-X app is ready. Start building by adding your own controllers, models, and views.
      This dashboard is your home base.
    </p>
  </div>
</div>
```

## Procfile

Create `Procfile.dev` for `bin/dev`:

```
web: bin/rails server -p 3000
css: bin/rails tailwindcss:watch
```

And `bin/dev`:

```bash
#!/usr/bin/env bash
if ! gem list foreman -i --silent; then
  echo "Installing foreman..."
  gem install foreman
fi
exec foreman start -f Procfile.dev "$@"
```

## Step 8: Generate README

As the final step of the build, generate a comprehensive `README.md` in the project root. The README should be tailored to the actual app that was generated — use the real app name, the billing provider the user chose, the auth providers configured, and the tenancy mode selected.

Include ALL of the following sections:

### README Structure

```markdown
# APP_NAME

One-line description: what the app is (a SaaS application built with Rails and Base-X).

## Getting Started

### Prerequisites

- Ruby VERSION (the version used during generation)
- Rails VERSION
- PostgreSQL
- Node.js (for asset pipeline)

### Setup

Step-by-step instructions:

1. Clone the repo
2. `bundle install`
3. Create `.env` file (list every env var the app expects with descriptions of what each one is for)
4. `rails db:create db:migrate db:seed`
5. `bin/dev` to start the development server
6. Visit http://localhost:3000

### Default Login

If seeds were run:
- Email: admin@example.com
- Password: password123

## Architecture

### Multi-Tenancy

Explain which tenancy mode is configured (session/subdomain/path) and how it works:
- How tenant resolution happens
- The role of `current_team` and `ActsAsTenant`
- How data isolation is enforced

### Core Models

Describe the data model relationships:
- **User** — authentication, belongs to teams through memberships
- **Team** — the primary tenant, all data scoped to teams
- **Membership** — join model with JSONB roles (`{ "admin": true }`)
- **Invitation** — token-based team invitations
- **Plan** — subscription tiers and feature flags

### Authentication

- Devise modules enabled (confirmable, lockable, trackable, etc.)
- Two-factor authentication (TOTP via `rotp`)
- Social login providers configured (list whichever were selected)
- Admin impersonation via `pretender`

### Authorization

- Pundit policies based on Membership roles
- `current_membership` helper
- Admin vs. member permissions

### Billing

- Which provider is configured (Stripe/Paddle/LemonSqueezy)
- Pay gem setup
- Team is the billable model
- Webhook endpoint: `/pay`
- Subscription lifecycle: trial → active → past_due → cancelled
- Billing portal and checkout flow

### Notifications

- Noticed v2 with database + email delivery
- How to create new notification types (extend `BaseNotification`)
- Notification bell in the navbar

### Admin Panel

- Avo (or Administrate) at `/admin`
- Admin-only access enforced
- Resources available: Users, Teams, Plans

### API

- Bearer token authentication
- Token management UI
- API namespace: `Api::V1`
- Example endpoint: `GET /api/v1/team`

## Environment Variables

Full table of every env var the app uses:

| Variable | Required | Description |
|----------|----------|-------------|
| `DATABASE_URL` | No | PostgreSQL connection string (defaults used if not set) |
| `RAILS_MAX_THREADS` | No | Database pool size (default: 5) |
| `STRIPE_PUBLIC_KEY` | Yes* | Stripe publishable key (*if Stripe billing) |
| `STRIPE_PRIVATE_KEY` | Yes* | Stripe secret key |
| `STRIPE_SIGNING_SECRET` | Yes* | Stripe webhook signing secret |
| `GOOGLE_CLIENT_ID` | Yes* | Google OAuth app ID (*if Google auth) |
| `GOOGLE_CLIENT_SECRET` | Yes* | Google OAuth app secret |
| `GITHUB_CLIENT_ID` | Yes* | GitHub OAuth app ID (*if GitHub auth) |
| `GITHUB_CLIENT_SECRET` | Yes* | GitHub OAuth app secret |

Only include rows relevant to the user's choices.

## Key Files & Directories

| Path | Purpose |
|------|---------|
| `app/controllers/concerns/set_current_team.rb` | Tenant resolution logic |
| `app/controllers/concerns/authorization.rb` | Membership-based auth helpers |
| `config/initializers/base_x.rb` | App configuration (tenant mode, app name) |
| `config/initializers/pay.rb` | Billing configuration |
| `lib/middleware/tenant_middleware.rb` | Subdomain/path tenant resolution |
| `app/notifications/` | Noticed notification classes |
| `app/policies/` | Pundit authorization policies |
| `app/components/` | ViewComponents |
| `app/javascript/controllers/` | Stimulus controllers |

## Common Tasks

### Create a new team-scoped model

```bash
rails generate model Widget name:string team:references
```

Add `acts_as_tenant(:team)` to the model.

### Add a new notification type

Create a class in `app/notifications/` extending `BaseNotification`.

### Add a new Pundit policy

Create a class in `app/policies/` extending `ApplicationPolicy`.

### Manage subscriptions

Use `current_team.subscribed?`, `current_team.on_trial?`, `current_team.plan`.

## Development

- `bin/dev` — start Rails + Tailwind watcher
- `rails db:seed` — seed plans and dev user
- `rails routes` — view all routes
- `/admin` — admin panel (requires admin user)
```

Adapt the above template to match what was actually generated. Remove sections for features the user didn't select (e.g., remove the Billing section if no billing provider was chosen). Use the real app name throughout.
