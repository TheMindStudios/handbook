# Configuration

We love **Convention over Configuration** paradigm our favorite framework Ruby on Rails follows.
This article proposes some approaches for **Convention _for_ Configuration**.

## Credentials and Secrets

Credentials and Secrets should never be committed into any repository, at least in unencrypted way.
There several ways to store such kind of sensitive data:

* Rails [Custom Credentials](https://guides.rubyonrails.org/security.html#custom-credentials)
* [ENV Variables](#env-variables)
* Other secure storages, e.g. [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)

We recommend pulling Credentials and Secrets into appropriate application or service configurations,
instad of calling credentials storage directly inside the code.
This approach will provide clean and unified access to all config settings, e.g.:

```ruby
# config/secrets.yml
some_api:
  uri: 'https://api.example.com'
  key: <%= Rails.application.credentials.some_api_key! %>
  token: <%= ENV.fetch('SOME_API_TOKEN') %>

# usage
Rails.application.secrets.some_api[:uri]
# or if you have newer Rails version with credentials
Rails.application.credentials.some_api[:uri]
```

## ENV Variables

ENV Variables is a classic way to store environment-specific settings, secrets, or just even some application settings.

- use gem `dotenv` - https://github.com/bkeepers/dotenv
- file `.env` should be committed to a repository as a reference of all variables used in the project with empty or default values

## Class/Object Configuration

* [`ActiveSupport::Configurable`](https://api.rubyonrails.org/classes/ActiveSupport/Configurable.html)
* [`Dry::Configurable`](https://dry-rb.org/gems/dry-configurable/)

## Service Module Configuration

See [Service Modules](https://github.com/themindstudios/handbook/blob/master/guides/service_modules.md).

We recommend starting with constant as a fastest and simple approach, and then switch to application or service configuration when you need more flexibility.
Thankfully to the unified interface, you won't need to change the code calling for the configs,
because any of the approaches below will provide with access to config value via `MyService.config.key`.

- Constant ("plain old ruby constant") + method `config` for a convenient and unified interface:
```ruby
# services/my_service.rb
module MyService
  CONFIG = {
    key: 'value'
  }

  module_function

  def config
    @config ||= ActiveSupport::OrderedOptions.new.update(CONFIG)
  end
end
```

- Section in the application configuration file:
```ruby
# config/secrets.yml
services:
  my_service:
    key: 'value'

# services/my_service.rb
module MyService
  module_function

  def config
    @config ||= Rails.application.secrets.services.my_service
  end
end
```

- Dedicated configuration file for a Service Module:
```ruby
# config/services/my_service.yml
default: &default
  key: 'value'

development:
  <<: *default

test:
  <<: *default

production:
  <<: *default

# services/my_service.rb
module MyService
  module_function

  def config
    @config ||= Rails.application.config_for('services/my_service')
  end
end
```

## References
- https://github.com/bkeepers/dotenv
- https://api.rubyonrails.org/classes/ActiveSupport/OrderedOptions.html
- https://api.rubyonrails.org/classes/Rails/Application.html#method-i-config_for
- https://guides.rubyonrails.org/security.html#custom-credentials
- https://guides.rubyonrails.org/configuring.html#custom-configuration
- https://api.rubyonrails.org/classes/ActiveSupport/Configurable.html
- https://dry-rb.org/gems/dry-configurable/
- https://kukicola.io/posts/useful-active-support-features-you-may-not-have-heard-of/
