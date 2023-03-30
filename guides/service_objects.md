# Service Objects

## Introduction

*Service Objects* are the place for the application's business logic. They allow you to simplify your models and controllers and allow them to focus on what they are responsible for. A Service should encapsulate a single user task such as registering for a new account, placing an order, publishing post.

## Conventions

* Services go under the `app/services` directory. In case when a service is a part of a module, consider placing it into the directory structure, for example: `app/services/subscriptions/retry_payment_service.rb`, where `Subscriptions` will be module name.
* Service name should have suffix `Service`.
* Service name should contain a verb (e.g. `RetryPaymentService`).
* Service should have only one public method (`#perform`).
* Focus on readability of the `#perform` method, keep it short, simple and clear.
* Put all business logic into private methods with a very clear, self-explanatory and meaningful names.
* Service should return `Result` monad. Use [dry-monads](https://dry-rb.org/gems/dry-monads/1.3/result/) gem.
For better readability condider using [Do notation](https://dry-rb.org/gems/dry-monads/1.3/do-notation/) with `yield`.

## References

* [dry-monads](https://dry-rb.org/gems/dry-monads/1.3/)
* [Railway Oriented Programming In Rails Using Dry-Monads](https://www.honeybadger.io/blog/railway-programming-dry-monads/)
* [Five common issues with services and dry-monads](https://www.davydovanton.com/2020/05/19/five-common-issues-with-services-and-dry-monads/)