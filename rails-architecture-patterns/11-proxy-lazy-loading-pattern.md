# Pattern 11: Proxy / Lazy Loading (Association Proxies)

## Overview

The **Proxy** pattern provides a stand-in object that controls access to
another object. Rails uses it for association collections: `blog.posts`
returns a `CollectionProxy`, not an `Array`. The proxy defers the SQL query
until the data is actually needed, transparently delegates collection methods
to the underlying `Relation`, and keeps the association's loading state.

## Linear Walkthrough

### 1. CollectionProxy is the proxy object

```ruby
# activerecord/lib/active_record/associations/collection_proxy.rb:31
class CollectionProxy < Relation
  def initialize(klass, association, **)
    @association = association      # holds state and the unloaded target
    super klass

    extensions = association.extensions
    extend(*extensions) if extensions.any?
  end
```

`CollectionProxy` inherits `Relation`, so every scope method (`.where`,
`.order`, `.limit`) works on it directly without loading records.

### 2. target and load_target — controlled access

```ruby
# activerecord/lib/active_record/associations/collection_proxy.rb:44
def target
  @association.target    # returns loaded records or nil
end

def load_target
  @association.load_target  # fires SQL if not yet loaded; returns records
end

def loaded?
  @association.loaded?      # true only after SQL has fired
end
alias :loaded :loaded?
```

The proxy knows whether the underlying data has been fetched. Calling
`loaded?` never triggers a query.

### 3. SQL deferred until needed

```ruby
blog = Blog.first           # SELECT blogs — one query

blog.posts                  # returns CollectionProxy — no SQL yet
blog.posts.loaded?          # => false

blog.posts.count            # SELECT COUNT(*) — fires SQL, does NOT load records
blog.posts.loaded?          # => false still

blog.posts.records          # SELECT posts WHERE blog_id = … — loads records
blog.posts.loaded?          # => true
```

The proxy delegates `count` to SQL directly without instantiating `Post`
objects. Only methods that require the actual records trigger `load_target`.

### 4. Unknown methods delegated via Relation

Because `CollectionProxy < Relation`, the full scope API is available:

```ruby
blog.posts.where(published: true).order(:created_at).limit(5)
# => still a Relation (proxy), no SQL yet

blog.posts.where(published: true).first
# => SQL fires: SELECT ... WHERE blog_id=? AND published=TRUE LIMIT 1
```

### 5. Lazy loading vs eager loading

Rails detects N+1 patterns and offers `includes` / `preload` / `eager_load`
to bypass the proxy's deferred load and batch-fetch associations:

```ruby
# N+1: fires one SQL per blog
Blog.all.each { |b| b.posts.count }

# Eager load: fires two SQL total
Blog.includes(:posts).each { |b| b.posts.loaded? }  # => true for all
```

The proxy's `loaded?` flag tells Rails whether to use the already-fetched
in-memory records or issue a new query.

## Key Design Decisions

| Decision | Rationale |
|---|---|
| `CollectionProxy < Relation` | Inherits the full scope API at zero cost |
| Separate `loaded?` flag | Clean sentinel; no need to check `nil` vs empty array |
| `count` hits SQL directly | Avoids loading N records to count them |
| `load_target` as the single load gate | One place to add caching, logging, strict-loading checks |

## Summary

Rails' association proxies apply the Proxy pattern to give associations a
lazy, transparent interface. The proxy looks and acts like an array or
relation but defers database access until the last responsible moment. This
is what makes `blog.posts.where(…)` chainable without triggering a query, and
what lets Rails detect and warn about N+1 access patterns.
