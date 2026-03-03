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
- **User settings** with profile, password, connected accounts, terms gate, account deletion
- **Rate limiting** via Rack::Attack (login, API, password reset throttling)
- **CORS** via Rack::Cors for API access
- **Pagination** via Pagy with Tailwind styling
- **Test suite** with Minitest (model, controller, system tests) and CI via GitHub Actions
- **Verification** script (`bin/verify`) to validate the entire setup

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

### Phase 5: Developer Experience & App Setup
**Read:** `references/phase5-dev-setup.md`

1. Add `letter_opener_web`, `mission_control-jobs`, and `discard` gems
2. Configure development email delivery via letter_opener_web
3. Fix seeds to set `confirmed_at` on the admin user
4. Build first-run setup wizard (`/setup`) with `SetupGuard` concern
5. Add soft deletes (Discard) to Users, Teams, Memberships, Invitations, ApiTokens
6. Configure Mission Control Jobs dashboard at `/admin/jobs`
7. Add routes for setup wizard, letter_opener (dev), and jobs dashboard

### Phase 6: User Settings & Account Management
**Read:** `references/phase6-settings.md`

1. Build tabbed settings page (Profile, Password, Connected Accounts, Danger Zone)
2. Create Tabs Stimulus controller for client-side tab switching
3. Add timezone selector using `ActiveSupport::TimeZone`
4. Build authenticated password change with `bypass_sign_in`
5. Create ConnectedAccount model for multi-provider OAuth
6. Update OmniAuth callbacks to use ConnectedAccount
7. Add TermsGate concern redirecting to `/terms` if terms not accepted
8. Build terms acceptance flow
9. Add account deletion with password confirmation (soft delete)

### Phase 7: Team Members & Invitations
**Read:** `references/phase7-teams.md`

1. Build MembershipsController with member list, role changes, and removal
2. Build InvitationsController with send, cancel, and accept flows
3. Handle invitation acceptance after sign-up via session token
4. Create InvitationMailer with HTML and text templates
5. Add Pundit policies for invitations and memberships
6. Build TeamSettingsController for team name and billing info
7. Create views for member management, invitation form, and team settings

### Phase 8: Rate Limiting, CORS & Pagination
**Read:** `references/phase8-infrastructure.md`

1. Add `rack-attack`, `rack-cors`, and `pagy` gems
2. Configure Rack::Attack throttles (login, password reset, invitations, API)
3. Configure Rack::Cors for API endpoints
4. Set up Pagy with 25 items/page defaults
5. Include Pagy in ApplicationController and API base controller
6. Apply pagination to memberships, notifications, and API tokens
7. Create pagination partial with Tailwind styling
8. Add API pagination headers (X-Total-Count, X-Page, X-Per-Page)

### Phase 9: Verification & Boot Check
**Read:** `references/phase9-verification.md`

1. Install SolidQueue, SolidCache, Noticed, and Pay migration tables
2. Run full migration suite with `db:create db:migrate db:seed`
3. Generate `bin/verify` script to validate database, tables, routes, and boot
4. Run boot smoke test
5. Fix any inconsistencies discovered during verification

### Phase 10: Test Suite
**Read:** `references/phase10-tests.md`

1. Configure test helper with Devise integration and multi-tenant helpers
2. Create fixtures for all models (users, teams, memberships, invitations, plans, api_tokens, announcements, connected_accounts)
3. Write model tests for all core models
4. Write controller/integration tests for settings, passwords, memberships, invitations, dashboard, teams, terms, and API
5. Write system tests for registration, team management, and settings
6. Create GitHub Actions CI workflow (PostgreSQL, test suite, RuboCop)

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
│   │   ├── setup_controller.rb            # First-run setup wizard
│   │   ├── settings_controller.rb         # User profile settings
│   │   ├── passwords_controller.rb        # Password change
│   │   ├── connected_accounts_controller.rb # OAuth account management
│   │   ├── terms_controller.rb            # Terms acceptance
│   │   ├── account_deletions_controller.rb # Account deletion
│   │   ├── teams_controller.rb
│   │   ├── team_settings_controller.rb    # Team name/billing
│   │   ├── memberships_controller.rb
│   │   ├── invitations_controller.rb
│   │   ├── billings_controller.rb
│   │   ├── concerns/
│   │   │   ├── setup_guard.rb             # Redirects to /setup when no admin
│   │   │   ├── terms_gate.rb              # Redirects to /terms if not accepted
│   │   │   └── ...
│   │   └── api/
│   │       └── base_controller.rb         # API token auth + pagination
│   ├── models/
│   │   ├── concerns/
│   │   │   └── soft_deletable.rb          # Discard wrapper
│   │   ├── user.rb
│   │   ├── team.rb
│   │   ├── membership.rb
│   │   ├── invitation.rb
│   │   ├── plan.rb
│   │   ├── api_token.rb
│   │   ├── announcement.rb
│   │   └── connected_account.rb           # OAuth provider links
│   ├── mailers/
│   │   └── invitation_mailer.rb           # Team invitation emails
│   ├── policies/                           # Pundit policies
│   ├── notifications/                      # Noticed notifications
│   ├── components/                         # ViewComponents
│   ├── javascript/
│   │   └── controllers/                    # Stimulus controllers
│   │       └── tabs_controller.js          # Settings tab switcher
│   └── views/
│       ├── layouts/
│       │   ├── application.html.erb        # Authenticated layout
│       │   ├── marketing.html.erb          # Public pages
│       │   └── admin.html.erb              # Admin layout
│       ├── settings/                       # Tabbed settings views
│       ├── terms/                          # Terms acceptance
│       ├── memberships/                    # Team member management
│       ├── invitations/                    # Invitation form
│       ├── team_settings/                  # Team settings
│       └── shared/
│           └── _pagination.html.erb        # Pagy pagination partial
├── bin/
│   └── verify                             # Setup verification script
├── config/
│   ├── initializers/
│   │   ├── base_x.rb                       # App configuration
│   │   ├── mission_control.rb              # Jobs dashboard config
│   │   ├── pay.rb                          # Billing config
│   │   ├── rack_attack.rb                  # Rate limiting
│   │   ├── cors.rb                         # API CORS config
│   │   └── pagy.rb                         # Pagination config
│   └── routes.rb
├── db/
│   └── migrate/
├── test/
│   ├── test_helper.rb                     # Multi-tenant test helpers
│   ├── application_system_test_case.rb
│   ├── fixtures/                          # YAML fixtures for all models
│   ├── models/                            # Model unit tests
│   ├── controllers/                       # Integration tests
│   └── system/                            # Browser-driven system tests
├── .github/
│   └── workflows/
│       └── ci.yml                         # GitHub Actions CI
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

1. Run `bundle install` and `rails db:create db:migrate db:seed`
2. Update their `.env` file with real API keys (the file was created in the Pre-Build step)
3. Configure their billing provider API keys
4. Configure OAuth app credentials for social auth
5. Run `bin/verify` to validate the entire setup (database, tables, routes, boot)
6. Run `rails test` to confirm all tests pass
7. Run `bin/dev` to start the development server
8. Review the generated `README.md` for a full overview of the app's architecture and setup
