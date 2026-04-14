# CLAUDE.md — Global Rules for Ruby on Rails Development
# Applies to: Every project, every file, every task, every environment
# Place this file at: ~/.claude/CLAUDE.md
#
# These instructions are MANDATORY and NON-NEGOTIABLE.
# Apply ALL sections to every code change, suggestion, review, generation, or refactor.
# No rule may be skipped to save time, reduce length, or simplify output.
# If something cannot be done securely, say so explicitly before proceeding.

---

## WHO I AM

I am a Ruby on Rails developer. I build production web applications that handle real users,
real data, and real business logic. My projects span local development, staging, and production
environments. I work across the full stack: models, controllers, views, APIs, background jobs,
authentication systems, file uploads, third-party integrations, and infrastructure.

Every piece of code Claude writes, suggests, or reviews must be production-grade from day one.
"We'll fix it later" is not acceptable. Flag issues now, inline, with severity markers.

---

## 1. RUBY ON RAILS — CORE STANDARDS

### Version & Compatibility
- Always ask or infer the Rails version before writing code. Syntax and APIs differ significantly
  between Rails 6, 7, and 8. Never assume.
- Always use the idiomatic Rails way for the detected version — no monkey-patching standard behavior.
- Prefer built-in Rails abstractions over custom implementations unless there is a documented reason.

### Code Style
- Follow the Ruby Style Guide and Rails best practices at all times.
- Use `frozen_string_literal: true` at the top of every Ruby file.
- Prefer keyword arguments for methods with more than two parameters.
- Avoid abbreviations in variable, method, and class names unless they are Rails conventions.
- Methods longer than 10 lines need a refactoring note unless complexity is unavoidable.
- Classes longer than 200 lines need a refactoring note suggesting service objects or concerns.
- Use `private` and `protected` appropriately — never leave all methods public by default.
- Never use `rescue Exception` — always rescue specific error classes.

### Models
- Every model attribute that accepts user input MUST have an ActiveRecord validation.
- Validate format, length, presence, numericality, and inclusion wherever applicable.
- Always add database-level constraints (NOT NULL, UNIQUE, FK) in migrations to back up model validations.
- Never trust model validations alone — the database is the last line of defense.
- Use scopes for reusable query logic. Never duplicate `where` chains across the codebase.
- Callbacks (`before_save`, `after_create`, etc.) must be used sparingly and always documented with
  a comment explaining why the callback is needed and what side effects it causes.
- Never put business logic in callbacks that belongs in a service object.
- Use `enum` with explicit integer mappings: `enum status: { draft: 0, active: 1, archived: 2 }`.
- Sensitive attributes (passwords, tokens, keys, SSNs, payment data) must use encrypted attributes
  (`attr_encrypted` or Rails 7+ `encrypts`) — never store plaintext.

### Controllers
- Controllers must be thin. Business logic belongs in service objects or models, not controllers.
- Every action must have explicit authorization — never assume a logged-in user can do anything.
- Use `before_action :authenticate_user!` on all controllers; opt out explicitly only where public.
- Always use strong parameters. Never use `params` directly with `.create`, `.update`, or `.new`.
- Never use `.permit!` under any circumstances.
- Admin-only fields (role, admin, verified, credits, etc.) must NEVER appear in standard strong params.
- Rescue specific ActiveRecord errors at the controller level and return appropriate HTTP status codes.
- Never let raw `ActiveRecord::RecordNotFound` bubble up — return 404 with a friendly response.

### Views & ERB
- Rails auto-escapes ERB. Never use `html_safe`, `raw`, or `.html_safe` on user-provided content.
- If rich HTML output is needed, use `rails-html-sanitizer` or `loofah` with an explicit allowlist.
- Never render user input directly into JavaScript contexts — always JSON-encode and escape.
- Keep views logic-free — any conditional or loop that is more than 2 lines belongs in a helper
  or presenter.

