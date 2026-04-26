# Pattern 01: Model-View-Controller (MVC)

## Overview

MVC is Rails' central organizing principle. Every web request is routed to a
**Controller** action, which coordinates a **Model** (data + business logic) and
a **View** (template rendering) and then returns a response. The three roles
are kept in separate layers so that each can evolve independently.

## Linear Walkthrough

### 1. The request enters via Rack → Router → Controller

A Rails app exposes a single Rack endpoint. The router maps the URL to a
controller class and action name. `ActionController::Metal#dispatch` wires the
two together:

```ruby
# actionpack/lib/action_controller/metal.rb:249
def dispatch(name, request, response)
  set_request!(request)
  set_response!(response)
  process(name)          # calls the named action method
  request.commit_flash
  to_a                   # returns [status, headers, body] to Rack
end
```

### 2. ActionController::Base assembles behaviour via modules

`ActionController::Base` inherits from `ActionController::Metal` (the bare
Rack adapter) and then includes a curated list of capability modules:

```ruby
# actionpack/lib/action_controller/base.rb:208
class Base < Metal
  abstract!

  MODULES = [
    AbstractController::Rendering,
    AbstractController::Translation,
    AbstractController::AssetPaths,
    Helpers,
    UrlFor,
    Redirecting,
    ActionView::Layouts,
    Rendering,
    Renderers::All,
    ConditionalGet,
    EtagWithTemplateDigest,
    EtagWithFlash,
    Caching,
    MimeResponds,
    ImplicitRender,
    StrongParameters,
    ParameterEncoding,
    Cookies,
    Flash,
    FormBuilder,
    RequestForgeryProtection,
    ContentSecurityPolicy,
    PermissionsPolicy,
    RateLimiting,
    AllowBrowser,
    Streaming,
    DataStreaming,
    HttpAuthentication::Basic::ControllerMethods,
    HttpAuthentication::Digest::ControllerMethods,
    # ...
  ]
end
```

The `without_modules` class method lets application developers strip out any
of these defaults when creating a minimal controller.

### 3. The Model layer: ActiveRecord::Base

`ActiveRecord::Base` similarly assembles all model capabilities through
`include`/`extend` of focused modules:

```ruby
# activerecord/lib/active_record/base.rb:282
class Base
  include ActiveModel::API

  extend  ConnectionHandling
  extend  Querying
  extend  DynamicMatchers
  extend  Enum

  include Core
  include Persistence
  include ModelSchema
  include Inheritance
  include Scoping
  include Validations
  include Callbacks
  include Timestamp
  include Associations
  include Transactions
  # ... many more
end

ActiveSupport.run_load_hooks(:active_record, Base)
```

### 4. The View layer: ActionView::Base

```ruby
# actionview/lib/action_view/base.rb:158
class Base
  # Rendering templates, partials, layouts, helpers
end
```

The controller renders a view by calling `render`, which instantiates
`ActionView::Base`, assigns instance variables, and executes the template.

## Key Design Decisions

| Decision | Rationale |
|---|---|
| `Metal` as the thin Rack layer | Allows pure-Rack apps with zero controller overhead |
| Capability modules instead of deep inheritance | Easy to cherry-pick features; testable in isolation |
| `run_load_hooks` at the end of Base | Third-party gems and the app can hook in after the core loads |

## Summary

Rails MVC is not just three folders — it is three distinct inheritance chains
(`Metal → Base`, `Relation → Base`, `ActionView::Base`) each assembled from
dozens of single-purpose modules, unified by the router and the Rack contract.
