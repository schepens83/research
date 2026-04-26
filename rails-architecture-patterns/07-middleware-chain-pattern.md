# Pattern 07: Chain of Responsibility (Middleware Stack)

## Overview

Rails' HTTP pipeline is built as a **Chain of Responsibility**: an ordered
list of middleware objects, each of which handles a Rack `env` hash (or
delegates to the next link in the chain). The chain is assembled into a
nested callable at boot time using `inject` — the innermost link is the
Rails router/dispatcher, and each outer middleware wraps it.

## Linear Walkthrough

### 1. The Middleware value object

Each entry in the stack is a lightweight `Middleware` wrapper that knows
how to instantiate its class:

```ruby
# actionpack/lib/action_dispatch/middleware/stack.rb:17
class Middleware
  attr_reader :args, :block, :klass

  def initialize(klass, args, block)
    @klass = klass
    @args  = args
    @block = block
  end

  def name; klass.name; end

  def build(app)
    klass.new(app, *args, &block)   # passes the next app into the constructor
  end

  def build_instrumented(app)
    InstrumentationProxy.new(build(app), inspect)
  end
end
```

### 2. InstrumentationProxy — transparent observability

When Notifications has a subscriber for `process_middleware.action_dispatch`,
every middleware call is instrumented with zero changes to the middleware code:

```ruby
# actionpack/lib/action_dispatch/middleware/stack.rb:49
class InstrumentationProxy
  EVENT_NAME = "process_middleware.action_dispatch"

  def initialize(middleware, class_name)
    @middleware = middleware
    @payload = { middleware: class_name }
  end

  def call(env)
    ActiveSupport::Notifications.instrument(EVENT_NAME, @payload) do
      @middleware.call(env)
    end
  end
end
```

### 3. The stack builds itself with inject/reverse

```ruby
# actionpack/lib/action_dispatch/middleware/stack.rb:162
def build(app = nil, &block)
  instrumenting = ActiveSupport::Notifications.notifier.listening?(
    InstrumentationProxy::EVENT_NAME
  )

  middlewares.freeze.reverse.inject(app || block) do |a, e|
    if instrumenting
      e.build_instrumented(a)
    else
      e.build(a)
    end
  end
end
```

`reverse.inject` walks the list from last to first, each middleware
wrapping the result of the previous step. The innermost app becomes the
seed value; each step wraps it in one more middleware layer.

### 4. Mutable stack management API

The stack exposes a rich API for inserting, swapping, and deleting
middleware so applications and engines can customise the chain:

```ruby
# actionpack/lib/action_dispatch/middleware/stack.rb
def use(klass, *args, &block)       # append to end
  middlewares.push(build_middleware(klass, args, block))
end

def insert_before(index, klass, *args, &block)
  index = assert_index(index, :before)
  middlewares.insert(index, build_middleware(klass, args, block))
end

def insert_after(index, *args, &block)
  index = assert_index(index, :after)
  insert(index + 1, *args, &block)
end

def swap(target, *args, &block)
  index = assert_index(target, :before)
  insert(index, *args, &block)
  middlewares.delete_at(index + 1)
end

def delete(target)
  middlewares.reject! { |m| m.name == target.name }
end
```

### 5. Per-action middleware filtering in controllers

`ActionController::MiddlewareStack` extends the base stack to support
`only:` and `except:` filter strategies:

```ruby
# actionpack/lib/action_controller/metal.rb (MiddlewareStack subclass)
def build(action, app = nil, &block)
  action = action.to_s

  middlewares.reverse.inject(app || block) do |a, middleware|
    middleware.valid?(action) ? middleware.build(a) : a
  end
end

# Strategies are plain lambdas:
INCLUDE = ->(list, action) { list.include? action }
EXCLUDE = ->(list, action) { !list.include? action }
NULL    = ->(list, action) { true }
```

`use AuthenticationMiddleware, except: [:index, :show]` attaches the
`EXCLUDE` strategy to that middleware entry.

## Key Design Decisions

| Decision | Rationale |
|---|---|
| `reverse.inject` to build the chain | Elegant functional composition; no recursion, no state machine |
| Separate `Middleware` value object | Defers instantiation until build time; supports inspection |
| `InstrumentationProxy` wrapping | Zero-cost observability: the proxy is only added when a subscriber exists |
| Per-action filtering via lambdas | Avoids conditional branching inside middleware code itself |

## Summary

Rails' middleware stack is a textbook Chain of Responsibility. The chain is
built once at startup, composed functionally via `inject`, and is deeply
inspectable and mutable via the `MiddlewareStack` API. Transparent
instrumentation is added by a proxy decorator at build time.
