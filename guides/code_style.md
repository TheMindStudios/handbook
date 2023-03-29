# Code Style

## Rubocop
Rubocop is a linter as well as a code formatter specifically for Ruby.

We have strong and opinionated coding style guidelines that are enforced using this tool.

### Add the gems

First, let's add the relevant gems to our Gemfile:

```ruby
# previous gems as it was

group :development, :test do
  # previous gems under this group as it was

  # For code formatting and linting
  gem 'rubocop', require: false
  gem 'rubocop-performance', require: false
  gem 'rubocop-rails', require: false
  gem 'rubocop-rspec', require: false
end
# other gems if any
```

Now install the gems by running the following from the terminal:

```
bundle install
```

### Add the config

Add our Rubocop config to the root of your project:

```yaml
require:
  - rubocop-performance
  - rubocop-rails
  - rubocop-rspec

AllCops:
  TargetRubyVersion: 2.4 #change to your project ruby version

  Exclude:
    - 'vendor/**/*'
    - 'public/**/*'
    - 'test/fixtures/**/*'
    - 'db/**/*'
    - 'bin/**/*'
    - 'log/**/*'
    - 'tmp/**/*'
    - 'app/views/**/*'
    - 'config/environments/*'
    - 'node_modules/**/*'

# Metrics Cops

Metrics/ClassLength:
  Description: 'Avoid classes longer than 250 lines of code.'
  Max: 250
  Enabled: true

Metrics/LineLength:
  Max: 120

Metrics/MethodLength:
  Description: 'Avoid methods longer than 30 lines of code.'
  StyleGuide: 'https://github.com/bbatsov/ruby-style-guide#short-methods'
  Max: 30
  Enabled: true

Metrics/ModuleLength:
  Description: 'Avoid modules longer than 250 lines of code.'
  Max: 250
  Enabled: true

Metrics/ParameterLists:
  Description: 'Pass no more than four parameters into a method.'
  Max: 4
  Enabled: true

Metrics/BlockLength:
  CountComments: false
  Max: 5
  Exclude:
    - 'config/**/*'
    - 'spec/**/*'
    - 'lib/tasks/**/*.rake'
  IgnoredMethods: # can be ExcludedMethods
    - 'aasm'

# Style Cops

Style/Documentation:
  Description: 'Document classes and non-namespace modules.'
  Enabled: false

Style/FrozenStringLiteralComment:
  Description: >-
    Add the frozen_string_literal comment to the top of files
    to help transition from Ruby 2.3.0 to Ruby 3.0.
  Enabled: false

Style/IfUnlessModifier:
  Description: >-
    Favor modifier if/unless usage when you have a
    single-line body.
  StyleGuide: 'https://github.com/bbatsov/ruby-style-guide#if-as-a-modifier'
  Enabled: false

Style/InlineComment:
  Description: 'Avoid inline comments.'
  Enabled: false

Style/LineEndConcatenation:
  Description: >-
    Use \ instead of + or << to concatenate two string literals at
    line end.
  Enabled: true

# Rails Cops

Rails/ApplicationController:
  Enabled: false

Rails/Output:
  Description: "Don't leave puts-debugging"
  Enabled: true

Rails/FindEach:
  Description: "each could severely affect the performance, use find_each"
  Enabled: true

# Bundler Cops

Bundler/OrderedGems:
  Enabled: false
```

### Running Rubocop on all Ruby files

The following needn't be run in your project at the moment, given that we haven't added any non-linted files to our git index.

But there are valid cases, like say adding a new Rubocop rule, and wanting to apply that and format all the Ruby files within the project.

In such cases run the following from the root of the project in your terminal:

```bash
bundle exec rubocop
```

The above command would output the offenses it finds. Some offenses are auto-correctable by Rubocop. But some are not.

We auto-correct the correctable ones by running:

```bash
bundle exec rubocop -a
```

That should fix all the safely correctable errors. The non-corrected ones should be manually corrected.

Moving forward, we won't be running these commands. Rather we will use git hooks to run these commands for us on modified files.

### Setup precommit Git hook

A pre-commit Git hook can re-format the files that are marked as **staged** by git add command before you commit.

Let's set up [lefhook](https://github.com/evilmartians/lefthook) to run the hooks.

Add lefthook gem to Gemfile:
```ruby
group :development do
  # previous gems under this group as it was

  gem 'lefthook'
end
```

Create and edit lefthook.yml:

```yaml
pre-commit:
  parallel: true
  commands:
    rubocop:
      files: git diff --name-only --staged
      glob: "*.rb"
      run: rubocop --force-exclusion {files}
```

Test it

```bash
lefthook install && lefthook run pre-commit
```