### Routing
- Use resourceful routes (`resources`, `resource`) wherever possible.
- Namespace admin routes: `namespace :admin do` and protect the entire namespace with a `before_action`.
- Never expose sequential IDs in URLs for user-facing resources — use UUIDs or `hashid` to prevent
  enumeration attacks.
- Keep `routes.rb` organized — group by namespace, annotate non-obvious routes with comments.

---

## 2. SECURITY — APPLY TO EVERY LINE OF CODE

### Principle of Least Privilege
- Users can only access their own data unless explicitly granted broader permissions.
- Every controller action needs authorization checked — use Pundit or CanCanCan.
- Never authorize at the view level only. Always enforce at the model/policy level.
- Admin functionality must be behind a separate namespace with role-verified `before_action`.

### SQL Injection Prevention
- NEVER interpolate user input into queries:
```ruby
# NEVER:
User.where("email = '#{params[:email]}'")
User.order("#{params[:sort]} DESC")

# ALWAYS:
User.where(email: params[:email])
User.where("email = ?", params[:email])
ALLOWED_SORT = %w[name email created_at].freeze
order_col = ALLOWED_SORT.include?(params[:sort]) ? params[:sort] : "created_at"
```
- Allowlist all dynamic column names for `.order()`, `.select()`, `.group()`, `.pluck()`.

### XSS Prevention
- Never use `html_safe`, `raw`, or `.html_safe` on any string containing user data.
- Set a strict Content Security Policy in all environments (production and staging at minimum):
```ruby
# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self
  policy.script_src  :self
  policy.style_src   :self
  policy.img_src     :self, :data
  policy.object_src  :none
  policy.frame_ancestors :none
end
```

### CSRF Protection
- NEVER disable `protect_from_forgery`. It must be active in all environments.
- Use `protect_from_forgery with: :exception` for browser apps.
- API-only endpoints using token auth: use `protect_from_forgery with: :null_session` scoped
  to those endpoints only, and ensure Bearer token validation replaces it.
- Never use `skip_before_action :verify_authenticity_token` without a comment explaining why.

### Mass Assignment
- Strong parameters on every controller action that accepts user input.
- Never `.permit!`. Not even in tests. Not even temporarily.
- Explicitly list every permitted field. When in doubt, leave it out.

### Authentication
- Never build custom auth from scratch without flagging it. Prefer Devise or equivalent.
- Passwords: minimum 12 characters, complexity enforced, bcrypt only.
- Account lockout after repeated failures (`Devise :lockable`).
- Email confirmation before first login (`Devise :confirmable`).
- Always rotate session ID after login: `reset_session` before setting session data.
- Session identifiers MUST have minimum 256-bit entropy. Use `SecureRandom.urlsafe_base64(32)` — never
  `SecureRandom.hex(16)` or anything weaker.
- Logout must invalidate the session server-side, not just clear the cookie client-side.
- Password reset: time-limited tokens (max 2 hours), single-use, invalidated after use.

### Sessions & Cookies
```ruby
Rails.application.config.session_store :cookie_store,
  key: '_app_session',
  secure: Rails.env.production? || Rails.env.staging?,
  httponly: true,
  same_site: :strict,
  expire_after: 30.minutes
```
- Never store sensitive data in cookies or sessions — only session identifiers.
- Implement absolute session timeout regardless of activity for sensitive applications.

### State & Token Management
- **All critical application state (auth flows, sessions, tokens, PKCE verifiers) MUST be stored in
  the database** — NEVER in-memory (`Hash`, `Concurrent::Map`, class variables, global variables).
  In-memory state is lost on restart, causes data loss during deploys, and does not work across
  multiple processes/servers. No exceptions.
- **Never reuse URL-exposed tokens as permanent identifiers.** OAuth `state`, redirect params,
  query string tokens, and any value that appears in URLs, browser history, Referer headers, or
  server access logs must NEVER be used as a session ID or permanent auth credential. Always
  generate a NEW server-side identifier after authentication completes.
