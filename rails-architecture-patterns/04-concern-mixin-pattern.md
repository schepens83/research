# Pattern 04: Concern / Module Mixin

## Overview

Rails decomposes large classes into focused, reusable **Concerns** —
Ruby modules that use `ActiveSupport::Concern` to cleanly inject both
instance methods and class methods, declare dependencies on other concerns,
and run setup code in the host class's context at include time.

This is Rails' answer to the problem of fat models / fat controllers: slice
cross-cutting behaviour (timestamps, encryption, callbacks, scoping …) into
separate modules and compose them into the host class.

## Linear Walkthrough

### 1. The problem with plain Ruby modules

Without `Concern`, a module that needs to add class methods and run class-level
setup requires boilerplate in every module:

```ruby
# Plain Ruby — verbose and error-prone with nested dependencies
module M
  def self.included(base)
    base.extend ClassMethods
    base.class_eval do
      scope :disabled, -> { where(disabled: true) }
    end
  end

  module ClassMethods
    # …
  end
end
```

### 2. ActiveSupport::Concern simplifies the declaration

```ruby
# activesupport/lib/active_support/concern.rb (from source comments)
module M
  extend ActiveSupport::Concern

  included do
    scope :disabled, -> { where(disabled: true) }
  end

  class_methods do
    def my_class_method
      # …
    end
  end
end
```

`included do … end` runs in the host class's scope at include time.
`class_methods do … end` extends the host class with those methods.

### 3. Dependency resolution

When `Bar` depends on `Foo`, `Concern` tracks and resolves the dependency
chain so the host class doesn't need to know about `Foo`:

```ruby
# activesupport/lib/active_support/concern.rb:80 (comments)
module Foo
  extend ActiveSupport::Concern
  included do
    def self.method_injected_by_foo
      # …
    end
  end
end

module Bar
  extend ActiveSupport::Concern
  include Foo          # Bar declares its own dependency

  included do
    self.method_injected_by_foo
  end
end

class Host
  include Bar          # Foo is included automatically; Host doesn't care
end
```

### 4. The append_features implementation

`Concern` delays the actual include until it knows it is being included into
a real class (not another module building up a dependency list):

```ruby
# activesupport/lib/active_support/concern.rb:130
def append_features(base)
  if base.instance_variable_defined?(:@_dependencies)
    # Being included into another Concern — defer; record dependency
    base.instance_variable_get(:@_dependencies) << self
    false
  else
    # Being included into a real class — flush all deferred deps first
    return false if base < self
    @_dependencies.each { |dep| base.include(dep) }
    super
    base.extend const_get(:ClassMethods) if const_defined?(:ClassMethods)
    base.class_eval(&@_included_block) if instance_variable_defined?(:@_included_block)
  end
end
```

### 5. Usage across Rails itself

Every significant ActiveRecord feature is a Concern:

```ruby
# activerecord/lib/active_record/base.rb:282 (excerpt)
class Base
  include Core
  include Persistence
  include ModelSchema
  include Validations
  include Callbacks
  include Associations
  include Transactions
  include Encryption::EncryptableRecord
  # …
end
```

The `ActiveRecord::Callbacks` concern itself uses `extend ActiveSupport::Concern`:

```ruby
# activerecord/lib/active_record/callbacks.rb:283
module Callbacks
  extend ActiveSupport::Concern

  CALLBACKS = [
    :after_initialize, :after_find, :after_touch, :before_validation, :after_validation,
    :before_save, :around_save, :after_save, :before_create, :around_create,
    :after_create, :before_update, :around_update, :after_update,
    :before_destroy, :around_destroy, :after_destroy, :after_commit, :after_rollback
  ]
  # …
end
```

## Key Design Decisions

| Decision | Rationale |
|---|---|
| Deferred `@_dependencies` accumulation | Avoids double-include and respects module composition order |
| `included do` block runs in host scope | Lets a concern issue class macros (`validates`, `scope`, …) |
| `class_methods` over nested `ClassMethods` module | Reads more naturally; same result |

## Summary

`ActiveSupport::Concern` is the load-bearing mechanism that keeps Rails from
becoming a monolith. It turns Ruby modules into first-class feature slices
that carry their own dependencies, class-level extensions, and setup hooks —
making large classes legible as a curated list of concerns.
