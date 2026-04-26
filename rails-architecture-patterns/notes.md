# Rails Architecture Patterns - Research Notes

## Date: 2026-04-26

## Plan
Clone official Rails repo and extract at least 8 architecture patterns with code examples.

## Rails components
- actioncable, actionmailbox, actionmailer, actionpack, actiontext, actionview
- activejob, activemodel, activerecord, activestorage, activesupport
- railties

## Patterns to investigate
1. MVC (Model-View-Controller)
2. ActiveRecord Pattern (ORM)
3. Convention over Configuration
4. Concern/Module Mixin (Cross-cutting concerns)
5. Observer/Callback Pattern (Lifecycle hooks)
6. Strategy Pattern (Adapters/Backends)
7. Chain of Responsibility (Middleware stack)
8. Template Method Pattern
9. Builder Pattern (Arel query builder)
10. Pub/Sub (ActiveSupport::Notifications)

## Completed patterns (2026-04-26)

1. MVC - actionpack/metal.rb, base.rb, activerecord/base.rb
2. Active Record ORM - activerecord/base.rb, core.rb
3. Convention over Configuration - model_schema.rb table_name inference, class_attribute defaults
4. Concern/Module Mixin - activesupport/concern.rb append_features
5. Callback/Observer - activesupport/callbacks.rb, activerecord/callbacks.rb CALLBACKS list
6. Strategy/Adapter - activejob/queue_adapters/abstract_adapter.rb, activesupport/cache/ stores
7. Chain of Responsibility (Middleware) - action_dispatch/middleware/stack.rb build inject
8. Template Method - action_controller/metal.rb dispatch, activejob/execution.rb perform_now
9. Builder (Arel) - arel/select_manager.rb, nodes/, visitors/
10. Pub/Sub Notifications - activesupport/notifications.rb instrument/subscribe
11. Proxy/Lazy Loading - activerecord/associations/collection_proxy.rb

## Key findings
- Rails decomposes almost everything into ActiveSupport::Concern modules
- Strategy pattern is used for ALL swappable infrastructure (cache, jobs, sessions, mailer delivery)
- The middleware chain uses functional composition (reverse.inject) - elegant
- Arel is a full Builder+Composite+Visitor triad hidden behind ActiveRecord::Relation
- Notifications bus is zero-overhead when no subscriber is registered (conditional proxy)
- Convention over Configuration is implemented via class_attribute inheritance, not YAML
