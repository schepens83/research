# Pattern 09: Builder (Arel Query Builder)

## Overview

The **Builder** pattern separates the construction of a complex object from
its representation. Rails uses it in **Arel** — the SQL abstract syntax tree
(AST) library that backs `ActiveRecord::Relation`. Arel builds an immutable
tree of AST nodes step-by-step via a fluent interface; the tree is only
compiled to SQL when actually needed.

## Linear Walkthrough

### 1. The Arel directory — managers, nodes, visitors

```
activerecord/lib/arel/
  select_manager.rb      ← builds SELECT queries
  insert_manager.rb      ← builds INSERT queries
  update_manager.rb      ← builds UPDATE queries
  delete_manager.rb      ← builds DELETE queries
  nodes/                 ← AST node types (And, Or, In, Eq, Join, …)
  visitors/              ← SQL renderers (ToSql, Dot, …)
  table.rb               ← entry point for building table references
```

Manager classes (the Builders) assemble the AST. Visitor classes traverse
the AST to render SQL.

### 2. SelectManager — fluent builder API

```ruby
# activerecord/lib/arel/select_manager.rb:1
class SelectManager < Arel::TreeManager
  include Arel::Crud

  def initialize(table = nil)
    super
    @ast = Nodes::SelectStatement.new(table)  # root of the AST
    @ctx = @ast.cores.last
  end

  def skip(amount)   # OFFSET
    if amount
      @ast.offset = Nodes::Offset.new(amount)
    else
      @ast.offset = nil
    end
    self               # returns self for chaining
  end
  alias :offset= :skip

  def exists
    Arel::Nodes::Exists.new @ast
  end

  def as(other)
    create_table_alias grouping(@ast), Nodes::SqlLiteral.new(other, retryable: true)
  end
end
```

Each method mutates the AST in-place and returns `self`, enabling
method chaining.

### 3. ActiveRecord::Relation wraps the Arel manager

`ActiveRecord::Relation` is the public-facing Builder that most Rails
developers interact with. It delegates to an Arel `SelectManager` and defers
SQL generation:

```ruby
# Developer code — chaining builds the AST progressively
Post
  .where(published: true)   # adds a WHERE node
  .order(:created_at)       # adds an ORDER node
  .limit(10)                # adds a LIMIT node
  .offset(20)               # adds an OFFSET node
# => no SQL yet; returns a Relation wrapping an Arel SelectStatement

Post.where(published: true).to_sql
# => "SELECT \"posts\".* FROM \"posts\" WHERE \"posts\".\"published\" = TRUE"
```

### 4. AST nodes are plain value objects

```
activerecord/lib/arel/nodes/
  and.rb         Arel::Nodes::And
  or.rb          Arel::Nodes::Or
  equality.rb    Arel::Nodes::Equality  (col = val)
  in.rb          Arel::Nodes::In        (col IN (...))
  join_source.rb Arel::Nodes::JoinSource
  select_statement.rb   root node
  …
```

Nodes hold references to child nodes, not SQL strings. This lets the
Visitor render the same AST for MySQL, PostgreSQL, or SQLite3 by
dispatching on node type.

### 5. Visitor renders the AST to SQL

```ruby
# The ToSql visitor walks the AST and emits SQL
# (activerecord/lib/arel/visitors/to_sql.rb — condensed illustration)
def visit_Arel_Nodes_SelectStatement(o, collector)
  visit_Arel_Nodes_SelectCore(o.cores.last, collector)
  visit_Arel_Nodes_Limit(o.limit, collector)    if o.limit
  visit_Arel_Nodes_Offset(o.offset, collector)  if o.offset
  collector
end
```

Different visitor subclasses generate vendor-specific SQL without touching
the builder or the AST nodes.

## Key Design Decisions

| Decision | Rationale |
|---|---|
| Immutable-ish AST nodes | Safe to cache; Relation dup creates a fresh manager copy |
| Fluent `return self` API | Enables scope chaining: `.where.order.limit` reads naturally |
| Separate Visitor for SQL generation | One AST; multiple SQL dialects without branching in nodes |
| `Relation` as the public API over `SelectManager` | Shields app code from the low-level Arel API |

## Summary

Arel is a textbook Builder + Composite + Visitor triad. The Builder
(`SelectManager` / `Relation`) assembles an AST incrementally; the Composite
(AST nodes) represents the complete query structure; the Visitor
(`ToSql`) traverses the tree to produce database-specific SQL. Deferring SQL
generation to query execution time makes scope composition safe and efficient.
