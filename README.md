# Base-X Rails

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that scaffolds production-ready Ruby on Rails SaaS applications with team-centric multi-tenancy, authentication, billing, notifications, and a Tailwind CSS UI.

Instead of spending days wiring up boilerplate, tell Claude Code "create a new Rails project" and Base-X generates a complete, ready-to-ship foundation in minutes.

## What Is a Claude Code Skill?

Skills are reusable prompt files that extend Claude Code's capabilities. When installed, Claude Code automatically detects when a skill is relevant and applies its instructions. Base-X teaches Claude Code how to scaffold an entire SaaS application following battle-tested Rails patterns.

## What You Get

Base-X generates a complete Rails 8.1 application with:

| Feature | Implementation |
|---------|---------------|
| **Multi-tenancy** | Team-centric with session, subdomain, or path-based resolution via [ActsAsTenant](https://github.com/ErwinM/acts_as_tenant) |
| **Authentication** | [Devise](https://github.com/heartcombo/devise) with confirmable, lockable, 2FA (TOTP), and social login (Google, GitHub) |
| **Authorization** | [Pundit](https://github.com/varvet/pundit) policies based on Membership roles (JSONB) |
| **Billing** | [Pay](https://github.com/pay-rails/pay) gem with Stripe, Paddle, or LemonSqueezy |
| **Notifications** | [Noticed](https://github.com/excid3/noticed) v2 with database + email delivery |
| **Admin panel** | [Avo](https://avohq.io/) with user, team, and plan management |
| **Background jobs** | [Solid Queue](https://github.com/rails/solid_queue) |
| **Caching** | [Solid Cache](https://github.com/rails/solid_cache) |
| **Frontend** | Tailwind CSS 4.0, Hotwire (Turbo + Stimulus), ViewComponents |
| **API** | Bearer token auth, versioned namespace (`Api::V1`) |
| **Impersonation** | Admin user-switching via [Pretender](https://github.com/ankane/pretender) |
| **Soft deletes** | [Discard](https://github.com/jmondo/discard) on critical models (Users, Teams, Memberships, Invitations, ApiTokens) |
| **Dev email** | [letter_opener_web](https://github.com/fgrehm/letter_opener_web) captures emails in browser at `/letter_opener` |
| **Job dashboard** | [Mission Control Jobs](https://github.com/rails/mission_control-jobs) web UI at `/admin/jobs` |
| **Setup wizard** | First-run `/setup` flow creates the initial admin account without seeds or console |

## Installation

### 1. Install Claude Code

If you haven't already, install [Claude Code](https://docs.anthropic.com/en/docs/claude-code):

```bash
npm install -g @anthropic-ai/claude-code
```

### 2. Install the Skill

Clone this repository into your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/jcroucher/base-x-rails.git ~/.claude/skills/base-x-rails
```

That's it. Claude Code will automatically discover and use the skill.

## Usage

Start Claude Code and ask it to create a new Rails project:

```
claude

> Create a new Rails SaaS app called "Acme"
```

Or be more specific:

```
> Scaffold a Rails app with subdomain-based multi-tenancy, Stripe billing, and Google OAuth
```

### Trigger Phrases

The skill activates when you mention any of:

- "Create a new Rails project"
- "Scaffold a SaaS app"
- "Build a multi-tenant Rails application"
- "Base-X"
- Any request for a Rails app with teams, subscriptions, or billing

### Configuration Prompts

Before generating, Claude Code will ask you to confirm:

| Setting | Options | Default |
|---------|---------|---------|
| **App name** | Any name | — |
| **Tenancy mode** | Session, subdomain, or path | Session |
| **Billing provider** | Stripe, Paddle, or LemonSqueezy | Stripe |
| **Social auth** | Google, GitHub, Twitter, or none | Google + GitHub |
| **Ruby/Rails version** | Any stable version | Latest |

## Architecture

### Multi-Tenancy Model

Every piece of data in the app belongs to a **Team** (the primary tenant). Users access teams through **Memberships**, which store roles as JSONB:

```
User ──has_many──▶ Membership ◀──has_many── Team
                   (roles: JSONB)
```

- Every new user gets a "Personal" team automatically
- `current_team` is available in all controllers and views
- `ActsAsTenant` enforces row-level data isolation
- Tenant resolution supports session, subdomain, or path modes

### Core Models

| Model | Purpose |
|-------|---------|
| `User` | Authentication, profile, global attributes |
| `Team` | Primary tenant — all scoped data belongs here |
| `Membership` | User-Team join with JSONB roles (`{ "admin": true }`) |
| `Invitation` | Secure token-based team invitations |
| `Plan` | Subscription tiers with feature flags |
| `ApiToken` | Bearer tokens for API access |
| `Announcement` | System-wide broadcasts from admins |

### Project Structure

```
app/
├── controllers/
│   ├── concerns/
│   │   ├── set_current_team.rb      # Tenant resolution
│   │   └── authorization.rb         # Membership helpers
│   ├── application_controller.rb
│   ├── dashboard_controller.rb
│   ├── teams_controller.rb
│   ├── memberships_controller.rb
│   ├── invitations_controller.rb
│   ├── billings_controller.rb
│   ├── notifications_controller.rb
│   ├── api_tokens_controller.rb
│   └── api/
│       └── base_controller.rb       # API token auth
├── models/
│   ├── user.rb
│   ├── team.rb
│   ├── membership.rb
│   ├── invitation.rb
│   ├── plan.rb
│   ├── api_token.rb
│   └── announcement.rb
├── policies/                         # Pundit policies
├── notifications/                    # Noticed notifications
├── components/                       # ViewComponents
├── javascript/
│   └── controllers/                  # Stimulus controllers
│       ├── dropdown_controller.js
│       ├── dismissible_controller.js
│       ├── clipboard_controller.js
│       ├── modal_controller.js
│       └── auto_submit_controller.js
└── views/
    └── layouts/
        ├── application.html.erb      # Authenticated shell
        ├── marketing.html.erb        # Public pages
        └── admin.html.erb            # Admin panel
config/
├── initializers/
│   ├── base_x.rb                     # App configuration
│   └── pay.rb                        # Billing config
└── routes.rb
lib/
└── middleware/
    └── tenant_middleware.rb           # Subdomain/path resolution
```

## Build Phases

The skill generates the app in five phases, each building on the last:

### Phase 1 — Project Setup & Core Models

- Creates the Rails 8.1 app with PostgreSQL, Solid Queue, Solid Cache
- Generates User, Team, Membership, Invitation, and Plan models
- Sets up personal-team auto-creation
- Implements multi-tenant resolution middleware
- Configures ActsAsTenant for data isolation

### Phase 2 — Authentication & Billing

- Installs Devise with confirmable, lockable, recoverable, trackable
- Adds two-factor auth (TOTP via `rotp` + `rqrcode`)
- Configures OmniAuth social login providers
- Sets up the Pay gem with your chosen billing provider
- Builds subscription lifecycle (trial → active → past_due → cancelled)
- Implements Pundit policies and admin impersonation

### Phase 3 — Notifications, Admin & API

- Installs Noticed v2 with database + email delivery
- Builds an admin panel with Avo (user/team/subscription management)
- Creates API token model and Bearer auth
- Adds announcement broadcasting system

### Phase 4 — UI, Frontend & Polish

- Configures Tailwind CSS 4.0 with custom theme
- Builds three layout shells (marketing, dashboard, admin)
- Creates shared partials (navbar, team switcher, flash messages, notification bell)
- Wires up Stimulus controllers (dropdown, modal, clipboard, dismissible, auto-submit)
- Generates the project README

### Phase 5 — Developer Experience & App Setup

- Adds `letter_opener_web` to capture development emails in the browser
- Fixes seeds to set `confirmed_at` so the admin user can log in immediately
- Builds a first-run setup wizard at `/setup` that creates the initial admin account
- Adds soft deletes via `discard` on Users, Teams, Memberships, Invitations, and ApiTokens
- Installs Mission Control Jobs dashboard at `/admin/jobs` for Solid Queue monitoring
- Configures development email delivery via letter_opener_web

## Post-Generation Setup

After the skill finishes, you'll need to:

1. **Install dependencies** — `bundle install`
2. **Create your `.env` file** with the required API keys (Claude Code will tell you exactly which variables are needed based on your choices)
3. **Set up the database** — `rails db:create db:migrate db:seed`
4. **Start the server** — `bin/dev`
5. **Visit** `http://localhost:3000`

### First-Run Setup

On a fresh install (no seeds), visit `http://localhost:3000` — you'll be redirected to `/setup` to create your admin account. This bypasses Devise's email confirmation requirement.

Alternatively, run `rails db:seed` to create the default admin user.

### Default Development Login

After running seeds:
- **Email:** `admin@example.com`
- **Password:** `password123`

### Development Routes

| Route | Purpose |
|-------|---------|
| `/setup` | First-run admin setup wizard (disabled after first admin exists) |
| `/letter_opener` | Email preview UI (development only) |
| `/admin/jobs` | Solid Queue job dashboard (admin only) |

## Environment Variables

The generated app may use these variables depending on your configuration:

| Variable | Required | Description |
|----------|----------|-------------|
| `DATABASE_URL` | No | PostgreSQL connection string (defaults are used if not set) |
| `RAILS_MAX_THREADS` | No | Database pool size (default: 5) |
| `STRIPE_PUBLIC_KEY` | If Stripe | Stripe publishable key |
| `STRIPE_PRIVATE_KEY` | If Stripe | Stripe secret key |
| `STRIPE_SIGNING_SECRET` | If Stripe | Stripe webhook signing secret |
| `GOOGLE_CLIENT_ID` | If Google auth | Google OAuth application ID |
| `GOOGLE_CLIENT_SECRET` | If Google auth | Google OAuth application secret |
| `GITHUB_CLIENT_ID` | If GitHub auth | GitHub OAuth application ID |
| `GITHUB_CLIENT_SECRET` | If GitHub auth | GitHub OAuth application secret |

## Common Tasks After Generation

### Add a new team-scoped model

```bash
rails generate model Widget name:string team:references
```

Then add `acts_as_tenant(:team)` to the model.

### Add a new notification type

Create a class in `app/notifications/` extending `BaseNotification`:

```ruby
class WidgetCreatedNotification < BaseNotification
  notification_methods do
    def message
      "A new widget was created: #{params[:widget].name}"
    end
  end
end
```

### Add a new authorization policy

Create a class in `app/policies/` extending `ApplicationPolicy`:

```ruby
class WidgetPolicy < ApplicationPolicy
  def show?
    member?
  end

  def update?
    admin?
  end
end
```

### Check subscription status

```ruby
current_team.subscribed?   # Has active subscription?
current_team.on_trial?     # Currently in trial?
current_team.plan          # Current Plan record
```

### Soft delete and restore records

Critical models (User, Team, Membership, Invitation, ApiToken) use soft deletes via Discard:

```ruby
user.soft_delete!           # Sets discarded_at, hides from default queries
user.restore!               # Clears discarded_at, makes visible again

User.all                    # Only kept (non-deleted) records
User.discarded              # Only soft-deleted records
User.with_discarded         # All records including soft-deleted
```

## Customization

### Changing the Tenancy Mode

Edit `config/initializers/base_x.rb`:

```ruby
BaseX.configure do |config|
  config.tenant_mode = :subdomain  # :session, :subdomain, or :path
end
```

### Changing the Theme

Edit the `@theme` block in `app/assets/stylesheets/application.css` to adjust the primary color palette.

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- Ruby 3.2+
- Rails 8.0+
- PostgreSQL
- Node.js (for asset pipeline)

## Contributing

Contributions are welcome! If you'd like to improve the skill:

1. Fork the repository
2. Create a feature branch
3. Make your changes to the skill files
4. Submit a pull request

### Skill File Structure

| File | Purpose |
|------|---------|
| `SKILL.md` | Main skill definition — triggers, overview, execution flow |
| `phase1-core.md` | Phase 1 reference: project setup, core models, multi-tenancy |
| `phase2-auth-billing.md` | Phase 2 reference: authentication, billing, authorization |
| `phase3-features.md` | Phase 3 reference: notifications, admin panel, API |
| `phase4-ui.md` | Phase 4 reference: UI, Stimulus controllers, layouts |
| `phase5-dev-setup.md` | Phase 5 reference: dev email, setup wizard, soft deletes, job dashboard |

## License

MIT