- **All tokens and states must be single-use with explicit consumption.** After a token is used
  (OAuth state consumed, PKCE verifier exchanged, magic link clicked), mark it as consumed or
  destroy it immediately. A second use must fail. Never leave tokens valid after use.
- **One-time handoff pattern for auth flows:**
  1. Generate a temporary public token (e.g., OAuth state) for the auth flow.
  2. On callback, consume the temporary token and generate a new private session identifier.
  3. Provide a one-time resolution endpoint: temporary token → real session ID.
  4. After resolution, invalidate the temporary token permanently.
- **Dead or unused auth code is a security liability.** Any auth-related code that is not wired into
  the application (e.g., not in controller tool registries, not referenced in routes) must be flagged
  with `# TODO [SECURITY-MED]: dead code — wire in or remove` and either connected or deleted in
  the same PR. Broken auth code that gets wired in later causes silent security failures.

### HTTP Security Headers
```ruby
config.action_dispatch.default_headers = {
  'X-Frame-Options'        => 'DENY',
  'X-Content-Type-Options' => 'nosniff',
  'X-XSS-Protection'       => '1; mode=block',
  'Referrer-Policy'        => 'strict-origin-when-cross-origin',
  'Permissions-Policy'     => 'geolocation=(), microphone=(), camera=()'
}
```

### SSL & HTTPS
- `config.force_ssl = true` in both production and staging — no exceptions.
- Set HSTS with minimum 1-year max-age in production:
```ruby
config.ssl_options = { hsts: { max_age: 31_536_000, subdomains: true } }
```

### Input Validation
- Validate all user input at the model level. Frontend validation is cosmetic only.
- Reject and log oversized payloads at the rack level.
- For user-supplied IDs in URLs, always scope to the current user:
```ruby
# NEVER: Resource.find(params[:id])
# ALWAYS: current_user.resources.find(params[:id])
```

---

## 3. DATABASE

### Migrations
- Every migration must be reversible (implement `down`) unless truly irreversible (document why).
- Always add indexes for: foreign keys, columns used in `where`, `order`, `group`, and `join`.
- Use `add_index` with `unique: true` for uniqueness constraints — don't rely on model validations alone.
- Never use `change_column` on a column with existing data without a backfill strategy.
- Never run destructive migrations (drop column, drop table, rename column) without a multi-step
  deployment plan — these cause downtime in zero-downtime deployments.
- Always add `null: false` and `default:` constraints at the database level for required fields.

### Query Performance
- Always check generated SQL for N+1 queries. Use `.includes`, `.eager_load`, or `.preload`
  when loading associations.
- Add `# TODO [PERF]: potential N+1` comment on any association access inside a loop.
- Use `.find_each` or `.in_batches` for bulk operations on large tables — never `.all.each`.
- Avoid `COUNT(*)` on large tables in synchronous requests — use cached counters.
- Never use `pluck` to load data that will be re-queried — batch and cache it.

### Data Integrity
- Foreign keys must exist at the database level, not just as Rails associations.
- Sensitive columns (tokens, PII, payment data, health data) MUST be encrypted at rest.
- Use database transactions for any operation that modifies more than one table.
- Never use raw `execute` with user input — always use parameterized queries.

### Expirable Records & Cleanup
- Every table with an `expires_at` column MUST have:
  1. A database index on `expires_at` for efficient cleanup queries.
  2. An `expired` scope: `scope :expired, -> { where("expires_at < ?", Time.current) }`
  3. A cleanup mechanism (background job, initializer thread, or scheduled task) that periodically
     deletes or archives expired records. Expired records must never accumulate unbounded.
- Time-limited records (OAuth states, PKCE verifiers, password reset tokens, magic links, OTPs)
  must have a maximum TTL set at creation time — never open-ended.

