---
name: infinispan-listeners
description: Use when user needs help with Infinispan listeners and events - cache listeners, cache manager listeners, Hot Rod client listeners, CDI events, clustered listeners, event filtering.
---

# Infinispan Listeners and Events Guide

Help users implement listeners and event notifications in Infinispan 16.x for both embedded and remote (Hot Rod) deployments.

## Workflow

1. Determine whether the user needs embedded (cache/cache manager) or remote (Hot Rod client) listeners
2. Ask about their event processing requirements (sync vs async, filtering needs)
3. Provide the appropriate listener implementation with code examples
4. Warn about performance implications and common mistakes

## Listener Types Overview

| Listener Type | Scope | Use When |
|--------------|-------|----------|
| **Cache Listener** | Embedded, per-cache | Reacting to entry changes on the node where they occur |
| **Clustered Listener** | Embedded, cluster-wide | Receiving all cache events on a single node |
| **Cache Manager Listener** | Embedded, manager-level | Monitoring cache lifecycle and cluster membership |
| **Hot Rod Client Listener** | Remote, per-cache | Client-side notifications of server-side cache changes |
| **CDI Events** | Embedded (CDI container) | Observing cache events in Jakarta EE / CDI applications |

---

## Cache Listeners (Embedded)

Cache-level events occur on a per-cache basis. In distributed caches, events are raised only on the owners of the affected data.

### Available Cache Event Annotations

| Annotation | Event Class | Triggered When |
|-----------|-------------|----------------|
| `@CacheEntryCreated` | `CacheEntryCreatedEvent` | A new entry is added |
| `@CacheEntryModified` | `CacheEntryModifiedEvent` | An existing entry is updated |
| `@CacheEntryRemoved` | `CacheEntryRemovedEvent` | An entry is removed |
| `@CacheEntryExpired` | `CacheEntryExpiredEvent` | An entry expires |

### Basic Cache Listener

```java
import org.infinispan.notifications.Listener;
import org.infinispan.notifications.cachelistener.annotation.*;
import org.infinispan.notifications.cachelistener.event.*;
import java.util.concurrent.CompletionStage;

@Listener
public class MyCacheListener {

   @CacheEntryCreated
   public CompletionStage<Void> onCreated(CacheEntryCreatedEvent<String, String> event) {
      System.out.printf("Entry created: key=%s, value=%s%n", event.getKey(), event.getValue());
      return null;
   }

   @CacheEntryModified
   public CompletionStage<Void> onModified(CacheEntryModifiedEvent<String, String> event) {
      System.out.printf("Entry modified: key=%s, newValue=%s%n", event.getKey(), event.getValue());
      return null;
   }

   @CacheEntryRemoved
   public CompletionStage<Void> onRemoved(CacheEntryRemovedEvent<String, String> event) {
      System.out.printf("Entry removed: key=%s%n", event.getKey());
      return null;
   }

   @CacheEntryExpired
   public CompletionStage<Void> onExpired(CacheEntryExpiredEvent<String, String> event) {
      System.out.printf("Entry expired: key=%s%n", event.getKey());
      return null;
   }
}
```

### Registering a Cache Listener

```java
Cache<String, String> cache = cacheManager.getCache("myCache");
cache.addListener(new MyCacheListener());
```

---

## Clustered Listeners

Clustered listeners receive events from all nodes in the cluster on a single node. Annotate the listener with `@Listener(clustered = true)`.

### Limitations

- Clustered listeners can only listen to `@CacheEntryCreated`, `@CacheEntryModified`, `@CacheEntryRemoved`, and `@CacheEntryExpired` events.
- Only the post event is sent; the pre event is ignored.

### Example

```java
@Listener(clustered = true)
public class MyClusteredListener {

   @CacheEntryCreated
   public void onCreated(CacheEntryCreatedEvent<String, String> event) {
      // This receives events from ALL nodes in the cluster
      System.out.printf("Cluster-wide entry created: key=%s%n", event.getKey());
   }

   @CacheEntryModified
   public void onModified(CacheEntryModifiedEvent<String, String> event) {
      System.out.printf("Cluster-wide entry modified: key=%s%n", event.getKey());
   }
}
```

