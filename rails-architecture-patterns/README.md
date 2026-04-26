# Rails Architecture Patterns

A distillation of 11 software architecture patterns found in the official
[Rails](https://github.com/rails/rails) source code. Each pattern document
contains a high-level overview, a linear walkthrough using code blocks
extracted directly from the Rails source, and a table of key design decisions.

## Pattern Documents

| # | File | Pattern | Rails Location |
|---|------|---------|----------------|
| 01 | [01-mvc-pattern.md](01-mvc-pattern.md) | Model-View-Controller | `actionpack`, `actionview`, `activerecord` |
| 02 | [02-activerecord-pattern.md](02-activerecord-pattern.md) | Active Record (ORM) | `activerecord/lib/active_record/base.rb` |
| 03 | [03-convention-over-configuration.md](03-convention-over-configuration.md) | Convention over Configuration | `activerecord/lib/active_record/model_schema.rb` |
| 04 | [04-concern-mixin-pattern.md](04-concern-mixin-pattern.md) | Concern / Module Mixin | `activesupport/lib/active_support/concern.rb` |
| 05 | [05-callback-observer-pattern.md](05-callback-observer-pattern.md) | Callback / Observer | `activesupport/lib/active_support/callbacks.rb` |
| 06 | [06-strategy-adapter-pattern.md](06-strategy-adapter-pattern.md) | Strategy / Adapter | `activejob/lib/active_job/queue_adapters/`, `activesupport/lib/active_support/cache/` |
| 07 | [07-middleware-chain-pattern.md](07-middleware-chain-pattern.md) | Chain of Responsibility (Middleware) | `actionpack/lib/action_dispatch/middleware/stack.rb` |
| 08 | [08-template-method-pattern.md](08-template-method-pattern.md) | Template Method | `actionpack/lib/action_controller/metal.rb`, `activejob/lib/active_job/execution.rb` |
| 09 | [09-builder-arel-pattern.md](09-builder-arel-pattern.md) | Builder (Arel) | `activerecord/lib/arel/` |
| 10 | [10-pubsub-notifications-pattern.md](10-pubsub-notifications-pattern.md) | Publish/Subscribe | `activesupport/lib/active_support/notifications.rb` |
| 11 | [11-proxy-lazy-loading-pattern.md](11-proxy-lazy-loading-pattern.md) | Proxy / Lazy Loading | `activerecord/lib/active_record/associations/collection_proxy.rb` |

---

## Pattern Summary

### 01 — Model-View-Controller

The foundational request-response architecture. A Rails app exposes a single
Rack endpoint. The router resolves the URL to a controller class and action
name. `ActionController::Metal#dispatch` executes the action; the action
coordinates a model and a view. Three separate inheritance chains
(`Metal → Base`, `ActiveRecord::Base`, `ActionView::Base`) are assembled from
focused modules and united by the router.

**Key file:** `actionpack/lib/action_controller/metal.rb:249`

---

### 02 — Active Record (ORM)

Rows become objects; objects carry both data and persistence behaviour.
`ActiveRecord::Base` is assembled from ~30 focused modules via `include` /
`extend`. A lazy `Relation` object wraps an Arel AST and defers SQL until the
result is enumerated. Column metadata is read from the database at first load
and drives auto-generated attribute accessors.

**Key file:** `activerecord/lib/active_record/base.rb:282`

---

### 03 — Convention over Configuration

Sensible defaults eliminate configuration for the common case. A class named
`Article` automatically maps to table `articles`, primary key `id`, controller
`ArticlesController`, and views in `app/views/articles/`. `class_attribute`
provides class-level inheritable settings that any subclass can override
locally.

**Key file:** `activerecord/lib/active_record/model_schema.rb:261`

---

### 04 — Concern / Module Mixin

`ActiveSupport::Concern` enables fat-class decomposition. A Concern carries
instance methods, class methods, setup code (`included do`), and a list of
dependencies — all resolved correctly when the module is included into a host
class. Every Rails subsystem (`Callbacks`, `Validations`, `Associations`, …)
is a Concern.

**Key file:** `activesupport/lib/active_support/concern.rb:130`

---

### 05 — Callback / Observer

A generic before/around/after chain (`ActiveSupport::Callbacks`) is mapped
onto lifecycle events by each subsystem. ActiveRecord defines 17 lifecycle
hooks from `:after_initialize` through `:after_rollback`. Callbacks can be
methods, blocks, lambdas, or duck-typed objects. Throwing `:abort` halts the
chain; `after_commit` ensures callbacks only fire once the transaction is
durable.

**Key file:** `activerecord/lib/active_record/callbacks.rb:283`

---

### 06 — Strategy / Adapter

Swappable backend implementations behind a fixed abstract interface. Job
adapters (`AsyncAdapter`, `ResqueAdapter`, `SidekiqAdapter`, …) all implement
`enqueue(job)` / `enqueue_at(job, timestamp)`. Cache stores (`MemoryStore`,
`RedisCacheStore`, `MemCacheStore`, …) all implement `read_entry` /
`write_entry` / `delete_entry`. Switching backends is a one-line config
change.

**Key file:** `activejob/lib/active_job/queue_adapters/abstract_adapter.rb`

---

### 07 — Chain of Responsibility (Middleware)

The HTTP pipeline is an ordered list of Rack middleware objects, each
receiving `env` and either handling it or delegating to `next_app.call(env)`.
The chain is built once at startup via `middlewares.reverse.inject(app)`. A
transparent `InstrumentationProxy` wraps each link — but only when a
Notifications subscriber is registered (zero overhead otherwise).

**Key file:** `actionpack/lib/action_dispatch/middleware/stack.rb:162`

---

### 08 — Template Method

A base class defines the algorithm skeleton; subclasses fill in the abstract
steps. `Metal#dispatch` defines how a request is processed; the controller's
action method is the abstract step. `ActiveJob::Execution#perform_now` defines
the job lifecycle; `perform(*)` raises `NotImplementedError` — the subclass
must override it. `Cache::Store#fetch` defines the cache algorithm; subclasses
implement `read_entry` / `write_entry`.

**Key file:** `activejob/lib/active_job/execution.rb:60`

---

### 09 — Builder (Arel)

Arel builds SQL queries as an immutable AST using a fluent Builder API.
`SelectManager` (and `ActiveRecord::Relation`) accumulate nodes
(`.where`, `.order`, `.limit`) without executing SQL. A Visitor
(`ToSql`) renders the AST to database-specific SQL at query time. The same
AST supports multiple SQL dialects via different Visitor subclasses.

**Key file:** `activerecord/lib/arel/select_manager.rb`

---

### 10 — Publish / Subscribe

`ActiveSupport::Notifications` provides a decoupled event bus. Framework code
calls `instrument(event_name, payload) { block }` (publish). Monitoring,
logging, and APM code calls `subscribe(pattern) { |event| … }` (subscribe).
Rails instruments SQL queries, template rendering, controller actions, cache
operations, job execution, and email delivery — all subscribable with zero
changes to framework code.

**Key file:** `activesupport/lib/active_support/notifications.rb:208`

---

### 11 — Proxy / Lazy Loading

Association collections return a `CollectionProxy`, not an array. The proxy
defers SQL until records are actually needed. `count` issues a `SELECT COUNT`
directly. `loaded?` reports whether records are in memory. Unknown methods
delegate to the underlying `Relation`, making the full scope API available
on every association without loading it first.

**Key file:** `activerecord/lib/active_record/associations/collection_proxy.rb:31`

---

## How to Read This

Each pattern document follows the same structure:

1. **Overview** — what the pattern is and where Rails uses it
2. **Linear Walkthrough** — numbered steps tracing the pattern through actual
   Rails source, with code blocks extracted via `grep`/`sed` from the repo
3. **Key Design Decisions** — a table of trade-offs and rationale
4. **Summary** — one paragraph distillation

Code blocks use the file path and line number from the cloned Rails `main`
branch as of 2026-04-26.