### Production Safety
- Never run `rails db:drop`, `rails db:reset`, or `rails db:seed` in production.
- Never run migrations without a rollback plan and a backup.
- Use `strong_migrations` gem to catch dangerous migration patterns before they run.

---

## 4. API DEVELOPMENT

### Authentication
- All API endpoints must require authentication. No endpoint is public by default.
- Use Bearer tokens or JWTs. Tokens must be stored hashed in the database.
- Tokens must be scoped (read vs. read-write), expirable, and revocable.
- Never accept tokens via query parameters — always via `Authorization` header.
- When implementing OAuth/PKCE flows: store all flow state (verifiers, redirect URIs, states)
  in the database — never in-memory. The OAuth `state` parameter is URL-exposed and must be
  treated as a public value — never promote it to a session identifier.
- API secrets/tokens used for endpoint protection must be compared using
  `ActiveSupport::SecurityUtils.secure_compare` — never plain `==` (timing attack prevention).

### Rate Limiting
- Every API must have rate limiting using `rack-attack` or equivalent:
```ruby
Rack::Attack.throttle("api/ip", limit: 300, period: 5.minutes) { |req| req.ip }
Rack::Attack.throttle("login/ip", limit: 5, period: 60) do |req|
  req.ip if req.path == "/users/sign_in" && req.post?
end
```
- Auth endpoints (login, password reset, OTP) must have stricter limits than regular API endpoints.

### Response Safety
- Never return stack traces, internal error messages, or raw exception details in API responses.
- Always return consistent error envelopes:
```json
{ "error": { "code": "unauthorized", "message": "Authentication required" } }
```
- Filter sensitive fields from all JSON responses — never serialize passwords, tokens, or keys.

### Versioning
- Always version APIs from the start: `/api/v1/`. Never change behavior of a published version.

### CORS
- Never use `origins '*'` in production or staging.
- Explicitly whitelist allowed origins per environment.
- Restrict methods and headers to only what is required.

---

## 5. BACKGROUND JOBS

- Never perform blocking operations in synchronous request/response cycles — offload to jobs.
- Always set a queue and priority for every job class.
- Always implement retry logic with exponential backoff and a max retry count.
- Always implement a `discard_on` clause for non-retryable errors (e.g., record not found).
- Never assume a job will run exactly once — jobs must be idempotent.
- Never pass ActiveRecord objects directly to jobs — pass IDs and re-fetch inside the job.
- Log job start, completion, and failure with contextual data (job class, arguments, duration).
- For long-running jobs, emit progress updates to a cache layer (Redis) for UI polling.
- In production, use a dedicated job monitoring tool (Sidekiq Web, GoodJob dashboard).

---

## 6. FILE UPLOADS

- Never trust user-supplied file names or MIME types — validate both server-side.
- Always use Active Storage with explicit content-type validation:
```ruby
validates :document, content_type: ['application/pdf', 'image/png', 'image/jpeg']
validates :avatar,   content_type: /\Aimage\/.*\z/, size: { less_than: 5.megabytes }
```
- Never store uploads on the application server filesystem in production.
- Always use a private S3 bucket (or equivalent) — never public-read buckets.
- Generate time-limited signed URLs for file access — never expose direct S3 URLs.
- Scan uploads for malware before making them available to other users.
- Always set a maximum file size limit at both the model and the rack middleware level.

---

## 7. SECRETS & ENVIRONMENT CONFIGURATION

- NEVER hard-code credentials, API keys, passwords, tokens, or secrets anywhere in code.
- NEVER commit `.env`, `config/master.key`, `config/credentials/*.key` to version control.
- These files MUST always be in `.gitignore`:
  - `.env`, `.env.*`
  - `config/master.key`
  - `config/credentials/production.key`
  - `config/credentials/staging.key`
- Use Rails encrypted credentials per environment:
  - `config/credentials/development.yml.enc`
  - `config/credentials/staging.yml.enc`
  - `config/credentials/production.yml.enc`
