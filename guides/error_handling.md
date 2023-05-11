# Error Handling

## Handle Failure

We use Railway programming approach in our **service objects**. It means that we don't use exceptions for control flow. For example, if we have a service object that creates a user, we don't want to use exceptions to handle cases when user already exists. Instead, we want to return a [Result](https://dry-rb.org/gems/dry-monads/1.3/result/) object that will contain information about the result of the operation.

After that we can handle this result in the controller and decide what to do next:
1. With pattern matching (Ruby 2.7+):
```ruby
case result
in Success(user)
  render 'show', locals: { user: user }
in Failure[:validation_failed, error]
  render_api_error(error)
end
```
2. Create universal [ResultMatcher](https://dry-rb.org/gems/dry-matcher/1.0/result-matcher/) in ApiController that will handle all possible results:
```ruby
class ApiController < ApplicationController
  def render_result(result)
    Dry::Matcher::ResultMatcher.(value) do |m|
      m.success do |value|
        yield(value) if block_given?
      end

      m.failure do |value|
        render_api_error(value)
      end
    end
  end
end
```

If you have 3rd-party library that will raise exceptions, you can wrap it with [Try](https://dry-rb.org/gems/dry-monads/1.3/try/):
```ruby
Try(ArgumentError) { JSON.parse('invalid json') }
# => Try::Error(ArgumentError: unexpected token at 'invalid json')
```
and then transform it to [Result](https://dry-rb.org/gems/dry-monads/1.3/try/#code-to_result-code-and-code-to_maybe-code).

## rescue_from

Define global error handler in `ApiController` to render json response for all exceptions and pass extra data to error tracking service:
```ruby
class ApiController < ApplicationController
  rescue_from StandardError, with: :handle_exception

  def handle_exception(exception)
    logger.fatal "API MESSAGE: #{exception.message}"
    logger.fatal "API BACKTRACE: \n\t #{exception.backtrace.grep_v(%r{/gems/}).join("\n\t")}"

    Sentry.capture_exception(exception, extra: { user_id: current_user&.id })

    render json: format_error_message(details: exception.message, title: 'Internal server error'),
             status: :internal_server_error
  end
end
```

* [Ruby Exceptions](http://rubylearning.com/satishtalim/ruby_exceptions.html)
* [Rescue StandardError, Not Exception](https://thoughtbot.com/blog/rescue-standarderror-not-exception)

## Standardized error response

While most REST APIs follow similar conventions, specifics usually vary, including the names of fields and the information included in the response body. These differences make it difficult for libraries and frameworks to handle errors uniformly.

In an effort to standardize REST API error handling, **the IETF devised [RFC 7807](https://tools.ietf.org/html/rfc7807), which creates a generalized error-handling schema**.

This schema is composed of five parts:

* *type* – a URI identifier that categorizes the error, can be omitted if the error is not specific to a resource
* *title* – a brief, human-readable message about the error
* *status* – the HTTP response code (optional)
* *detail* – a human-readable explanation of the error
* *instance* – a URI that identifies the specific occurrence of the error

Instead of using our custom error response body, we can convert our body:

```json
{
    "type": "/errors/incorrect-user-pass",
    "title": "Incorrect username or password.",
    "status": 401,
    "detail": "Authentication failed due to incorrect username or password.",
    "instance": "/users/login"
}
```

Note that the *type* field categorizes the type of error, while *instance* identifies a specific occurrence of the error in a similar fashion to classes and objects, respectively.

Adhering to RFC 7807 is optional, but it is advantageous if uniformity is desired.

## Error Monitoring Services

- [sentry.io](https://docs.sentry.io/platforms/ruby/guides/rails/)
- [datadog](https://www.datadoghq.com/product/error-tracking/)