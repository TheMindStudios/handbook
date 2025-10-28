# Localization

## Table of Contents
- [Why Localization Matters](#why-localization-matters)
- [Key Storage Structure](#key-storage-structure)
- [Separation by Scopes](#separation-by-scopes)
- [Application Configuration](#application-configuration)
- [Locale Detection](#locale-detection)
- [Preventing Hardcoded Strings](#preventing-hardcoded-strings)
- [Using I18n Tasks](#using-i18n-tasks)
- [Best Practices](#best-practices)
- [CI/CD Integration](#cicd-integration)

## Why Localization Matters

Proper localization setup is a critical investment that pays dividends when your client wants to expand to new markets. Without proper i18n structure:
- Adding a new language requires hunting through the entire codebase for hardcoded strings
- Translation costs increase significantly due to scattered text
- Risk of missing translations in production
- Weeks or months of developer time spent on refactoring

With proper localization from the start:
- Adding a new language takes hours, not weeks
- All text is centralized and easy to send to translators
- Consistent terminology across the application
- Professional translation services can work with standard YAML/JSON formats

**The key principle: Every user-facing string should be in a locale file, never hardcoded.**

## Key Storage Structure

All translation keys should be stored in `config/locales/` directory following Rails i18n conventions:

```
config/locales/
├── en.yml                    # Default English translations
├── uk.yml                    # Ukrainian translations (example)
├── en/
│   ├── admin/
│   │   ├── users.yml         # Admin users management
│   │   ├── dashboard.yml     # Admin dashboard
│   │   └── settings.yml      # Admin settings
│   ├── api/
│   │   ├── v1/
│   │   │   ├── users.yml     # API v1 user endpoints
│   │   │   ├── posts.yml     # API v1 post endpoints
│   │   │   └── errors.yml    # API v1 error messages
│   │   └── v2/
│   │       └── users.yml     # API v2 user endpoints
│   ├── models/
│   │   ├── user.yml          # User model attributes
│   │   └── post.yml          # Post model attributes
│   ├── views/
│   │   ├── common.yml        # Shared view translations
│   │   └── mailers.yml       # Email templates
│   └── shared/
│       ├── buttons.yml       # Common button labels
│       ├── validations.yml   # Validation messages
│       └── notifications.yml # Notification texts
└── uk/                       # Mirror structure for Ukrainian
    ├── admin/
    ├── api/
    └── ...
```

### Naming Conventions

**For API responses:**
```yaml
# config/locales/en/api/v1/users.yml
en:
  api:
    v1:
      users:
        created: "User successfully created"
        updated: "User successfully updated"
        not_found: "User not found"
        errors:
          invalid_email: "Email is invalid"
          password_too_short: "Password must be at least 8 characters"
```

**For Admin panel:**
```yaml
# config/locales/en/admin/users.yml
en:
  admin:
    users:
      index:
        title: "Users Management"
        new_user: "Create New User"
      form:
        email_label: "Email Address"
        role_label: "User Role"
        submit: "Save User"
```

**For Models:**
```yaml
# config/locales/en/models/user.yml
en:
  activerecord:
    models:
      user:
        one: "User"
        other: "Users"
    attributes:
      user:
        email: "Email"
        first_name: "First Name"
        last_name: "Last Name"
        created_at: "Registration Date"
```

## Separation by Scopes

### Admin Scope

Admin panel translations should be completely separate from API and public-facing content:

```ruby
# In admin controllers, set locale namespace
class Admin::UsersController < Admin::BaseController
  def index
    @title = t('.title') # Looks up admin.users.index.title
    @users = User.all
  end
end
```

```erb
<!-- app/views/admin/users/index.html.erb -->
<h1><%= t('.title') %></h1>
<%= link_to t('.new_user'), new_admin_user_path, class: 'btn' %>
```

### API Scope

API responses should use versioned scopes for flexibility:

```ruby
# app/controllers/api/v1/users_controller.rb
module Api
  module V1
    class UsersController < Api::BaseController
      def create
        user = User.create(user_params)

        if user.persisted?
          render json: { message: t('api.v1.users.created') }, status: :created
        else
          render json: {
            errors: user.errors.full_messages
          }, status: :unprocessable_entity
        end
      end
    end
  end
end
```

### Shared Translations

For common elements used across admin and API:

```yaml
# config/locales/en/shared/validations.yml
en:
  shared:
    validations:
      required: "This field is required"
      invalid_format: "Invalid format"
      too_short: "Must be at least %{count} characters"
      too_long: "Cannot exceed %{count} characters"
```

```ruby
# Usage in custom validators or service objects
I18n.t('shared.validations.required')
I18n.t('shared.validations.too_short', count: 8)
```

## Application Configuration

### Adding a New Language

When adding a new language to your Rails application, you need to configure several settings in `config/application.rb` or `config/initializers/locale.rb`.

**config/application.rb:**
```ruby
module YourApp
  class Application < Rails::Application
    # Set default locale to :en
    config.i18n.default_locale = :en

    # Whitelist available locales
    config.i18n.available_locales = [:en, :uk, :es, :de]

    # Load all locale files from nested directories
    config.i18n.load_path += Dir[Rails.root.join('config', 'locales', '**', '*.{rb,yml}')]

    # Raise exception on missing translations (recommended for development/test)
    config.i18n.raise_on_missing_translations = true

    # Fallback to default locale when translation is missing
    config.i18n.fallbacks = [:en]
  end
end
```

Or create a dedicated initializer `config/initializers/locale.rb`:
```ruby
# Set default locale
I18n.default_locale = :en

# Set available locales
I18n.available_locales = [:en, :uk, :es, :de]

# Enable fallbacks
I18n.fallbacks = [:en]

# Optionally, configure fallback chains per locale
# Ukrainian falls back to Russian, then English
# Spanish falls back to English
I18n.fallbacks = {
  uk: [:uk, :ru, :en],
  es: [:es, :en],
  de: [:de, :en]
}

# Exception handler for missing translations (optional)
I18n.exception_handler = lambda do |exception, locale, key, options|
  if exception.is_a?(I18n::MissingTranslation)
    Rails.logger.warn "Missing translation: #{key} for locale #{locale}"
    "Translation missing: #{key}"
  else
    raise exception
  end
end
```

### Environment-Specific Configuration

**Development/Test** - Raise errors on missing translations:
```ruby
# config/environments/development.rb
config.i18n.raise_on_missing_translations = true
config.i18n.fallbacks = false
```

**Production** - Use fallbacks gracefully:
```ruby
# config/environments/production.rb
config.i18n.raise_on_missing_translations = false
config.i18n.fallbacks = [:en]
```

### Locale Files Configuration

Ensure your i18n-tasks configuration matches available locales:
```yaml
# config/i18n-tasks.yml
base_locale: en
locales: [en, uk, es, de]  # Update when adding new languages

data:
  read:
    - config/locales/%{locale}/**/*.yml
  write:
    - config/locales/%{locale}/%{namespace}.yml
```

## Locale Detection

Rails needs a way to determine which locale to use for each request. Here are several approaches.

### Application Controller Setup

Basic locale setup in your base controller:
```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  around_action :switch_locale

  private

  def switch_locale(&action)
    locale = extract_locale || I18n.default_locale
    I18n.with_locale(locale, &action)
  end

  def extract_locale
    # Override in specific implementations below
  end
end
```

### Approach 1: URL Path Parameter

Best for public-facing websites where SEO matters (e.g., `/en/users`, `/uk/users`).

**Routes configuration:**
```ruby
# config/routes.rb
Rails.application.routes.draw do
  scope '(:locale)', locale: /#{I18n.available_locales.join('|')}/ do
    resources :users
    resources :posts

    namespace :admin do
      resources :users
      resources :dashboard
    end
  end
end
```

**Controller implementation:**
```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  around_action :switch_locale
  before_action :set_locale_from_params

  private

  def switch_locale(&action)
    locale = extract_locale || I18n.default_locale
    I18n.with_locale(locale, &action)
  end

  def extract_locale
    locale_from_params || locale_from_user || locale_from_headers
  end

  def locale_from_params
    locale = params[:locale]
    return locale if locale.present? && I18n.available_locales.include?(locale.to_sym)
  end

  def locale_from_user
    # If user is logged in, use their preferred language
    current_user&.locale if current_user&.locale.present?
  end

  def locale_from_headers
    # Fallback to Accept-Language header
    extract_locale_from_accept_language_header
  end

  def extract_locale_from_accept_language_header
    return nil unless request.headers['Accept-Language']

    # Parse Accept-Language header (e.g., "en-US,en;q=0.9,uk;q=0.8")
    accepted_locales = request.headers['Accept-Language']
                              .split(',')
                              .map { |lang| lang.split(';').first.split('-').first.to_sym }

    # Find first matching available locale
    (accepted_locales & I18n.available_locales).first
  end

  def set_locale_from_params
    # Store locale in session for consistency
    session[:locale] = params[:locale] if params[:locale].present?
  end

  def default_url_options
    # Ensure locale is included in all generated URLs
    { locale: I18n.locale }
  end
end
```

### Approach 2: Accept-Language Header (API-First)

Best for APIs where clients control the language via HTTP headers.

**Controller implementation:**
```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  around_action :switch_locale

  private

  def switch_locale(&action)
    locale = extract_locale || I18n.default_locale
    I18n.with_locale(locale, &action)
  end

  def extract_locale
    locale_from_headers || I18n.default_locale
  end

  def locale_from_headers
    return nil unless request.headers['Accept-Language']

    # Parse Accept-Language: en-US,en;q=0.9,uk;q=0.8
    parsed = parse_accept_language_header

    # Find best matching locale
    parsed.find { |locale, _| I18n.available_locales.include?(locale.to_sym) }&.first&.to_sym
  end

  def parse_accept_language_header
    request.headers['Accept-Language']
      .split(',')
      .map do |lang|
        locale, quality = lang.split(';')
        quality = quality ? quality.split('=').last.to_f : 1.0
        [locale.split('-').first, quality]
      end
      .sort_by { |_, quality| -quality }  # Sort by quality (highest first)
  end
end
```

**API Response with locale information:**
```ruby
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < ApplicationController
      before_action :set_locale_header

      private

      def set_locale_header
        response.headers['Content-Language'] = I18n.locale.to_s
      end
    end
  end
end
```

### Approach 3: Middleware (Advanced)

For complex applications requiring locale detection before controller actions.

**Create middleware:**
```ruby
# lib/middleware/locale_detector.rb
module Middleware
  class LocaleDetector
    def initialize(app)
      @app = app
    end

    def call(env)
      locale = detect_locale(env)
      I18n.locale = locale if locale

      @app.call(env)
    end

    private

    def detect_locale(env)
      request = ActionDispatch::Request.new(env)

      # Priority order:
      # 1. URL parameter (?locale=uk or /uk/...)
      # 2. Session
      # 3. Cookie
      # 4. Accept-Language header
      # 5. Default locale

      locale_from_param(request) ||
        locale_from_session(env) ||
        locale_from_cookie(request) ||
        locale_from_header(request) ||
        I18n.default_locale
    end

    def locale_from_param(request)
      # Check URL path (e.g., /uk/users)
      locale = request.path.split('/')[1]
      return locale.to_sym if valid_locale?(locale)

      # Check query parameter (e.g., ?locale=uk)
      locale = request.params['locale']
      locale.to_sym if valid_locale?(locale)
    end

    def locale_from_session(env)
      locale = env['rack.session']&.[](:locale)
      locale.to_sym if valid_locale?(locale)
    end

    def locale_from_cookie(request)
      locale = request.cookies['locale']
      locale.to_sym if valid_locale?(locale)
    end

    def locale_from_header(request)
      return nil unless request.headers['Accept-Language']

      accepted = request.headers['Accept-Language']
                        .split(',')
                        .map { |l| l.split(';').first.split('-').first }

      accepted.find { |l| valid_locale?(l) }&.to_sym
    end

    def valid_locale?(locale)
      locale.present? && I18n.available_locales.include?(locale.to_sym)
    end
  end
end
```

**Register middleware:**
```ruby
# config/application.rb
module YourApp
  class Application < Rails::Application
    # ... other configuration ...

    # Add locale detection middleware
    config.middleware.insert_before ActionDispatch::Static, Middleware::LocaleDetector
  end
end
```

**Store user preference:**
```ruby
# app/controllers/locales_controller.rb
class LocalesController < ApplicationController
  def update
    locale = params[:locale]

    if I18n.available_locales.include?(locale.to_sym)
      session[:locale] = locale
      cookies[:locale] = { value: locale, expires: 1.year.from_now }

      # Update user preference if logged in
      current_user&.update(locale: locale)

      redirect_back(fallback_location: root_path, notice: t('locale.changed'))
    else
      redirect_back(fallback_location: root_path, alert: t('locale.invalid'))
    end
  end
end
```

**Routes:**
```ruby
# config/routes.rb
Rails.application.routes.draw do
  post '/locale', to: 'locales#update', as: :change_locale
end
```

### Testing Locale Detection

```ruby
# spec/requests/locale_detection_spec.rb
require 'rails_helper'

RSpec.describe 'Locale Detection', type: :request do
  describe 'GET /users' do
    context 'with locale in URL' do
      it 'uses URL locale' do
        get '/uk/users'
        expect(I18n.locale).to eq(:uk)
      end
    end

    context 'with Accept-Language header' do
      it 'uses header locale' do
        get '/users', headers: { 'Accept-Language' => 'es,en;q=0.9' }
        expect(I18n.locale).to eq(:es)
      end
    end

    context 'with no locale specified' do
      it 'uses default locale' do
        get '/users'
        expect(I18n.locale).to eq(:en)
      end
    end
  end
end
```

## Preventing Hardcoded Strings

### In ERB Templates

**Bad:**
```erb
<h1>Welcome to our application</h1>
<button>Submit</button>
<p>Please enter your email address</p>
```

**Good:**
```erb
<h1><%= t('.welcome_message') %></h1>
<button><%= t('.submit_button') %></button>
<p><%= t('.email_instruction') %></p>
```

### In Ruby Code

**Bad:**
```ruby
class UserService
  def create_user(params)
    user = User.create(params)
    return { message: "User created successfully" } if user.persisted?

    { error: "Failed to create user" }
  end
end
```

**Good:**
```ruby
class UserService
  def create_user(params)
    user = User.create(params)
    return { message: I18n.t('services.user.created') } if user.persisted?

    { error: I18n.t('services.user.creation_failed') }
  end
end
```

### In Mailers

**Bad:**
```ruby
class UserMailer < ApplicationMailer
  def welcome_email(user)
    @user = user
    mail(
      to: user.email,
      subject: "Welcome to Our Platform!"
    )
  end
end
```

**Good:**
```ruby
class UserMailer < ApplicationMailer
  def welcome_email(user)
    @user = user
    mail(
      to: user.email,
      subject: t('mailers.user.welcome.subject')
    )
  end
end
```

### Tools to Detect Hardcoded Strings

#### ERB-Lint Configuration

Add to `.erb-lint.yml`:

```yaml
linters:
  HardCodedString:
    enabled: true
    addendum:
      - "Use I18n.t() instead"
```

Run check:
```bash
bundle exec erblint --lint-all
```

#### Rubocop-i18n Configuration

Add to `.rubocop.yml`:

```yaml
require:
  - rubocop-i18n

# Added in version 2.14 of rubocop-rails
Rails/I18nLocaleTexts:
  Enabled: true

Rails/I18nLazyLookup:
  Enabled: true
  EnforcedStyle: explicit

I18n/GetText:
  Enabled: false

I18n/RailsI18n:
  Enabled: true

I18n/RailsI18n/DecorateStringFormattingUsingInterpolation:
  Enabled: false
```

Run check:
```bash
bundle exec rubocop
```

## Using I18n Tasks

[i18n-tasks](https://github.com/glebm/i18n-tasks) is a powerful gem for managing translations.

### Installation

Add to `Gemfile`:
```ruby
gem 'i18n-tasks', '~> 1.0.12', require: false
```

### Configuration

Create `config/i18n-tasks.yml`:

```yaml
base_locale: en
locales: [en, uk]

data:
  read:
    - config/locales/%{locale}/**/*.yml
  write:
    - config/locales/%{locale}/%{namespace}.yml

search:
  paths:
    - app/
  exclude:
    - app/assets/
    - app/javascript/

ignore_unused:
  - 'activerecord.*'
  - 'errors.messages.*'
  - 'date.*'
  - 'time.*'
  - 'number.*'
```

### Common Commands

**Find missing translations:**
```bash
bundle exec i18n-tasks missing
```

Output:
```
Missing translations (2):
  uk.admin.users.index.title
  uk.api.v1.users.created
```

**Find unused translations:**
```bash
bundle exec i18n-tasks unused
```

**Add missing keys with empty values:**
```bash
bundle exec i18n-tasks add-missing
```

**Remove unused keys:**
```bash
bundle exec i18n-tasks remove-unused
```

**Translate missing keys automatically (requires API key):**
```bash
bundle exec i18n-tasks translate-missing --from=en --to=uk
```

**Health check (finds all issues):**
```bash
bundle exec i18n-tasks health
```

**Normalize YAML files (sort keys, consistent formatting):**
```bash
bundle exec i18n-tasks normalize
```

## Best Practices

### 1. Use Lazy Lookup in Views

Instead of full path:
```erb
<%= t('admin.users.index.title') %>
```

Use relative path (Rails automatically looks up based on view path):
```erb
<%= t('.title') %>
```

### 2. Interpolation for Dynamic Content

```yaml
en:
  messages:
    welcome: "Welcome, %{name}!"
    items_count: "You have %{count} items"
```

```ruby
t('messages.welcome', name: user.name)
t('messages.items_count', count: 5)
```

### 3. Pluralization

```yaml
en:
  items:
    zero: "No items"
    one: "1 item"
    other: "%{count} items"
```

```ruby
t('items', count: 0)  # => "No items"
t('items', count: 1)  # => "1 item"
t('items', count: 5)  # => "5 items"
```

### 4. Default Values

```ruby
t('some.missing.key', default: 'Fallback text')
```

### 5. Rich Text with HTML

```yaml
en:
  messages:
    terms: "I agree to the <a href='%{url}'>Terms of Service</a>"
```

```erb
<%= t('messages.terms', url: terms_path).html_safe %>
```

Or better, use `_html` suffix for automatic html_safe:
```yaml
en:
  messages:
    terms_html: "I agree to the <a href='%{url}'>Terms of Service</a>"
```

```erb
<%= t('messages.terms_html', url: terms_path) %>
```

### 6. Date and Time Formatting

```yaml
en:
  time:
    formats:
      short: "%b %d, %Y"
      long: "%B %d, %Y at %I:%M %p"
```

```ruby
I18n.l(Time.now, format: :short)
```

### 7. Keep Translation Keys Organized

- Group by feature/namespace
- Use descriptive key names
- Avoid deeply nested structures (max 4-5 levels)
- Document complex interpolations

## CI/CD Integration

Add to `.gitlab-ci.yml`:

```yaml
lint_html:
  extends: .base
  stage: lint
  script:
    - bundle exec erblint --lint-all

lint_translations:
  extends: .base
  stage: lint
  script:
    - bundle exec i18n-tasks health
    - bundle exec i18n-tasks missing
    - bundle exec i18n-tasks unused

lint_ruby:
  extends: .base
  stage: lint
  script:
    - bundle exec rubocop
```

This ensures:
- No hardcoded strings in templates
- No hardcoded strings in Ruby code
- All translations are present
- No unused translation keys
- Consistent code style

## Summary Checklist

When working with localization:

- [ ] All user-facing text is in locale files
- [ ] Translations are organized by scope (admin, api, shared)
- [ ] Using lazy lookup (`.title`) in views
- [ ] ERB-Lint configured to detect hardcoded strings
- [ ] Rubocop-i18n configured to detect hardcoded strings
- [ ] i18n-tasks configured and health checks passing
- [ ] CI/CD checks for hardcoded strings and missing translations
- [ ] All locales have complete translations (no missing keys)
- [ ] No unused translation keys
- [ ] Proper interpolation used for dynamic content
- [ ] Pluralization rules defined where needed

With this setup, adding a new language becomes as simple as:
1. Create new locale file: `config/locales/es.yml`
2. Copy structure from `en/` to `es/`
3. Send YAML files to translator
4. Import translated files
5. Add locale to `I18n.available_locales`

**Time investment: A few hours vs. weeks of refactoring!**
