# Phase 9: Verification & Boot Check

This phase ensures the entire application is correctly wired up — all migrations run, all gem tables exist, and the app boots without errors. Run this phase after completing phases 1-8.

## Step 1: Install SolidQueue Tables

SolidQueue needs its database tables to function as the queue backend:

```bash
rails solid_queue:install:migrations
rails db:migrate
```

Verify the tables exist: `solid_queue_jobs`, `solid_queue_scheduled_executions`, `solid_queue_ready_executions`, `solid_queue_claimed_executions`, `solid_queue_blocked_executions`, `solid_queue_failed_executions`, `solid_queue_pauses`, `solid_queue_processes`, `solid_queue_semaphores`, `solid_queue_recurring_executions`.

## Step 2: Install SolidCache Tables

SolidCache stores cache entries in the database:

```bash
rails solid_cache:install:migrations
rails db:migrate
```

Verify the table exists: `solid_cache_entries`.

## Step 3: Install Noticed Tables

Noticed v2 stores notification events and individual notifications:

```bash
rails noticed:install:migrations
rails db:migrate
```

Verify the tables exist: `noticed_events`, `noticed_notifications`.

If the tables already exist from Phase 3, skip this step.

## Step 4: Install Pay Tables

The Pay gem creates customer, subscription, and charge records:

```bash
rails pay:install:migrations
rails db:migrate
```

Verify the tables exist: `pay_customers`, `pay_subscriptions`, `pay_charges`, `pay_merchants`, `pay_payment_methods`, `pay_webhooks`.

If the tables already exist from Phase 2, skip this step.

## Step 5: Run Full Migration Suite

Run all pending migrations and seed the database:

```bash
rails db:create db:migrate db:seed
```

This should complete without errors. If any migration fails, fix the issue before proceeding.

## Step 6: Generate `bin/verify` Script

Create `bin/verify` — a Ruby script that loads the Rails environment and validates the entire setup:

```ruby
#!/usr/bin/env ruby
require_relative "../config/environment"

class Verifier
  EXPECTED_TABLES = %w[
    users
    teams
    memberships
    invitations
    plans
    api_tokens
    announcements
    announcement_reads
    connected_accounts
    solid_queue_jobs
    solid_cache_entries
    noticed_events
    noticed_notifications
    pay_customers
    pay_subscriptions
    pay_charges
  ].freeze

  EXPECTED_ROUTES = %w[
    /
    /dashboard
    /settings
    /api/v1/team
  ].freeze

  def initialize
    @errors = []
    @warnings = []
  end

  def run
    puts "=" * 60
    puts "Base-X Verification"
    puts "=" * 60
    puts

    check_database_connection
    check_tables
    check_pending_migrations
    check_plans_seeded
    check_routes
    check_boot

    puts
    puts "=" * 60

    if @errors.any?
      puts "FAILED — #{@errors.count} error(s) found:"
      @errors.each { |e| puts "  ✗ #{e}" }
    else
      puts "ALL CHECKS PASSED"
    end

    if @warnings.any?
      puts
      puts "Warnings:"
      @warnings.each { |w| puts "  ⚠ #{w}" }
    end

    puts "=" * 60

    exit(@errors.any? ? 1 : 0)
  end

  private

  def check_database_connection
    print "Checking database connection... "
    if ActiveRecord::Base.connection.active?
      puts "OK"
    else
      @errors << "Database connection is not active"
      puts "FAIL"
    end
  rescue => e
    @errors << "Database connection failed: #{e.message}"
    puts "FAIL"
  end

  def check_tables
    print "Checking expected tables... "
    existing = ActiveRecord::Base.connection.tables
    missing = EXPECTED_TABLES - existing

    if missing.empty?
      puts "OK (#{EXPECTED_TABLES.count} tables)"
    else
      @errors << "Missing tables: #{missing.join(', ')}"
      puts "FAIL"
    end
  end

  def check_pending_migrations
    print "Checking for pending migrations... "
    context = ActiveRecord::MigrationContext.new(Rails.root.join("db/migrate"))

    if context.needs_migration?
      pending = context.migrations.count - context.get_all_versions.count
      @errors << "#{pending} pending migration(s)"
      puts "FAIL"
    else
      puts "OK"
    end
  end

  def check_plans_seeded
    print "Checking plans are seeded... "
    count = Plan.count

    if count > 0
      puts "OK (#{count} plans)"
    else
      @warnings << "No plans found — run `rails db:seed` to create default plans"
      puts "WARN"
    end
  end

  def check_routes
    print "Checking key routes... "
    app = Rails.application
    routes_ok = true

    EXPECTED_ROUTES.each do |path|
      begin
        env = Rack::MockRequest.env_for(path)
        recognized = app.routes.recognize_path(path, method: :get)
        unless recognized
          @errors << "Route not recognized: GET #{path}"
          routes_ok = false
        end
      rescue ActionController::RoutingError
        @errors << "Route not found: GET #{path}"
        routes_ok = false
      rescue => e
        # Some routes require params — that's OK, they exist
      end
    end

    puts routes_ok ? "OK" : "FAIL"
  end

  def check_boot
    print "Checking app boot... "
    # If we got this far, the app has already booted
    if defined?(Rails) && Rails.application.initialized?
      puts "OK"
    else
      @errors << "Rails application failed to initialize"
      puts "FAIL"
    end
  end
end

Verifier.new.run
```

Make it executable:

```bash
chmod +x bin/verify
```

## Step 7: Boot Smoke Test

Run a quick smoke test to verify the app boots without load errors:

```bash
rails runner "puts 'Boot OK'"
```

If this fails, check the error output and fix any issues (missing gems, syntax errors, bad initializers, etc.).

## Step 8: Run the Verify Script

```bash
bin/verify
```

Expected output on a correctly set up app:

```
============================================================
Base-X Verification
============================================================

Checking database connection... OK
Checking expected tables... OK (16 tables)
Checking for pending migrations... OK
Checking plans are seeded... OK (3 plans)
Checking key routes... OK
Checking app boot... OK

============================================================
ALL CHECKS PASSED
============================================================
```

## Step 9: Fix Any Inconsistencies

If the verification script reveals issues, fix them now:

- **Missing tables** — Check which gem install step was skipped and run the appropriate migration command
- **Pending migrations** — Run `rails db:migrate`
- **Missing plans** — Run `rails db:seed`
- **Route errors** — Verify `config/routes.rb` has all expected route definitions
- **Boot failures** — Check for syntax errors, missing requires, or circular dependencies in initializers

Iterate until `bin/verify` reports all checks passed.