### Initial State Events

When `includeCurrentState = true` on a clustered listener, Infinispan iterates the existing cache contents and fires a `@CacheEntryCreated` event for each entry upon registration. This is useful for building local data structures from existing data.

```java
@Listener(clustered = true, includeCurrentState = true)
public class StateAwareListener {

   @CacheEntryCreated
   public void onCreated(CacheEntryCreatedEvent<String, String> event) {
      // Called for existing entries and new entries
   }
}
```

**Note:** `includeCurrentState` only works for clustered listeners.

---

## Cache Manager Listeners

Cache Manager events are cluster-wide and involve events affecting all caches managed by a single CacheManager.

### Available Cache Manager Event Annotations

| Annotation | Event Class | Triggered When |
|-----------|-------------|----------------|
| `@CacheStarted` | `CacheStartedEvent` | A cache starts |
| `@CacheStopped` | `CacheStoppedEvent` | A cache stops |
| `@ViewChanged` | `ViewChangedEvent` | Cluster membership changes (node joins/leaves) |
| `@Merged` | `MergeEvent` | Cluster partitions merge back together |

### Example

```java
import org.infinispan.notifications.Listener;
import org.infinispan.notifications.cachemanagerlistener.annotation.*;
import org.infinispan.notifications.cachemanagerlistener.event.*;

@Listener
public class MyCacheManagerListener {

   @CacheStarted
   public void onCacheStarted(CacheStartedEvent event) {
      System.out.printf("Cache started: %s%n", event.getCacheName());
   }

   @CacheStopped
   public void onCacheStopped(CacheStoppedEvent event) {
      System.out.printf("Cache stopped: %s%n", event.getCacheName());
   }

   @ViewChanged
   public void onViewChanged(ViewChangedEvent event) {
      System.out.printf("Cluster view changed: %d members%n", event.getNewMembers().size());
   }

   @Merged
   public void onMerge(MergeEvent event) {
      System.out.println("Cluster partitions merged");
   }
}
```

### Registering a Cache Manager Listener

```java
EmbeddedCacheManager cacheManager = ...;
cacheManager.addListener(new MyCacheManagerListener());
```

---

## Synchronicity: Sync vs Async Listeners

Infinispan supports three listener execution models. Choosing the right one is critical for performance.

| Model | Annotation | Return Type | Behavior |
|-------|-----------|-------------|----------|
| **Blocking synchronous** | `@Listener` | `void` | Blocks the calling thread until the listener method completes. Delays the cache operation. |
| **Non-blocking synchronous** | `@Listener` | `CompletionStage<Void>` | Does not block the calling thread. Cache operation waits for the CompletionStage to complete. |
| **Asynchronous** | `@Listener(sync = false)` | `void` | Cache operation proceeds immediately. Listener runs on the notification thread pool. |

### Blocking Synchronous Listener

```java
@Listener
public class MySyncListener {
   @CacheEntryCreated
   void listen(CacheEntryCreatedEvent event) {
      // Blocks the thread performing the cache operation
   }
}
```

### Non-Blocking Synchronous Listener (Preferred)

```java
@Listener
public class MyNonBlockingListener {
   @CacheEntryCreated
   CompletionStage<Void> listen(CacheEntryCreatedEvent event) {
      // Returns immediately, operation waits for CompletionStage
      return CompletableFuture.runAsync(() -> {
         // Process event without blocking the cache operation thread
      });
   }
}
```

### Asynchronous Listener

```java
@Listener(sync = false)
public class MyAsyncListener {
   @CacheEntryCreated
   void listen(CacheEntryCreatedEvent event) {
      // Runs on the notification thread pool
      // Cache operation does NOT wait for completion
   }
}
```

### Tuning the Async Thread Pool

Configure the async notification thread pool in the Infinispan configuration:

```xml
<cache-container>
  <threads>
    <listener-executor/>
  </threads>
</cache-container>
```

---

## Event Filtering and Conversion (Embedded)

### Key Filtering

Filter events by key using `KeyFilter`:

