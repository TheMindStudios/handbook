# Building an API

## Introduction

API is an important part of web applications. They provide interfaces for communication with the app from the outside. We are using Rails default API mode to build APIs. It means that we don't use views, assets, and other things that are not needed for API.

## Conventions

- We create one parent `api_controller.rb` and all controllers for API inherit from it.
- All controllers are placed in `app/controllers/api/*` folder. It can be nested if needed. For example, if we need to create separate API for mobile clients, we can create `app/controllers/mobile/api/*` folder.
- We use `jbuilder` gem to render JSON responses. We create a separate folder for views: `app/views/api/*`.
- We document all API endpoints with Postman. You will be invited to Postman team related to our organisation. You can find more information about Postman [here](https://www.postman.com/). You need to create corresponding collection and environments for your project.

  If you need to update existing collection, please create a fork of it in your local workspace. After all changes are applied and tested, you can **Merge changes** to the original collection. Do not forget to **Pull changes** before it to avoid conflicts.

- We use `pagy` gem for pagination. On old projects, you can see `kaminari` gem.

  Basic response with pagination looks like this:

  ```ruby
  {
    "items": [
      # collection of items to be rendered
    ],
    "pagination": {
      "page": 1,
      "per_page": 10,
      "total_pages": 1,
      "total_count": 1
    }
  }
  ```

Sometimes you need to use **cursor-based pagination**. It is a technique for pagination that avoids many of the pitfalls of **per page** pagination. An endpoint with pagination params applied could look like this: `/api/v1/countries?page[after]=dQw4w9&page[size]=10`. `page[after]` can be replaced with `page[before]` if we want to retrieve records preceding a cursor. Then response for this type of pagination looks like this:

```ruby
{
  "items": [
    # collection of items to be rendered
  ],
  "pagination": {
    "after": "MjAyMC0wNS0xMlQxNTo1MzowMC4wMDBa",
    "before": "MjAyMC0wNS0xMlQxNTo1MzowMC4wMDBa",
    "size": 10,
  }
}
```

- We use `versioncake` gem for API versioning. You can find more information about it [here](https://github.com/bwillis/versioncake)

### Example Classes

```ruby
# app/controllers/api/base_controller.rb
class Api::BaseController < ApplicationController
  include Api::Concerns::ExceptionHandler
  include Api::Concerns::Pagination
end
```

```ruby
# app/controllers/api/users_controller.rb
class Api::UsersController < Api::BaseController
  before_action :authenticate_user!

  def index
    @pagy, @users = pagy(User.all, pagination_params)
  end
end
```

```ruby
# app/controllers/api/concerns/exception_handler.rb
module Api::Concerns::ExceptionHandler
  extend ActiveSupport::Concern

  included do
    rescue_from StandardError, with: :render_error
  end

  def render_error(exception)
    # your error handling logic
  end
end
```

```ruby
# app/controllers/api/concerns/pagination.rb
module Api::Concerns::Pagination
  extend ActiveSupport::Concern

  def pagination_params
    params.permit(:page, :items).to_h.each_with_object({}) do |(key, value), memo|
      memo[key.to_sym] = value.to_i
    end
  end
end
```

## HTTP headers and status codes

Some of the more commonly used HTTP status codes are:

#### Success codes

- `200 OK` — request has succeeded.
- `201 Created` — new resource has been created.
- `204 No Content` — no content needs to be returned (e.g. when deleting a resource).

#### Client error codes

- `400 Bad Request` — request is malformed in some way (e.g. wrong formatting of JSON, invalid request params, etc).
- `401 Unauthorized` — authentication failed (valid credentials are required).
- `403 Forbidden` — authorization failed (client/user is authenticated but does not have permissions for the requested resource or action).
- `404 Not Found` — resource could not be found.
- `422 Unprocessable Entity` — resource hasn't be saved (e.g. validations on the resource failed) or action can't be performed.

#### Server error codes

- `500 Internal Server Error` — something exploded in your application.

For more information about HTTP status codes, please refer to [this article](https://www.restapitutorial.com/httpstatuscodes.html).

## CORS

For local development of frontend apps, we need to allow cross-origin requests. We use `rack-cors` gem for this purpose. You can find more information about it [here](https://github.com/ynab/rack-cors).

Example configuration:

```ruby
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins 'localhost:3000', %r{\Ahttp://127\.0\.0\.\d{1,3}(:\d+)?\z}, /.*\.themindstudios\.com/

    resource '*',
             headers:     :any,
             methods:     %i[get post put patch delete options head],
             credentials: true,
             expose:      ['Set-Cookie']
  end
end
```
