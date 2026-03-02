---
name: base-x-rails
description: Scaffold a production-ready Rails SaaS application with team-centric multi-tenancy, authentication, billing, notifications, and a Tailwind CSS UI. Use this skill whenever the user wants to create a new Rails project, scaffold a SaaS app, build a multi-tenant Rails application, set up a Rails starter/boilerplate, or mentions "Base-X". Also trigger when the user asks for a Rails app with teams, subscriptions, billing, or multi-tenancy. This is the go-to skill for any "create a new Rails project" or "scaffold a SaaS" request.
---

# Base-X: SaaS-Ready Rails Scaffold

Base-X generates a complete, production-ready Rails SaaS application from scratch. It follows a team-centric architecture where every piece of data belongs to a Team (the primary tenant), and users access teams through Memberships.

## When to Use This Skill

- User says "create a new Rails project", "scaffold a SaaS app", "new Rails app", or similar
- User mentions wanting teams, multi-tenancy, subscriptions, or billing in a Rails app
- User references "Base-X" by name

## Overview

The generated app includes:

- **Team-centric multi-tenancy** (session, subdomain, or path-based)
- **Authentication** via Devise (2FA, social auth, impersonation)
- **Authorization** via Pundit (role-based on Memberships)
- **Billing** via the Pay gem (Stripe, Paddle, LemonSqueezy)
- **Notifications** via Noticed (in-app, email, SMS, webhooks)
- **Admin panel** via Avo or Administrate
- **UI** with Tailwind CSS 4.0, Hotwire (Turbo + Stimulus)
- **Background jobs** via Solid Queue
- **Caching** via Solid Cache

## Execution Flow

Work through these phases in order. Read each reference file before starting that phase.

### Phase 1: Project Setup & Core Models
**Read:** `references/phase1-core.md`

1. Create the Rails 8.1 app with PostgreSQL, Solid Queue, Solid Cache
2. Generate User, Team, Membership, Invitation, and Plan models
3. Set up the "personal team" auto-creation logic
4. Implement multi-tenant resolution middleware
5. Configure ActsAsTenant for row-level data isolation

### Phase 2: Authentication & Billing
**Read:** `references/phase2-auth-billing.md`

1. Install and configure Devise with confirmable, lockable, recoverable
2. Add two-factor auth with `rotp` and `rqrcode`
3. Set up OmniAuth providers (Google, GitHub)
4. Install the Pay gem and configure Stripe
5. Build subscription lifecycle (trial → active → past_due → cancelled)
6. Implement Pundit policies based on Membership roles
7. Add impersonation with the `pretender` gem

### Phase 3: Notifications, Admin & API
**Read:** `references/phase3-features.md`

1. Install Noticed v2 and set up delivery channels (database, email)
2. Build the admin panel with Avo (user/team/subscription management)
3. Create API token model and Bearer auth for API access
4. Add announcement broadcasting system

### Phase 4: UI, Frontend & README
**Read:** `references/phase4-ui.md`

1. Configure Tailwind CSS 4.0
2. Build layout shells (marketing site, authenticated dashboard, admin)
3. Create reusable ViewComponents (modals, flash messages, nav, forms)
4. Wire up Stimulus controllers (clipboard, auto-submit, dropdown, modal)
5. Set up Turbo Streams for real-time updates
6. Generate a comprehensive `README.md` covering setup, architecture, and usage

## Key Decisions to Confirm with the User

Before generating, ask the user:

1. **App name** — What should the project be called?
2. **Multi-tenancy mode** — Session-based (default), subdomain, or path-based?
3. **Billing provider** — Stripe (default), Paddle, or LemonSqueezy?
4. **Social auth providers** — Which ones? (Google, GitHub, Twitter, or none)
5. **Ruby/Rails version** — Use latest stable defaults unless specified

If the user just says "create a new Rails project" without details, use these defaults:
- Session-based tenancy
- Stripe billing
- Google + GitHub social auth
- Latest stable Ruby + Rails

## Pre-Build: Environment File Setup

**IMPORTANT:** Claude Code cannot write to `.env` files. Before starting the build, you MUST ask the user to create a `.env` file in their project directory with the required environment variables.

After confirming the user's choices above, present them with the exact `.env` contents they need based on their selections. Tell them to create the file manually before you proceed.

### Required `.env` contents (adjust based on user choices):

```
# Database (optional — defaults will be used if not set)
# RAILS_MAX_THREADS=5
# DATABASE_URL=

# Stripe (if Stripe billing selected)
STRIPE_PUBLIC_KEY=pk_test_xxx
STRIPE_PRIVATE_KEY=sk_test_xxx
STRIPE_SIGNING_SECRET=whsec_xxx

# Google OAuth (if Google auth selected)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# GitHub OAuth (if GitHub auth selected)
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
```

Only include the sections relevant to the user's choices. Tell the user they can use placeholder values for now and fill in real keys later, but the file must exist.

Wait for the user to confirm they have created the `.env` file before proceeding with Phase 1.

## Project Structure

```
app-name/
├── app/
│   ├── controllers/
│   │   ├── application_controller.rb      # Tenant-aware base
│   │   ├── dashboard_controller.rb
│   │   ├── teams_controller.rb
│   │   ├── memberships_controller.rb
│   │   ├── invitations_controller.rb
│   │   ├── billings_controller.rb
│   │   └── api/
│   │       └── base_controller.rb         # API token auth
│   ├── models/
│   │   ├── user.rb
│   │   ├── team.rb
│   │   ├── membership.rb
│   │   ├── invitation.rb
│   │   ├── plan.rb
│   │   ├── api_token.rb
│   │   └── announcement.rb
│   ├── policies/                           # Pundit policies
│   ├── notifications/                      # Noticed notifications
│   ├── components/                         # ViewComponents
│   ├── javascript/
│   │   └── controllers/                    # Stimulus controllers
│   └── views/
│       ├── layouts/
│       │   ├── application.html.erb        # Authenticated layout
│       │   ├── marketing.html.erb          # Public pages
│       │   └── admin.html.erb              # Admin layout
│       └── ...
├── config/
│   ├── initializers/
│   │   ├── base_x.rb                       # App configuration
│   │   └── pay.rb                          # Billing config
│   └── routes.rb
├── db/
│   └── migrate/
└── lib/
    └── middleware/
        └── tenant_middleware.rb
```

## Naming Conventions

| Concept | Name |
|---|---|
| Primary tenant | `Team` |
| User-Team join | `Membership` |
| Config namespace | `BaseX` |
| Admin route | `/admin` |
| API namespace | `Api::V1` |

## Important Implementation Notes

- Every new user automatically gets a "Personal" team created via an `after_create` callback
- The `current_team` helper is available in all controllers and views
- All tenant-scoped models include `acts_as_tenant(:team)`
- Membership roles are stored as JSONB: `{ "admin": true }` pattern
- The Pay gem handles webhook processing — mount its engine in routes
- Use `team_signed_in?` helper alongside Devise's `user_signed_in?`

## Post-Generation Steps

After the skill finishes generating the app, remind the user to:

1. Run `bundle install` and `rails db:create db:migrate`
2. Update their `.env` file with real API keys (the file was created in the Pre-Build step)
3. Configure their billing provider API keys
4. Configure OAuth app credentials for social auth
5. Run `bin/dev` to start the development server
6. Review the generated `README.md` for a full overview of the app's architecture and setup
