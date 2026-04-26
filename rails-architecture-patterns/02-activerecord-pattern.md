# Pattern 02: Active Record (ORM)

## Overview

Martin Fowler's **Active Record** pattern wraps a database row in an object
that carries both data and behaviour for that data. Each model class
corresponds to a table; each instance corresponds to a row. Rails'
`ActiveRecord::Base` is the canonical Ruby implementation of this pattern,
augmented with an internal query-builder (Arel) so that finders compose
cleanly rather than interpolating SQL strings.

## Linear Walkthrough

### 1. One class = one table

By convention a class named `Post` maps to the table `posts`. The class
inherits all CRUD behaviour automatically:

```ruby
# developer-written model (no SQL anywhere)
class Post < ActiveRecord::Base
  belongs_to :author
  validates  :title, presence: true
end
```

### 2. The Base class wires in every capability module

```ruby
# activerecord/lib/active_record/base.rb:282
class Base
  include ActiveModel::API   # validations, naming, conversion, …

  extend  ConnectionHandling  # connects to the database
  extend  Querying            # .where, .find, .order, …
  extend  DynamicMatchers     # .find_by_*

  include Core                # instantiation, equality, inspect
  include Persistence         # save, update, destroy
  include ModelSchema         # column introspection
  include Scoping             # default_scope, named scopes
  include Validations         # validate, errors
  include Callbacks           # before_save, after_commit, …
  include Associations        # has_many, belongs_to, …
  include Transactions        # transaction { … }
  include AttributeMethods    # dynamic attr readers / writers
  include Serialization
  include Encryption::EncryptableRecord
  # …
end
```

### 3. Schema introspection drives attribute generation

When a model is first loaded Rails reads column metadata from the database
and auto-generates attribute accessors. `class_attribute` is used for
class-level defaults so each subclass can override independently:

```ruby
# activerecord/lib/active_record/core.rb:21
class_attribute :logger, instance_writer: false
class_attribute :strict_loading_by_default,  instance_accessor: false, default: false
class_attribute :has_many_inversing,          instance_accessor: false, default: false
class_attribute :belongs_to_required_by_default, instance_accessor: false
```

### 4. Finders return a Relation (lazy query)

`ActiveRecord::Relation` wraps an Arel AST and delays SQL execution until
the result is actually needed. Chained scopes keep building the AST:

```ruby
Post.where(published: true).order(:created_at).limit(10)
# No SQL yet — returns a Relation

Post.where(published: true).each { |p| puts p.title }
# SQL fires here when the Relation is enumerated
```

### 5. Persistence writes back to the row

The `Persistence` module translates object state changes into `INSERT` or
`UPDATE` statements. The full save lifecycle runs through callbacks (see
Pattern 05).

## Key Design Decisions

| Decision | Rationale |
|---|---|
| Active Record (object = row) vs Data Mapper | Trades strict separation for developer ergonomics |
| Lazy `Relation` object | Allows scope composition without N+1 round-trips |
| `class_attribute` for defaults | Subclasses inherit and can override without mutating parent |

## Summary

Rails' Active Record combines the classic pattern (object carries its own
persistence) with a lazy query builder, schema introspection, and a rich
callback lifecycle — all assembled through Ruby's module system so any piece
can be replaced or extended.
