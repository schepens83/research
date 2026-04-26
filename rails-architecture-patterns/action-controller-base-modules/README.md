# ActionController::Base MODULES

## Question

How are the modules listed in `ActionController::Base::MODULES` eventually loaded into `ActionController::Base`, given that the same modules are also listed below as `include` calls?

## Short Answer

In Rails 8.1.3, `ActionController::Base::MODULES` itself does not load the modules into `ActionController::Base`.

The modules are mixed into `ActionController::Base` by the explicit `include` calls immediately below the constant in `actionpack/lib/action_controller/base.rb`.

```ruby
MODULES = [
  AbstractController::Rendering,
  AbstractController::Translation,
  # ...
  ParamsWrapper
]

include AbstractController::Rendering
include AbstractController::Translation
# ...
include ParamsWrapper
```

The constant is an ordered manifest of the default modules. Its main public use is `ActionController::Base.without_modules`, which lets developers build a custom `ActionController::Metal` subclass with most of the `Base` stack but with selected modules omitted.

## Loading Sequence

When Rails loads Action Controller:

1. `action_controller.rb` defines the `ActionController` namespace and sets up `ActiveSupport::Autoload` entries for `Base`, `Metal`, and the modules under `action_controller/metal`.
2. Referencing `ActionController::Base` autoloads `action_controller/base.rb`.
3. `ActionController::Base` subclasses `ActionController::Metal`.
4. Ruby evaluates the `MODULES` constant. Referenced constants such as `Helpers`, `Rendering`, and `ParamsWrapper` are resolved, using Rails autoloads when needed.
5. Ruby evaluates the explicit `include` calls below the constant. These are what actually add the module methods and hooks to `ActionController::Base`.
6. Rails runs load hooks:

```ruby
ActiveSupport.run_load_hooks(:action_controller_base, self)
ActiveSupport.run_load_hooks(:action_controller, self)
```

## Why Keep Both?

The duplication is intentional in the current source shape:

- The `include` calls define the actual behavior of `ActionController::Base`.
- `MODULES` exposes the same ordered set for reuse by custom controller bases.
- `without_modules(*modules)` returns `MODULES - modules`, supporting patterns like:

```ruby
class MyBaseController < ActionController::Metal
  ActionController::Base.without_modules(:ParamsWrapper, :Streaming).each do |mod|
    include mod
  end
end
```

This lets an application start from the same default stack as `ActionController::Base` while opting out of specific features.

## Include Order

Ruby's `include` order matters. If a class includes `A` and then includes `B`, method lookup checks `B` before `A`:

```ruby
class Example
  include A
  include B
end

Example.ancestors # => [Example, B, A, ...]
```

That explains the comments near the bottom of `ActionController::Base`: callbacks, rescue handling, instrumentation, and params wrapping are included late so they land early enough in the ancestor chain to wrap or observe more of the behavior stack.

## Contrast With ActionController::API

`ActionController::API` uses the pattern one might expect:

```ruby
MODULES.each do |mod|
  include mod
end
```

`ActionController::Base` does not currently use that loop. It keeps explicit `include` calls instead.

## Takeaway

For `ActionController::Base`, read `MODULES` as the reusable recipe and the `include` block as the actual assembly step. The modules are loaded by constant resolution/autoloading, but they become part of `Base` only when Ruby executes the explicit `include` statements.
