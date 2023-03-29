# Deployment

## Capistrano

You should use **capistrano** gem if you need to deploy project on EC2 or other remote VM.

Add these dependencies to your Gemfile:

```ruby
group :development, :deploy do
  gem 'capistrano', '~> 3.16', require: false
  gem 'capistrano3-puma', require: false
  gem 'capistrano-rails',   '~> 1.6.1', require: false
  gem 'capistrano-bundler', '~> 2.0.1', require: false
  gem 'capistrano-rvm', '~> 0.1', require: false
  gem 'capistrano-sidekiq', require: false
end
```

Run this command to add **capistrano** config files to project:
```bash
bundle exec cap install
```

Add to **Capfile** these lines:
```ruby
require 'capistrano/rvm'
# require 'capistrano/rbenv'
# require 'capistrano/chruby'
require 'capistrano/bundler'
require 'capistrano/puma'
require 'capistrano/rails/assets'
require 'capistrano/rails/migrations'
require 'capistrano/sidekiq'

install_plugin Capistrano::Puma
install_plugin Capistrano::Puma::Systemd

install_plugin Capistrano::Sidekiq  # Default sidekiq tasks
# Then select your service manager
install_plugin Capistrano::Sidekiq::Systemd
```

Add deploy configuration for dev environment.

```ruby
# config/deploy/dev.rb

role :app, %w[deploy@0.0.0.0] # ssh user and ip of remote server
role :web, %w[deploy@0.0.0.0]
role :db,  %w[deploy@0.0.0.0]

set :tmp_dir, '/home/app/tmp-app'
set :nginx_server_name, 'app.themindstudios.com' # not required but nice to mention
set :rails_env, 'dev'

set :deploy_to, '/var/www/app' # directory where project located on remote server
set :rvm_ruby_version, 'ruby-3.2.0@app' # set this from .ruby-version and .ruby-gemset
set :rvm_custom_path, '/opt/rvm'

set :puma_service_unit_name, 'app' # do not forget to create service in /etc/systemd/system

set :branch, :dev # git branch to deploy from

set :sidekiq_service_unit_name, 'app-sidekiq' # same as for puma
set :sidekiq_service_unit_user, :system
```

### Puma systemd service

```
[Unit]
Description=app
After=network.target

# Uncomment for socket activation (see below)
# Requires=puma.socket

[Service]
# Foreground process (do not use --daemon in ExecStart or config.rb)
Type=simple

# Preferably configure a non-privileged user
# User=deploy

# The path to the puma application root
# Also replace the "<WD>" place holders below with this path.

# Helpful for debugging socket activation, etc.
# Environment=PUMA_DEBUG=1
# The command to start Puma. This variant uses a binstub generated via
# `bundle binstubs puma --path ./sbin` in the WorkingDirectory
# (replace "<WD>" below)
WorkingDirectory=/var/www/app/current
User=deploy
Group=deploy
ExecStart=/opt/rvm/gems/ruby-3.2.0@app/wrappers/bundle exec puma -C /var/www/app/shared/puma.rb
# Variant: Use config file with `bind` directives instead:
# ExecStart=<WD>/sbin/puma -C config.rb
# Variant: Use `bundle exec --keep-file-descriptors puma` instead of binstub

Restart=always

[Install]
WantedBy=multi-user.target
```

### Sidekiq systemd service

```
[Unit]
Description=app-sidekiq
After=network.target

[Service]
Type=simple

WorkingDirectory=/var/www/app/current
User=deploy
Group=deploy

ExecStart=/opt/rvm/gems/ruby-3.2.0@app/wrappers/bundle exec sidekiq -e dev -C /var/www/app/current/config/sidekiq.yml

# SyslogIdentifier=app-sidekiq
StandardOutput=append:/var/www/app/shared/log/sidekiq.log
StandardError=append:/var/www/app/shared/log/sidekiq.log

Restart=always

[Install]
WantedBy=multi-user.target
```

### Gitlab CI

Example with pipeline to deploy dev branch.

```yaml
stages:
  - deploy

.deploy:
  image: ruby:3.2.0-alpine
  variables:
    BUNDLE_PATH: vendor/ruby
    BUNDLE_VERSION: 2.2.22
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - vendor/ruby
    policy: pull-push
  before_script:
    - apk add --no-cache --update build-base linux-headers
    - which ssh-agent || apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    # Add the SSH key stored in SSH_PRIVATE_KEY variable to the agent store
    - echo "$SSH_WEB_PRIVATE_KEY" | tr -d '\r' | ssh-add - > /dev/null
    - gem install bundler:$BUNDLE_VERSION --no-document
    - bundle config set --local with deploy
    - bundle config set --local without "default:development:test:production"
    - bundle install --jobs=$(nproc) --quiet

dev_deploy:
  extends: .deploy
  environment: dev
  stage: deploy
  script:
    - bundle exec cap dev deploy
  only:
    - dev
```