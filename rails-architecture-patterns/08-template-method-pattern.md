# Pattern 08: Template Method

## Overview

The **Template Method** pattern defines a skeleton algorithm in a base class,
deferring specific steps to subclasses. Rails uses it extensively:
`ActionController::Metal` defines the dispatch skeleton; `ActiveJob::Base`
defines the job execution skeleton; `ActiveSupport::Cache::Store` defines the
caching algorithm skeleton. Subclasses (or the user's application code) fill
in the abstract steps.

## Linear Walkthrough

### 1. ActionController::Metal — the dispatch skeleton

`Metal` defines the full request–response lifecycle. The abstract step is the
action method itself — it is never defined in Metal; it is always provided by
the concrete controller:

```ruby
# actionpack/lib/action_controller/metal.rb:249
def dispatch(name, request, response)
  set_request!(request)
  set_response!(response)
  process(name)          # calls self.send(name) — the subclass's action
  request.commit_flash
  to_a                   # [status, headers, body]
end
```

The framework calls `dispatch`; the developer defines `index`, `show`, etc.
`process(name)` is the "hook" that runs through the subclass's concrete
implementation.

### 2. ActionController::Base extends via MODULES

`Base` plugs in before/after hooks around the abstract action step via
`process_action` callbacks. The controller lifecycle is:

```
before_action filters
  → process_action (calls the concrete action method)
    → render / redirect (concrete step defined per-action)
after_action filters
```

The developer fills in only the action body:

```ruby
class PostsController < ApplicationController
  def index
    @posts = Post.all    # concrete step
  end
end
```

### 3. ActiveJob — perform_now as skeleton

`ActiveJob::Execution#perform_now` defines the full job execution algorithm.
The concrete step — `perform` — raises `NotImplementedError` in the base,
forcing subclasses to define it:

```ruby
# activejob/lib/active_job/execution.rb:44
def perform_now
  self.executions = (executions || 0) + 1
  deserialize_arguments_if_needed

  _perform_job        # runs callbacks then calls perform(*)
rescue Exception => exception
  handled = rescue_with_handler(exception)
  return handled if handled

  run_after_discard_procs(exception)
  raise
end

def perform(*)
  fail NotImplementedError    # abstract step — subclass must implement
end

private
  def _perform_job
    ActiveSupport::ExecutionContext[:job] = self
    run_callbacks :perform do
      perform(*arguments)     # calls subclass's concrete perform
    end
  end
```

A concrete job provides only the abstract step:

```ruby
class MyJob < ActiveJob::Base
  def perform(user_id)
    User.find(user_id).send_welcome_email
  end
end
```

### 4. Cache::Store — caching algorithm skeleton

`ActiveSupport::Cache::Store` defines the caching algorithm (`fetch`, `read`,
`write`, `delete`). The abstract storage primitives (`read_entry`,
`write_entry`, `delete_entry`) are left to subclasses:

```ruby
# Subclasses implement read_entry, write_entry, delete_entry
# Store.fetch orchestrates:
#   1. read_entry → hit? return cached value
#   2. miss → yield block to compute value
#   3. write_entry → persist computed value
#   4. return computed value
```

`MemoryStore`, `RedisCacheStore`, `FileStore` each implement only the three
storage primitives; the fetch/expire/version logic lives once in `Store`.

### 5. Abstract vs concrete in Rails idiom

Rails uses two signals to mark abstract steps:

| Signal | Meaning |
|---|---|
| `raise NotImplementedError` | Must be overridden (hard enforcement) |
| `# :nodoc: all` on base | Signals the class is infrastructure, not API |
| `abstract!` class macro | Prevents direct instantiation |

```ruby
# actionpack/lib/action_controller/base.rb:209
class Base < Metal
  abstract!    # prevents ApplicationController.new being called directly
```

## Key Design Decisions

| Decision | Rationale |
|---|---|
| `fail NotImplementedError` in `perform` | Explicit contract; fails at runtime with a clear message |
| Callbacks wrap the abstract step | Lets the framework add cross-cutting concerns without modifying the step |
| `abstract!` macro | Documents intent and prevents misuse of base classes |

## Summary

Rails' Template Method pattern appears at every layer: HTTP dispatch, job
execution, caching, and mailer delivery all define a fixed algorithm skeleton
with concrete hooks that application code fills in. The pattern keeps
framework boilerplate out of application code while giving the framework
control over lifecycle concerns like serialization, instrumentation, and
error handling.