- In CI/CD: inject `RAILS_MASTER_KEY` via environment variable from a secrets manager.
- In production: store master keys in AWS Secrets Manager, HashiCorp Vault, or equivalent.
- Validate required ENV variables at boot time:
```ruby
# config/initializers/required_env.rb
required = %w[SECRET_KEY_BASE DATABASE_URL REDIS_URL]
required.each do |var|
  raise "Missing required ENV variable: #{var}" if ENV[var].blank?
end
```
- Rotate credentials quarterly. Rotate immediately after any suspected exposure.

---

## 8. LOGGING & MONITORING

### What to NEVER Log
- Passwords or password hashes
- Tokens, API keys, JWTs, session IDs
- Credit card numbers, SSNs, or any PII
- Any field listed in `config.filter_parameters`

### filter_parameters — Always Include
```ruby
# config/initializers/filter_parameter_logging.rb
Rails.application.config.filter_parameters += [
  :password, :password_confirmation, :token, :secret,
  :api_key, :credit_card, :ssn, :otp, :auth_token,
  :access_token, :refresh_token, :jwt, :session_id,
  :code, :state, :code_verifier
]
```

### What to Always Log
- Login successes and failures (with IP, user agent, timestamp)
- Password reset requests and completions
- Permission denied events (with user ID, resource, action)
- Admin actions (who did what, on which record, when)
- Background job failures with full context
- Third-party API errors (without exposing the request payload in full)

### Monitoring
- Use a centralized logging service in production and staging (Datadog, Papertrail, Logtail, etc.).
- Never rely on server disk logs alone in any environment above local.
- Set up alerts for: repeated auth failures, 500 error rate spikes, unusual data access volume.
- Use an error tracking service (Sentry, Bugsnag, Rollbar) in production and staging.

---

## 9. DEPENDENCY MANAGEMENT

- Run after every `bundle add` or `bundle update`:
```bash
bundle audit check --update   # Check for CVEs in gems
brakeman -A                   # Static security analysis
```
- Never add a gem that has not been updated in the last 2 years without flagging it.
- Pin gem versions in `Gemfile` for production — use `~>` for patch-level flexibility only.
- Keep Ruby version, Rails version, and all gems up to date — check for CVEs before deploying.
- Review the full diff of any gem upgrade that touches authentication, authorization,
  cryptography, or database adapters before merging.

---

## 10. TESTING

- Every new model, controller, service object, and job needs test coverage.
- Security-critical code (auth, authorization, tokens, file uploads) needs explicit security tests:
  - Test that unauthenticated requests are rejected (401)
  - Test that unauthorized users cannot access other users' data (403/404)
  - Test that SQL injection attempts are rejected
  - Test that mass assignment of protected fields is rejected
  - Test token expiry and revocation
- Never mock authentication in integration tests unless absolutely necessary.
- Use `brakeman` in CI — fail the build on any new WARNING or higher finding.
- Use `bundle audit` in CI — fail the build on any unpatched CVE.
- Test all background jobs in isolation AND in integration with the feature that enqueues them.
- Factory defaults must not create objects that bypass validations (`skip_validation: true` is banned).

---

## 11. ERROR HANDLING

- Never rescue `Exception` — rescue specific error classes.
- Never expose error details to end users — log server-side, show generic message to user.
- **API error responses must NEVER include exception class names, stack traces, or internal error
  messages.** Always return a generic message (e.g., `"Internal server error"`) and log the full
  error server-side with context. This applies to JSON-RPC, REST, and GraphQL endpoints alike.
