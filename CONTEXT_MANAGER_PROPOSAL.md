# Context Manager for posthog-rs

## Overview
Add context management to posthog-rs for automatic propagation of `distinct_id`, `session_id`, and properties through call stacks - similar to Python SDK's context system.

## Problem
Currently every `capture()` call requires explicit parameters:
```rust
// Current: repetitive and error-prone
client.capture(Event::new("login", "user-123").property("plan", "pro"))?;
client.capture(Event::new("action", "user-123").property("plan", "pro"))?;
```

## Solution
RAII-based context management with automatic cleanup:
```rust
// Proposed: set once, use everywhere
let _ctx = Context::new()
    .distinct_id("user-123")
    .tag("plan", "pro")
    .activate();

client.capture(Event::new("login", None))?;  // Auto-uses context
client.capture(Event::new("action", None))?; // Auto-uses context
```

## Core Design

### Architecture
Based on Python SDK's linked-list pattern:
```rust
pub struct ContextScope {
    distinct_id: Option<String>,
    session_id: Option<String>,
    tags: HashMap<String, Value>,
    parent: Option<Arc<ContextScope>>,  // Parent context
    fresh: bool,  // If true, don't inherit
}

pub struct ContextGuard {
    previous: Option<Arc<ContextScope>>,
}

impl Drop for ContextGuard {
    fn drop(&mut self) {
        // Restore previous context
    }
}
```

### Storage Strategy
```rust
// Sync code
thread_local! {
    static CURRENT_CONTEXT: RefCell<Option<Arc<ContextScope>>>;
}

// Async code
tokio::task_local! {
    static ASYNC_CONTEXT: RefCell<Option<Arc<ContextScope>>>;
}
```

### Tag Inheritance
Child contexts override parent tags:
```rust
impl ContextScope {
    pub fn collect_tags(&self) -> HashMap<String, Value> {
        let mut tags = HashMap::new();

        // Parent tags first (unless fresh)
        if !self.fresh {
            if let Some(parent) = &self.parent {
                tags.extend(parent.collect_tags());
            }
        }

        // Child tags override
        tags.extend(self.tags.clone());
        tags
    }
}
```

## API Examples

### Basic Usage
```rust
// Simple context
let _guard = Context::new()
    .distinct_id("user-123")
    .session_id("session-456")
    .tag("plan", "enterprise")
    .activate();

// Convenience functions
posthog_rs::identify_context("user-123");
posthog_rs::tag("feature", "beta");
```

### Nested Contexts
```rust
let _parent = Context::new()
    .distinct_id("user-123")
    .tag("level", "parent")
    .activate();

{
    let _child = Context::new()
        .tag("level", "child")  // Overrides parent
        .tag("extra", "data")   // New tag
        .activate();

    // Uses: distinct_id=user-123, level=child, extra=data
}
// Back to parent context
```

### Web Framework
```rust
async fn handle_request(headers: HeaderMap) -> Response {
    let _ctx = Context::new()
        .distinct_id_opt(get_header(&headers, "X-POSTHOG-DISTINCT-ID"))
        .session_id_opt(get_header(&headers, "X-POSTHOG-SESSION-ID"))
        .tag("url", req.uri())
        .tag("method", req.method())
        .activate();

    // All events in request use context
    process_request().await
}
```

## Identity Resolution

3-tier fallback (matches Python SDK):
1. Explicit `distinct_id` on event (highest priority)
2. Context `distinct_id`
3. Generated UUID with `$personless` flag

```rust
fn resolve_distinct_id(&self, context: Option<&ContextScope>) -> String {
    self.distinct_id
        .or_else(|| context?.get_distinct_id())
        .unwrap_or_else(|| {
            self.properties.insert("$personless", true);
            Uuid::new_v7().to_string()
        })
}
```

## Property Merging Order

From lowest to highest priority:
1. Parent context tags
2. Current context tags
3. Event-specific properties
4. Special properties (`$context_tags`)

## Implementation Plan

### Core
- [ ] ContextScope with parent references
- [ ] Thread-local and task-local storage
- [ ] RAII guard with Drop trait
- [ ] Tag collection and inheritance

### Integration
- [ ] Update capture() to use context
- [ ] Identity resolution logic
- [ ] Panic hook integration
- [ ] Web framework helpers

### Polish
- [ ] Test suite (port Python tests)
- [ ] Documentation
- [ ] Performance benchmarks
- [ ] Migration guide

## Key Decisions

| Decision | Rationale |
|----------|-----------|
| **RAII Guards** | Automatic cleanup, Rust idiomatic |
| **Linked-list pattern** | Simple parent tracking, matches Python |
| **Thread/task-local** | Thread safety without locks |
| **Opt-in** | Zero breaking changes |
| **Arc for sharing** | Cheap cloning, safe concurrent access |

## Trade-offs

| Aspect | Trade-off | Mitigation |
|--------|-----------|------------|
| **Memory** | ~200 bytes per context | Negligible for typical use |
| **Cross-thread** | No auto-propagation | Document, allow manual clone |
| **Complexity** | Dual sync/async impl | Feature gates, shared logic |

## Performance Targets
- **Overhead**: < 5% when active, zero when not used
- **Activation**: < 1μs per context
- **Tag collection**: O(depth), typically 1-3 levels

## Migration

Fully backward compatible:
```rust
// Old code still works
client.capture(Event::new("action", "user-123"))?;

// New context-based approach
let _ctx = Context::new().distinct_id("user-123").activate();
client.capture(Event::new("action", None))?;
```

## Success Criteria
- ✅ Zero breaking changes
- ✅ Passes all existing tests
- ✅ Sub-microsecond overhead
- ✅ Works with async/sync
- ✅ Matches Python SDK behavior

## References
- [Python Context Implementation](https://github.com/PostHog/posthog-python/blob/main/posthog/contexts.py)
- [Original RFC #29](https://github.com/PostHog/posthog-rs/issues/29)