```java
import org.infinispan.filter.KeyFilter;

public class SpecificKeyFilter implements KeyFilter<String> {
   private final String keyToAccept;

   public SpecificKeyFilter(String keyToAccept) {
      this.keyToAccept = keyToAccept;
   }

   @Override
   public boolean accept(String key) {
      return keyToAccept.equals(key);
   }
}

// Register with filter
cache.addListener(listener, new SpecificKeyFilter("importantKey"));
```

### Cache Event Filtering

Use `CacheEventFilter` for filtering on keys, old/new values, metadata, and event type:

```java
import org.infinispan.notifications.cachelistener.filter.CacheEventFilter;
import org.infinispan.filter.NamedFactory;

class MyCacheEventFilter implements CacheEventFilter<String, String> {
   @Override
   public boolean accept(String key, String oldValue, Metadata oldMetadata,
                          String newValue, Metadata newMetadata, EventType eventType) {
      // Only accept events where the new value is non-null
      return newValue != null && newValue.startsWith("important");
   }
}
```

### Cache Event Conversion

Use `CacheEventConverter` to transform event data before delivery:

```java
import org.infinispan.notifications.cachelistener.filter.CacheEventConverter;

class SummaryConverter implements CacheEventConverter<String, String, String> {
   @Override
   public String convert(String key, String oldValue, Metadata oldMetadata,
                          String newValue, Metadata newMetadata, EventType eventType) {
      return key + "=" + newValue; // Send a summary instead of full event
   }
}
```

**Performance tip:** Filters and converters are especially beneficial with clustered listeners. The filtering and conversion happens on the node where the event originates, avoiding unnecessary network traffic.

---

## Hot Rod Client Listeners (Remote)

Hot Rod clients can register listeners for cache-entry level events on the server.

### Available Client Event Annotations

| Annotation | Event Class | Provides |
|-----------|-------------|----------|
| `@ClientCacheEntryCreated` | `ClientCacheEntryCreatedEvent` | Key, version |
| `@ClientCacheEntryModified` | `ClientCacheEntryModifiedEvent` | Key, version |
| `@ClientCacheEntryRemoved` | `ClientCacheEntryRemovedEvent` | Key |
| `@ClientCacheFailover` | `ClientCacheFailoverEvent` | Failover notification |

### Basic Hot Rod Client Listener

```java
import org.infinispan.client.hotrod.annotation.*;
import org.infinispan.client.hotrod.event.*;

@ClientListener
public class MyClientListener {

   @ClientCacheEntryCreated
   public void handleCreated(ClientCacheEntryCreatedEvent<String> event) {
      System.out.printf("Created: key=%s, version=%d%n", event.getKey(), event.getVersion());
   }

   @ClientCacheEntryModified
   public void handleModified(ClientCacheEntryModifiedEvent<String> event) {
      System.out.printf("Modified: key=%s, version=%d%n", event.getKey(), event.getVersion());
   }

   @ClientCacheEntryRemoved
   public void handleRemoved(ClientCacheEntryRemovedEvent<String> event) {
      System.out.printf("Removed: key=%s%n", event.getKey());
   }
}
```

### Registering and Removing Client Listeners

```java
RemoteCache<String, String> cache = remoteCacheManager.getCache("myCache");

// Register
MyClientListener listener = new MyClientListener();
cache.addClientListener(listener);

// Remove when no longer needed
cache.removeClientListener(listener);
```

### Skipping Notifications

Use `SKIP_LISTENER_NOTIFICATION` flag to perform operations without generating events:

```java
remoteCache.withFlags(Flag.SKIP_LISTENER_NOTIFICATION).put("key", "value");
```

### Listener Failover Handling

When the server node hosting a client listener fails, the Hot Rod client transparently fails over to another node. Use `@ClientCacheFailover` to handle this:

```java
@ClientListener(includeCurrentState = true)
public class ResilientListener {

   @ClientCacheEntryCreated
   public void handleCreated(ClientCacheEntryCreatedEvent<String> event) {
      // Process event
   }

   @ClientCacheFailover
   public void handleFailover(ClientCacheFailoverEvent event) {
      // Clear local state - events may have been missed
      localCache.clear();
      // With includeCurrentState=true, CacheEntryCreated events
      // will be replayed for all existing entries
   }
}
```

