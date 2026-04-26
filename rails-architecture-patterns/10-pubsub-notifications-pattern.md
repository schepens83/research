# Pattern 10: Publish / Subscribe (ActiveSupport::Notifications)

## Overview

`ActiveSupport::Notifications` implements a **Pub/Sub** (Publish/Subscribe)
bus that decouples event producers from event consumers. Any code can
`instrument` an event (publish); any code can `subscribe` to an event
pattern (subscribe). Neither side knows about the other. Rails uses this
throughout the framework for logging, performance monitoring, query tracing,
and cache instrumentation.

## Linear Walkthrough

### 1. Publishing an event — instrument

The producer wraps a block with an event name and an optional payload:

```ruby
# activesupport/lib/active_support/notifications.rb:208
ActiveSupport::Notifications.instrument('render', extra: :information) do
  render plain: 'Foo'
end
```

The block executes first; then all subscribers for `'render'` are notified
with timing and payload information.

### 2. Subscribing to events — subscribe

Subscribers register a pattern (string or regex) and a handler block or
callable object:

```ruby
# activesupport/lib/active_support/notifications.rb:244
ActiveSupport::Notifications.subscribe('render') do |event|
  event.name          # => "render"
  event.duration      # => 10 (milliseconds)
  event.payload       # => { extra: :information }
  event.allocations   # => 1826 (objects)
end

# Low-level 5-argument form
ActiveSupport::Notifications.subscribe('render') do |name, start, finish, id, payload|
  # name    — event name
  # start   — wall-clock start time
  # finish  — wall-clock end time
  # id      — unique instrumenter ID
  # payload — Hash
end
```

### 3. Monotonic subscribers for accurate timing

```ruby
ActiveSupport::Notifications.monotonic_subscribe('render') do |name, start, finish, id, payload|
  # start / finish are monotonic floats — unaffected by system clock changes
end
```

### 4. The instrument / publish API

```ruby
# activesupport/lib/active_support/notifications.rb:200
def publish(name, *args)
  # Fires synchronously to all subscribers in the current thread
end

def instrument(name, payload = {})
  # …
end

def subscribe(pattern = nil, callback = nil, prepend: false, &block)
  # pattern — string (exact match) or regex (prefix/glob)
end

def subscribed(callback, pattern = nil, monotonic: false, &block)
  # Temporarily subscribe for the duration of the block, then unsubscribe
end
```

### 5. Rails instruments itself everywhere

Every major subsystem publishes events. Application code can subscribe to
any of them transparently:

```
sql.active_record              — every SQL query
instantiation.active_record    — model instantiation
render_template.action_view    — template rendering
render_partial.action_view     — partial rendering
process_action.action_controller — controller action
deliver.action_mailer          — email delivery
enqueue.active_job             — job enqueue
perform.active_job             — job perform
cache_read.active_support      — cache read
cache_write.active_support     — cache write
process_middleware.action_dispatch — each middleware call
```

Example — logging every SQL query's duration:

```ruby
ActiveSupport::Notifications.subscribe('sql.active_record') do |event|
  Rails.logger.debug "#{event.payload[:name]} (#{event.duration.round(1)}ms)"
end
```

### 6. Middleware stack integrates transparently

The `InstrumentationProxy` in `MiddlewareStack` publishes
`process_middleware.action_dispatch` only when there is an active subscriber:

```ruby
# actionpack/lib/action_dispatch/middleware/stack.rb:65
def call(env)
  ActiveSupport::Notifications.instrument(EVENT_NAME, @payload) do
    @middleware.call(env)
  end
end
```

The proxy is only inserted into the chain when
`notifier.listening?(EVENT_NAME)` returns true — zero overhead when no
subscriber is registered.

## Key Design Decisions

| Decision | Rationale |
|---|---|
| Event name as dot-namespaced string | `"sql.active_record"` — library.component pattern enables pattern matching |
| `subscribed` helper for scoped subscriptions | Avoids subscription leaks in tests and benchmarks |
| `Event` object vs raw arguments | Provides duration and allocation counts computed once by the framework |
| Conditional proxy insertion | Zero overhead when not monitoring — no allocations for unused instrumentation |

## Summary

`ActiveSupport::Notifications` decouples instrumentation producers from
consumers throughout the entire Rails stack. Framework code publishes events;
APM tools, loggers, and test helpers subscribe to them — with no coupling
between the two sides. The lazy proxy mechanism ensures zero overhead when
nobody is listening.
