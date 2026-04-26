# Pattern 05: Callback / Observer (Lifecycle Hooks)

## Overview

Rails uses a **Callback** pattern — a specialised form of the Observer
pattern — to let objects hook into their own lifecycle events without
subclassing. `ActiveSupport::Callbacks` provides a generic before/around/after
chain; `ActiveRecord::Callbacks` maps that chain onto the full persistence
lifecycle; `ActiveJob` and `ActionController` expose their own callback sets
using the same mechanism.

## Linear Walkthrough

### 1. ActiveSupport::Callbacks — the generic engine

Any class can declare callback events and run them:

```ruby
# activesupport/lib/active_support/callbacks.rb:37
class Record
  include ActiveSupport::Callbacks
  define_callbacks :save

  def save
    run_callbacks :save do
      puts "- save"
    end
  end
end

class PersonRecord < Record
  set_callback :save, :before, :saving_message
  def saving_message
    puts "saving..."
  end

  set_callback :save, :after do |object|
    puts "saved"
  end
end

person = PersonRecord.new
person.save
# Output:
#   saving...
#   - save
#   saved
```

### 2. Three kinds of callback

```ruby
# activesupport/lib/active_support/callbacks.rb:97
CALLBACK_FILTER_TYPES = [:before, :after, :around].freeze

def run_callbacks(kind, type = nil)
  callbacks = __callbacks[kind.to_sym]

  if callbacks.empty?
    yield if block_given?
  else
    env = Filters::Environment.new(self, false, nil)
    next_sequence = callbacks.compile(type)

    # Common case: no 'around' callbacks defined
    if next_sequence.final?
      next_sequence.invoke_before(env)
      env.value = !env.halted && (!block_given? || yield)
      # …
    end
  end
end
```

Throwing `:abort` inside a `before_` callback halts the chain and prevents
the wrapped action from executing.

### 3. ActiveRecord maps callbacks onto the persistence lifecycle

```ruby
# activerecord/lib/active_record/callbacks.rb:283
module Callbacks
  extend ActiveSupport::Concern

  CALLBACKS = [
    :after_initialize, :after_find, :after_touch,
    :before_validation, :after_validation,
    :before_save,   :around_save,   :after_save,
    :before_create, :around_create, :after_create,
    :before_update, :around_update, :after_update,
    :before_destroy,:around_destroy,:after_destroy,
    :after_commit, :after_rollback
  ]
end
```

The full callback order during `record.save` on a new record is:

```
before_validation
after_validation
before_save
  around_save (wraps the INSERT)
    before_create
      around_create (wraps the core DB write)
      after_create
after_save
after_commit  (fires after the transaction commits)
```

### 4. Callbacks can be methods, procs, lambdas, or objects

```ruby
# activerecord/lib/active_record/callbacks.rb (source comments)

# Method name
before_save :normalize_credit_card_number

# Block
after_create { |record| record.logger.info("Created #{record.id}") }

# Object that responds to before_save(record)
before_save EncryptionWrapper.new("credit_card_number")
```

### 5. Callback set_callback and define_callbacks wiring

```ruby
# activesupport/lib/active_support/callbacks.rb:697
#   set_callback :save, :before, :before_method
#   set_callback :save, :after,  :after_method, if: :condition
#   set_callback :save, :around, ->(r, block) { stuff; result = block.call; stuff }
```

Conditional callbacks (`:if`, `:unless`) are evaluated lazily at runtime, not
at definition time.

## Key Design Decisions

| Decision | Rationale |
|---|---|
| Separate `define_callbacks` / `set_callback` / `run_callbacks` | Clear lifecycle: declare, register, execute |
| `:abort` halt convention | Uniform way to cancel across all callback types |
| `after_commit` separate from `after_save` | Ensures callbacks only fire when the DB transaction is durable |
| Callback objects (duck-typed) | Allows rich, reusable callback logic with its own state |

## Summary

Rails' callback system is a layered, composable lifecycle hook framework.
`ActiveSupport::Callbacks` provides the generic engine; `ActiveRecord`,
`ActiveJob`, and `ActionController` each declare their own event vocabularies
on top of it, giving objects rich introspection points without requiring
subclass overrides.
