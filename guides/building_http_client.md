# Building a HTTP Client

## Introduction

For building HTTP Client we are using [faraday](https://github.com/lostisland/faraday).

## Conventions

- Class should be named according to the service name it interacts with (e.g.: `FacebookClient`)
- Class should have suffix `Client` (e.g.: `FacebookClient`).
- Class should be stored inside `app/services` or `lib` directory (e.g.: `app/services/facebook_client.rb`).

## Examples

### FacebookClient

```ruby
# app/services/facebook_client.rb
class FacebookClient
  BASE_URL = 'https://graph.facebook.com'

  def initialize(access_token)
    @access_token = access_token
  end

  def me(fields)
    conn.get '/v2.12/me', fields: fields
  end

  private

  def conn
    Faraday.new(url: BASE_URL, params: { access_token: @access_token }) do |f|
      f.request :json
      f.response :json
    end
  end
end
```

```ruby
## usage

facebook_client = FacebookClient.new(access_token)
facebook_client.me
```

## References

- [faraday](https://github.com/lostisland/faraday)
- [Middleware](https://github.com/lostisland/awesome-faraday/#middleware)
