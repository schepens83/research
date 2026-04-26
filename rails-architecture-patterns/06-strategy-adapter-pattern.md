# Pattern 06: Strategy / Adapter

## Overview

Rails uses the **Strategy** pattern extensively to swap backend
implementations without changing calling code. An abstract interface is
declared; concrete adapters implement it. The rest of the framework only
talks to the interface. This appears in cache stores, job queue adapters,
session stores, mailer delivery methods, and more.

## Linear Walkthrough

### 1. The AbstractAdapter contract (ActiveJob)

`ActiveJob::QueueAdapters::AbstractAdapter` is the minimal interface every
job backend must satisfy:

```ruby
# activejob/lib/active_job/queue_adapters/abstract_adapter.rb
module ActiveJob
  module QueueAdapters
    class AbstractAdapter
      attr_accessor :stopping

      def enqueue(job)
        raise NotImplementedError
      end

      def enqueue_at(job, timestamp)
        raise NotImplementedError
      end

      def stopping?
        !!@stopping
      end
    end
  end
end
```

`enqueue` and `enqueue_at` are the only two methods the framework calls. Any
backend (Sidekiq, Resque, Delayed Job, Sneakers …) just implements those.

### 2. Concrete adapters available out of the box

```
activejob/lib/active_job/queue_adapters/
  abstract_adapter.rb
  async_adapter.rb       ← in-process thread pool (development)
  inline_adapter.rb      ← execute immediately, no queue
  resque_adapter.rb
  sidekiq_adapter.rb     (in activejob gem)
  backburner_adapter.rb
  delayed_job_adapter.rb
  queue_classic_adapter.rb
  sneakers_adapter.rb
  test_adapter.rb        ← captures jobs for assertions
```

Switching backends is a one-line config change:

```ruby
config.active_job.queue_adapter = :sidekiq
```

### 3. Cache store — same pattern

`ActiveSupport::Cache::Store` is the abstract base; each store inherits from
it and overrides the storage primitives:

```ruby
# activesupport/lib/active_support/cache/memory_store.rb:30
class MemoryStore < Store
  prepend Strategy::LocalCache

  # Stores entries in a plain Hash in process memory
  # Implements read_entry, write_entry, delete_entry
end
```

```ruby
# activesupport/lib/active_support/cache/redis_cache_store.rb:30
class RedisCacheStore < Store
  # Stores entries via Redis
  # Same read_entry / write_entry / delete_entry interface
end
```

Available stores:

```
activesupport/lib/active_support/cache/
  memory_store.rb
  mem_cache_store.rb
  redis_cache_store.rb
  file_store.rb
  null_store.rb
```

Changing cache backend:

```ruby
config.cache_store = :redis_cache_store, { url: ENV["REDIS_URL"] }
```

### 4. Strategy selection at runtime

ActiveJob resolves the adapter class from a symbol at configuration time:

```ruby
# activejob (conceptual — adapter loading)
config.active_job.queue_adapter = :resque
# => resolves to ActiveJob::QueueAdapters::ResqueAdapter.new
```

The rest of the job infrastructure calls only `adapter.enqueue(job)` —
completely unaware of which concrete adapter is in play.

### 5. ActionMailer delivery method — same pattern

```
actionmailer/lib/action_mailer/
  delivery_methods.rb      ← registers strategies by name
  mail_delivery_job.rb

  # Deliveries:
  smtp_delivery.rb
  sendmail_delivery.rb
  file_delivery.rb
  test_mailer.rb
```

`delivery_method :smtp` vs `delivery_method :test` swaps the strategy.

## Key Design Decisions

| Decision | Rationale |
|---|---|
| `raise NotImplementedError` in abstract base | Documents required interface; fails loudly if partial implementation |
| Symbol-to-class resolution at config time | Keeps config readable; resolves once, not per-call |
| `prepend Strategy::LocalCache` in MemoryStore | Decorates the store with an in-request cache layer transparently |

## Summary

Rails' Strategy pattern lets applications treat caching, job queuing, email
delivery, and session storage as infrastructure choices rather than code
choices. Swapping backends requires only a config change; no business logic
needs to know which strategy is active.
