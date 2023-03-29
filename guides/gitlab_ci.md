# Gitlab CI

We use Gitlab CI to automate code style checking and run tests before reviewing merge request.

## Common example

There is an example for project which use:
- Postgres
- RSpec
- Rubocop
- Commitlint

Place Gitlab CI config file under your project `.gitlab-ci.yml`

```yaml
stages:
  - lint
  - test

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - vendor/bundle

.build:
  image: ruby:3.2.0-slim-buster
  variables:
    BUNDLE_VERSION: 2.4.1
    BUNDLE_PATH: vendor/bundle
    NODE_MAJOR: 16
  cache:
    key: $CI_COMMIT_REF_SLUG
    paths:
      - vendor/ruby
    policy: pull
  before_script:
    - apt-get update -qq && DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends build-essential gnupg2 curl wget git
    - echo "deb https://apt.postgresql.org/pub/repos/apt buster-pgdg main" > /etc/apt/sources.list.d/pgdg.list
    - wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -
    - apt-get update -qq && DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends libpq-dev postgresql-client-14
    - gem install bundler:$BUNDLE_VERSION --no-document
    - bundle install --jobs=$(nproc) "${FLAGS[@]}" --path=vendor

.db:
  extends: .build
  services:
    - postgres:14-alpine
  variables:
    POSTGRES_USER: app
    POSTGRES_HOST_AUTH_METHOD: trust
    DB_USERNAME: app
    DB_HOST: postgres
    RAILS_ENV: test
    DISABLE_SPRING: 1
  before_script:
    - apt-get update -qq && DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends build-essential gnupg2 curl wget git
    - echo "deb https://apt.postgresql.org/pub/repos/apt buster-pgdg main" > /etc/apt/sources.list.d/pgdg.list
    - wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -
    - apt-get update -qq && DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends libpq-dev postgresql-client-14
    - gem install bundler:$BUNDLE_VERSION --no-document
    - bundle install --jobs=$(nproc) "${FLAGS[@]}" --path=vendor
    - echo "$MASTER_KEY" > config/master.key
    - bundle exec rails db:create db:schema:load --trace

commitlint:
  extends: .build
  stage: lint
  before_script:
    - apt-get update -qq && DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends build-essential gnupg2 curl
    - curl -sL https://deb.nodesource.com/setup_$NODE_MAJOR.x | bash -
    - apt-get update -qq && DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends nodejs
    - npm install -g @commitlint/cli @commitlint/config-conventional
  script:
    - echo "${CI_COMMIT_MESSAGE}" | commitlint

rubocop:
  extends: .build
  stage: lint
  script:
    - bundle exec rubocop

rspec:
  extends: .db
  stage: test
  script:
    - bundle exec rspec --profile 10 --format progress
```