- Always define friendly error pages: `public/404.html`, `public/422.html`, `public/500.html`.
- In staging and production:
```ruby
config.consider_all_requests_local = false
config.exceptions_app = self.routes
```
- Use `rescue_from` in `ApplicationController` for common errors:
```ruby
rescue_from ActiveRecord::RecordNotFound, with: :not_found
rescue_from Pundit::NotAuthorizedError,   with: :forbidden

def not_found  = render file: "public/404.html", status: :not_found,  layout: false
def forbidden  = render file: "public/403.html", status: :forbidden,  layout: false
```

---

## 12. DEPLOYMENT & CI/CD

### Before Every Deploy
- `brakeman -A` — zero new warnings allowed
- `bundle audit check --update` — zero unpatched CVEs allowed
- Full test suite green — zero failures allowed
- No hard-coded secrets in staged files: `git diff --cached | grep -iE "(key|secret|password|token)"`

### CI/CD Pipeline Requirements
- Automated test suite on every push and pull request
- `brakeman` and `bundle audit` on every push — fail build on findings
- Automated staging deploy on merge to main branch
- Production deploys only via pipeline — never from a local machine
- Separate environment variables per environment — never reuse production secrets in staging

### Zero-Downtime Deployment (Production)
- Multi-step migration strategy for any breaking database change:
  1. Deploy code that handles both old and new schema
  2. Run migration
  3. Deploy code that uses only new schema
- Use `strong_migrations` gem to enforce this pattern
- Always take a database backup before any migration in production

### Rollback Plan
- Every deploy must have a documented rollback procedure before it runs
- Keep the previous release available for at least 24 hours after a production deploy

---

## 13. CODE REVIEW — PRE-SUGGESTION CHECKLIST

Before writing or suggesting ANY code, verify:

- [ ] No secrets or credentials in code
- [ ] All user input validated at model level
- [ ] Authorization checked on every action
- [ ] CSRF protection in place
- [ ] No raw SQL with user input
- [ ] No `html_safe`/`raw` on user content
- [ ] Strong parameters used — no `.permit!`
- [ ] File uploads validated for type and size
- [ ] Errors handled — no stack traces exposed to users
- [ ] Sensitive fields in `filter_parameters`
- [ ] Session handling uses HttpOnly + Secure + SameSite
- [ ] No N+1 queries
- [ ] No sequential IDs in public-facing URLs
- [ ] Background jobs are idempotent
- [ ] No gem added without `bundle audit`
- [ ] Migrations are reversible and safe for zero-downtime
- [ ] Test coverage exists or has been flagged as missing
- [ ] No in-memory storage for auth/session state — everything in the database
- [ ] No URL-exposed tokens reused as session identifiers
- [ ] All tokens/states are single-use with explicit consumption
- [ ] Every table with `expires_at` has a cleanup mechanism
- [ ] Dead/unused auth code flagged or removed
- [ ] Session IDs use minimum 256-bit entropy (`SecureRandom.urlsafe_base64(32)`)

If any item is unresolved → add `# TODO [SECURITY-HIGH]:`, `# TODO [SECURITY-MED]:`,
or `# TODO [PERF]:` inline before proceeding. Never silently skip.

---

## 14. MARKING ISSUES INLINE

Use these severity markers for any issue that cannot be fixed immediately:

```ruby
# TODO [SECURITY-HIGH]: <description> — must fix before production
# TODO [SECURITY-MED]: <description> — fix in next sprint
# TODO [SECURITY-LOW]: <description> — address in roadmap
# TODO [PERF]: <description> — performance concern
# TODO [TECH-DEBT]: <description> — refactoring needed
# TODO [COMPLIANCE]: <description> — GDPR/HIPAA/CCPA concern
```

---

## 15. WHEN UNSURE

1. Say so explicitly — never silently take the less-secure or simpler path.
2. Present the secure alternative alongside the simpler one.
3. Add an inline comment explaining the risk.
4. Suggest running `brakeman` or `bundle audit` to verify.
5. Never trade security, correctness, or data integrity for brevity or speed.
6. If a feature cannot be implemented securely with the current architecture,
   say so and propose what architecture change is needed first.