### Duplicate Events

In non-transactional caches, duplicate events can occur when the primary owner fails during a write. Use `isCommandRetried()` to detect this:

```java
@Listener
public class RetryAwareListener {
   @CacheEntryModified
   public void onModified(CacheEntryModifiedEvent<String, String> event) {
      if (event.isCommandRetried()) {
         // This event may be a duplicate due to topology change
         // Handle idempotently
      }
   }
}
```

---

## Hot Rod Event Filtering (Server-Side)

Deploy server-side filters to reduce the number of events sent to clients.

### Step 1: Implement the Filter Factory

```java
import org.infinispan.notifications.cachelistener.filter.CacheEventFilterFactory;
import org.infinispan.notifications.cachelistener.filter.CacheEventFilter;
import org.infinispan.filter.NamedFactory;

@NamedFactory(name = "my-filter")
public class MyCacheEventFilterFactory implements CacheEventFilterFactory {
   @Override
   public CacheEventFilter<String, String> getFilter(Object[] params) {
      return new MyCacheEventFilter(params);
   }
}

// Must be marshallable in a cluster
class MyCacheEventFilter implements CacheEventFilter<String, String>, Serializable {
   private final Object[] params;

   MyCacheEventFilter(Object[] params) {
      this.params = params;
   }

   @Override
   public boolean accept(String key, String oldValue, Metadata oldMetadata,
                          String newValue, Metadata newMetadata, EventType eventType) {
      return key.equals(params[0]); // Only accept events for a specific key
   }
}
```

### Step 2: Deploy to Server

1. Package the filter in a JAR file.
2. Create `META-INF/services/org.infinispan.notifications.cachelistener.filter.CacheEventFilterFactory` containing the fully qualified class name.
3. Place the JAR in the `server/lib` directory.

### Step 3: Use in Client Listener

```java
@ClientListener(filterFactoryName = "my-filter")
public class FilteredListener {
   @ClientCacheEntryCreated
   public void handleCreated(ClientCacheEntryCreatedEvent<String> event) {
      // Only receives events that pass the filter
   }
}

// Register with dynamic filter parameters
cache.addClientListener(new FilteredListener(), new Object[]{"targetKey"}, null);
```

---

## Hot Rod Custom Events (Converters)

Use converters to customize the event payload sent to clients.

### Converter Factory

```java
import org.infinispan.notifications.cachelistener.filter.CacheEventConverterFactory;
import org.infinispan.notifications.cachelistener.filter.CacheEventConverter;
import org.infinispan.filter.NamedFactory;

@NamedFactory(name = "my-converter")
public class MyConverterFactory implements CacheEventConverterFactory {
   @Override
   public CacheEventConverter<String, String, CustomEvent> getConverter(Object[] params) {
      return new MyCacheEventConverter();
   }
}

class MyCacheEventConverter implements CacheEventConverter<String, String, CustomEvent>, Serializable {
   @Override
   public CustomEvent convert(String key, String oldValue, Metadata oldMetadata,
                                String newValue, Metadata newMetadata, EventType eventType) {
      return new CustomEvent(key, newValue);
   }
}

// Must be marshallable
@Proto
public record CustomEvent(String key, String value) {}
```

### Client Listener for Custom Events

```java
@ClientListener(converterFactoryName = "my-converter")
public class CustomEventListener {

   @ClientCacheEntryCreated
   @ClientCacheEntryModified
   @ClientCacheEntryRemoved
   public void handleCustomEvent(ClientCacheEntryCustomEvent<CustomEvent> event) {
      CustomEvent data = event.getEventData();
      System.out.printf("Custom event: key=%s, value=%s, type=%s%n",
            data.key(), data.value(), event.getType());
   }
}
```

### Combined Filter and Converter

For efficiency, implement both filtering and conversion in a single class using `CacheEventFilterConverter`:

