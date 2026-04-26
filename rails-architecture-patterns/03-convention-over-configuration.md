# Pattern 03: Convention over Configuration (CoC)

## Overview

**Convention over Configuration** is Rails' foundational design philosophy:
the framework assumes sensible defaults so developers only need to specify
deviations. The result is that a class named `Article` automatically maps to
the table `articles`, the primary key `id`, the controller
`ArticlesController`, and views in `app/views/articles/` — with zero
configuration files.

## Linear Walkthrough

### 1. Table name is inferred from the class name

`ActiveRecord::ModelSchema` computes `table_name` by pluralising and
underscoring the class name. Developers only declare it when they need to
override:

```ruby
# activerecord/lib/active_record/model_schema.rb:261
def table_name
  reset_table_name unless defined?(@table_name)
  @table_name
end

def reset_table_name
  self.table_name = if self == Base
    nil
  elsif abstract_class?
    superclass.table_name
  else
    # Applies table_name_prefix + pluralized underscored class name + suffix
    compute_table_name
  end
end
```

Explicit override is always available:

```ruby
class Mouse < ActiveRecord::Base
  self.table_name = "mice"   # override when the convention doesn't fit
end
```

### 2. Table name prefix / suffix for namespacing

Applications can add a namespace-wide prefix without touching each model:

```ruby
# activerecord/lib/active_record/model_schema.rb (comments)
#
# table_name_prefix= "basecamp_"
#   => "basecamp_projects", "basecamp_people", …
#
# table_name_suffix= "_basecamp"
#   => "projects_basecamp", "people_basecamp", …
```

### 3. Class-level attributes carry sensible defaults

`class_attribute` from `ActiveSupport` creates inheritable class-level
settings with sane defaults that each subclass can override locally:

```ruby
# activerecord/lib/active_record/core.rb:21
class_attribute :logger,                       instance_writer: false
class_attribute :strict_loading_by_default,    instance_accessor: false, default: false
class_attribute :has_many_inversing,           instance_accessor: false, default: false
class_attribute :enumerate_columns_in_select_statements, instance_accessor: false, default: false
class_attribute :run_commit_callbacks_on_first_saved_instances_in_transaction,
                  instance_accessor: false, default: true
```

Setting any attribute on `ApplicationRecord` propagates to all models; a
single model can override without affecting the others.

### 4. Primary key defaults to `id`

```ruby
# activerecord/lib/active_record/model_schema.rb (comments)
#
# primary_key_prefix_type = :table_name
#   => "productid" instead of "id"
#
# primary_key_prefix_type = :table_name_with_underscore
#   => "product_id" instead of "id"
```

No setting means just `id` — the most common case requires no code at all.

### 5. Controller / view path conventions

ActionPack infers view paths from the controller namespace. A controller
`Admin::PostsController` looks for templates in `app/views/admin/posts/` by
default. The partial naming (`_post.html.erb` for `Post` objects) follows the
same class-name → path mapping.

## Key Design Decisions

| Decision | Rationale |
|---|---|
| Pluralised table names | English-natural mapping; overridable when needed |
| `class_attribute` for defaults | Class-level inheritance without global mutation |
| Override > configure | Deviations are opt-in, not the default burden |

## Summary

Convention over Configuration shifts the cost of "obvious" setup from the
developer to the framework. Rails encodes decades of web application
conventions into its defaults; the escape hatches (`self.table_name=`,
`class_attribute` overrides) are always available but rarely needed.