```java
import org.infinispan.notifications.cachelistener.filter.AbstractCacheEventFilterConverter;
import org.infinispan.filter.NamedFactory;

@NamedFactory(name = "my-filter-converter")
public class MyFilterConverterFactory implements CacheEventFilterConverterFactory {
   @Override
   public CacheEventFilterConverter<String, String, CustomEvent> getFilterConverter(Object[] params) {
      return new MyFilterConverter(params);
   }
}

class MyFilterConverter extends AbstractCacheEventFilterConverter<String, String, CustomEvent>
      implements Serializable {

   private final Object[] params;

   MyFilterConverter(Object[] params) {
      this.params = params;
   }

   @Override
   public CustomEvent filterAndConvert(String key, String oldValue, Metadata oldMetadata,
                                        String newValue, Metadata newMetadata, EventType eventType) {
      // Return null to filter out the event, return a value to include it
      if (key.startsWith((String) params[0])) {
         return new CustomEvent(key, newValue);
      }
      return null; // Filtered out
   }
}
```

Register with matching filter and converter factory names:

```java
@ClientListener(filterFactoryName = "my-filter-converter", converterFactoryName = "my-filter-converter")
public class FilteredCustomEventListener {
   @ClientCacheEntryCreated
   @ClientCacheEntryModified
   public void handle(ClientCacheEntryCustomEvent<CustomEvent> event) {
      // Receives only filtered and converted events
   }
}

cache.addClientListener(new FilteredCustomEventListener(), new Object[]{"prefix"}, null);
```

---

## CDI Events Integration

In Jakarta EE / CDI environments, you can observe Infinispan cache events using standard CDI `@Observes`.

```java
import jakarta.enterprise.event.Observes;
import org.infinispan.notifications.cachemanagerlistener.event.CacheStartedEvent;
import org.infinispan.notifications.cachelistener.event.*;

public class CacheEventObserver {

   // Cache-level events
   public void onEntryCreated(@Observes CacheEntryCreatedEvent event) {
      System.out.printf("CDI: Entry created in cache %s%n", event.getCache().getName());
   }

   // Cache manager-level events
   public void onCacheStarted(@Observes CacheStartedEvent event) {
      System.out.printf("CDI: Cache started: %s%n", event.getCacheName());
   }
}
```

**Note:** CDI integration requires the Infinispan CDI module on the classpath.

---

## Common Mistakes and Pitfalls

| Mistake | Problem | Fix |
|---------|---------|-----|
| Blocking synchronous listener with heavy processing | Delays all cache operations on the calling thread | Use non-blocking listener (return `CompletionStage<Void>`) or async listener (`sync = false`) |
| Not handling duplicate events | Data inconsistency in non-transactional caches | Check `isCommandRetried()` and implement idempotent handlers |
| No server-side filtering for Hot Rod listeners | Excessive network traffic and event volume | Deploy `CacheEventFilter` on the server to filter at source |
| Listener throws exception | Sync listeners can abort the cache operation | Wrap listener logic in try-catch, log errors |
| Too many client listeners | Server backpressure kicks in, delays write responses | Use fewer listeners with proper filtering; consider continuous queries |
| Forgetting to remove client listeners | Resource leak on server; orphaned event queues | Always call `cache.removeClientListener(listener)` when done |
| Using clustered listener for all event types | Silently ignores unsupported event types | Clustered listeners only support Created, Modified, Removed, and Expired |
| Filter/converter not marshallable | Fails in clustered deployments | Implement `Serializable` and use `@Proto` for ProtoStream |

## Performance Considerations

- **Embedded listeners** share CPU with Infinispan. Heavy listener processing reduces throughput for all cache operations.
- **Hot Rod listeners** involve server-to-client network traffic. The server sends events from the primary owner to the listener's registered node, then to the client.
- **Backpressure mechanism**: When a client listener cannot keep up, the server applies backpressure by delaying write responses. If the queue grows too large, the server may close the client connection.
- **Backpressure tuning**: Configure backpressure thresholds in the Hot Rod connector settings.
- **Continuous queries** can be an alternative to client listeners when you already use indexing and need event filtering based on query predicates.

## Official Docs

- Listeners and notifications: https://infinispan.org/docs/stable/titles/developing/developing.html#listeners-notifications
- Hot Rod client listeners: https://infinispan.org/docs/stable/titles/hotrod_java/hotrod_java.html
- Performance tuning: https://infinispan.org/docs/stable/titles/tuning/tuning.html
