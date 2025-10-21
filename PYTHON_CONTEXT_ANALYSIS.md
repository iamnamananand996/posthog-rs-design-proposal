# Deep Analysis: PostHog Python SDK Context Implementation

## Executive Summary

This document provides a comprehensive analysis of the PostHog Python SDK's context implementation, focusing on how it can be adapted to Rust for the posthog-rs SDK. The Python implementation uses `contextvars` for thread-safe, async-compatible context management with hierarchical tag inheritance.

---

## 1. Context Implementation Details (`posthog/contexts.py`)

### 1.1 Core Architecture

The Python implementation uses a **single ContextVar** to maintain a stack of context scopes:

```python
_context_stack: contextvars.ContextVar[Optional[ContextScope]] = contextvars.ContextVar(
    "posthog_context_stack", default=None
)
```

**Key Design Choice**: Instead of storing a list/stack directly, each `ContextScope` maintains a reference to its parent, forming a linked list structure.

### 1.2 ContextScope Class

```python
class ContextScope:
    def __init__(
        self,
        parent=None,
        fresh: bool = False,
        capture_exceptions: bool = True,
        client: Optional["Client"] = None,
    ):
        self.client: Optional[Client] = client
        self.parent = parent
        self.fresh = fresh
        self.capture_exceptions = capture_exceptions
        self.session_id: Optional[str] = None
        self.distinct_id: Optional[str] = None
        self.tags: Dict[str, Any] = {}
```

**Important Properties**:
- `parent`: Reference to parent scope (linked list)
- `fresh`: If `True`, doesn't inherit from parent
- `capture_exceptions`: Auto-capture exceptions in this scope
- `client`: Optional client override for exception capture
- `session_id`, `distinct_id`: Identity state
- `tags`: Key-value properties

### 1.3 Tag Collection with Inheritance

```python
def collect_tags(self) -> Dict[str, Any]:
    tags = self.tags.copy()
    if self.parent and not self.fresh:
        # We want child tags to take precedence over parent tags,
        # so we can't use a simple update here, instead collecting
        # the parent tags and then updating with the child tags.
        new_tags = self.parent.collect_tags()
        tags.update(new_tags)  # This looks backwards but isn't!
    return tags
```

**Critical Insight**: The comment is important! The order seems backwards, but it's correct:
1. Start with child tags
2. Get parent tags (which recursively collects grandparent tags)
3. Update child with parent (child takes precedence due to dict update semantics)

**Rust Adaptation**: This pattern translates well to Rust using recursive collection:
```rust
fn collect_tags(&self) -> HashMap<String, Value> {
    let mut tags = self.tags.clone();
    if let Some(parent) = &self.parent {
        if !self.fresh {
            let parent_tags = parent.collect_tags();
            // Insert parent tags that don't exist in child
            for (k, v) in parent_tags {
                tags.entry(k).or_insert(v);
            }
        }
    }
    tags
}
```

### 1.4 Identity Inheritance

```python
def get_distinct_id(self) -> Optional[str]:
    if self.distinct_id is not None:
        return self.distinct_id
    if self.parent is not None and not self.fresh:
        return self.parent.get_distinct_id()
    return None

def get_session_id(self) -> Optional[str]:
    if self.session_id is not None:
        return self.session_id
    if self.parent is not None and not self.fresh:
        return self.parent.get_session_id()
    return None
```

**Pattern**: Check self first, then recursively check parent (if not fresh).

**Rust Adaptation**: Similar recursive pattern:
```rust
fn get_distinct_id(&self) -> Option<&str> {
    if let Some(id) = &self.distinct_id {
        return Some(id.as_str());
    }
    if !self.fresh {
        if let Some(parent) = &self.parent {
            return parent.get_distinct_id();
        }
    }
    None
}
```

### 1.5 Context Manager Implementation

```python
@contextmanager
def new_context(
    fresh: bool = False,
    capture_exceptions: bool = True,
    client: Optional["Client"] = None,
):
    from posthog import capture_exception

    current_context = _get_current_context()
    new_context = ContextScope(current_context, fresh, capture_exceptions, client)
    _context_stack.set(new_context)

    try:
        yield
    except Exception as e:
        if new_context.capture_exceptions:
            if new_context.client:
                new_context.client.capture_exception(e)
            else:
                capture_exception(e)
        raise
    finally:
        _context_stack.set(new_context.get_parent())
```

**Flow**:
1. Get current context (may be None)
2. Create new scope with current as parent
3. Set as active context
4. Yield control to user code
5. On exception: capture if enabled, then re-raise
6. Finally: restore parent context

**Rust Adaptation Challenge**: Rust doesn't have context managers. Options:
1. **Drop trait**: Restore context on drop (like Python's finally)
2. **Guard pattern**: Return a guard that restores on drop
3. **Explicit API**: Separate `enter()/exit()` methods

**Recommended Rust Pattern**:
```rust
pub struct ContextGuard {
    previous: Option<Arc<ContextScope>>,
}

impl Drop for ContextGuard {
    fn drop(&mut self) {
        CONTEXT_STACK.with(|stack| {
            *stack.borrow_mut() = self.previous.take();
        });
    }
}

pub fn new_context(fresh: bool, capture_exceptions: bool) -> ContextGuard {
    let current = get_current_context();
    let new_scope = Arc::new(ContextScope::new(
        current.clone(),
        fresh,
        capture_exceptions,
    ));

    let guard = ContextGuard {
        previous: current,
    };

    CONTEXT_STACK.with(|stack| {
        *stack.borrow_mut() = Some(new_scope);
    });

    guard
}
```

### 1.6 Decorator Implementation

```python
def scoped(fresh: bool = False, capture_exceptions: bool = True):
    def decorator(func: F) -> F:
        from functools import wraps

        @wraps(func)
        def wrapper(*args, **kwargs):
            with new_context(fresh=fresh, capture_exceptions=capture_exceptions):
                return func(*args, **kwargs)

        return cast(F, wrapper)

    return decorator
```

**Rust Adaptation**: Procedural macros could provide similar functionality:
```rust
#[posthog::scoped]
fn my_function() {
    // Automatically wrapped in context
}

// Or using a closure-based approach:
posthog::with_context(|| {
    // Code here runs in new context
});
```

---

## 2. Integration with Client (`posthog/client.py`)

### 2.1 Identity Resolution

```python
def get_identity_state(passed) -> tuple[str, bool]:
    """Returns the distinct id to use, and whether this is a personless event or not"""
    stringified = stringify_id(passed)
    if stringified and len(stringified):
        return (stringified, False)

    context_id = get_context_distinct_id()
    if context_id:
        return (context_id, False)

    return (str(uuid4()), True)
```

**Priority Order**:
1. Explicitly passed distinct_id (highest priority)
2. Context-level distinct_id
3. Random UUID (marked as "personless")

**Rust Adaptation**:
```rust
fn get_identity_state(passed: Option<&str>) -> (String, bool) {
    if let Some(id) = passed {
        if !id.is_empty() {
            return (id.to_string(), false);
        }
    }

    if let Some(context_id) = get_context_distinct_id() {
        return (context_id.to_string(), false);
    }

    (Uuid::new_v4().to_string(), true)
}
```

### 2.2 Property Merging

```python
def add_context_tags(properties):
    properties = properties or {}
    current_context = _get_current_context()
    if current_context:
        context_tags = current_context.collect_tags()
        properties["$context_tags"] = set(context_tags.keys())
        # We want explicitly passed properties to override context tags
        context_tags.update(properties)
        properties = context_tags

    if "$session_id" not in properties and get_context_session_id():
        properties["$session_id"] = get_context_session_id()

    return properties
```

**Merging Strategy**:
1. Start with context tags
2. Add a special `$context_tags` property with tag keys
3. Override with explicitly passed properties (passed properties win)
4. Add `$session_id` if not already present

**Important**: Context tags are **data**, not metadata. They become regular event properties.

**Rust Adaptation**:
```rust
fn add_context_tags(mut properties: Map<String, Value>) -> Map<String, Value> {
    if let Some(context) = get_current_context() {
        let context_tags = context.collect_tags();

        // Store context tag keys
        let tag_keys: Vec<String> = context_tags.keys().cloned().collect();
        properties.insert(
            "$context_tags".to_string(),
            Value::Array(tag_keys.into_iter().map(Value::String).collect())
        );

        // Merge: context tags first, then override with passed properties
        let mut merged = context_tags;
        for (k, v) in properties {
            merged.insert(k, v);
        }
        properties = merged;
    }

    // Add session_id if not present
    if !properties.contains_key("$session_id") {
        if let Some(session_id) = get_context_session_id() {
            properties.insert("$session_id".to_string(), Value::String(session_id));
        }
    }

    properties
}
```

### 2.3 Capture Method Integration

```python
@no_throw()
def capture(
    self, event: str, **kwargs: Unpack[OptionalCaptureArgs]
) -> Optional[str]:
    distinct_id = kwargs.get("distinct_id", None)
    properties = kwargs.get("properties", None)
    # ... other kwargs ...

    properties = {**(properties or {}), **system_context()}
    properties = add_context_tags(properties)  # <-- Context integration

    (distinct_id, personless) = get_identity_state(distinct_id)

    if personless and "$process_person_profile" not in properties:
        properties["$process_person_profile"] = False

    msg = {
        "properties": properties,
        "timestamp": timestamp,
        "distinct_id": distinct_id,
        "event": event,
        "uuid": uuid,
    }

    # ... feature flags, etc ...

    return self._enqueue(msg, disable_geoip)
```

**Integration Points**:
1. Properties merge happens early
2. Context tags are added via `add_context_tags()`
3. Identity resolution uses context as fallback
4. Personless events get special flag

---

## 3. Web Framework Integrations

### 3.1 Django Middleware (`posthog/integrations/django.py`)

```python
class PosthogContextMiddleware:
    sync_capable = True
    async_capable = True  # Supports both!

    def __init__(self, get_response):
        self._is_coroutine = iscoroutinefunction(get_response)
        # ... extract settings ...
        self.extra_tags = settings.POSTHOG_MW_EXTRA_TAGS  # Callable[[HttpRequest], Dict[str, Any]]
        self.request_filter = settings.POSTHOG_MW_REQUEST_FILTER  # Callable[[HttpRequest], bool]
        self.tag_map = settings.POSTHOG_MW_TAG_MAP  # Callable[[Dict], Dict]
        self.capture_exceptions = settings.POSTHOG_MW_CAPTURE_EXCEPTIONS  # bool
        self.client = settings.POSTHOG_MW_CLIENT  # Optional[Client]

    def extract_tags(self, request):
        tags = {}

        # Extract user from Django request
        (user_id, user_email) = self.extract_request_user(request)

        # Extract from headers
        session_id = request.headers.get("X-POSTHOG-SESSION-ID")
        if session_id:
            contexts.set_context_session(session_id)

        distinct_id = request.headers.get("X-POSTHOG-DISTINCT-ID") or user_id
        if distinct_id:
            contexts.identify_context(distinct_id)

        # Extract request metadata
        if user_email:
            tags["email"] = user_email
        tags["$current_url"] = request.build_absolute_uri()
        tags["$request_method"] = request.method
        tags["$request_path"] = request.path

        # Extract forwarding headers
        if ip := request.headers.get("X-Forwarded-For"):
            tags["$ip_address"] = ip
        if ua := request.headers.get("User-Agent"):
            tags["$user_agent"] = ua

        # Apply customizations
        if self.extra_tags:
            extra = self.extra_tags(request)
            if extra:
                tags.update(extra)

        if self.tag_map:
            tags = self.tag_map(tags)

        return tags

    def __call__(self, request):  # Sync version
        if self.request_filter and not self.request_filter(request):
            return self._sync_get_response(request)

        with contexts.new_context(self.capture_exceptions, client=self.client):
            for k, v in self.extract_tags(request).items():
                contexts.tag(k, v)
            return self._sync_get_response(request)

    async def __acall__(self, request):  # Async version
        if self.request_filter and not self.request_filter(request):
            return await self._async_get_response(request)

        with contexts.new_context(self.capture_exceptions, client=self.client):
            for k, v in self.extract_tags(request).items():
                contexts.tag(k, v)
            return await self._async_get_response(request)
```

**Key Features**:
1. **Header Extraction**: `X-POSTHOG-SESSION-ID`, `X-POSTHOG-DISTINCT-ID`
2. **User Detection**: Django's `request.user`
3. **Automatic Tagging**: URL, method, path, IP, user agent
4. **Customization Hooks**:
   - `extra_tags`: Add custom tags
   - `tag_map`: Transform tags before applying
   - `request_filter`: Skip tracking for certain requests
5. **Client Override**: Can use custom client per request
6. **Dual Mode**: Same logic for sync and async

**Rust Adaptation for Web Frameworks**:

For **Axum**:
```rust
pub struct PosthogContextLayer {
    extra_tags: Option<Arc<dyn Fn(&Request) -> HashMap<String, Value> + Send + Sync>>,
    request_filter: Option<Arc<dyn Fn(&Request) -> bool + Send + Sync>>,
    capture_exceptions: bool,
}

impl<S> Layer<S> for PosthogContextLayer {
    type Service = PosthogContextService<S>;

    fn layer(&self, inner: S) -> Self::Service {
        PosthogContextService {
            inner,
            config: self.clone(),
        }
    }
}

impl<S> Service<Request> for PosthogContextService<S>
where
    S: Service<Request>,
{
    async fn call(&mut self, req: Request) -> Result<Response, Error> {
        let _guard = new_context(false, self.config.capture_exceptions);

        // Extract headers
        if let Some(session_id) = req.headers().get("X-POSTHOG-SESSION-ID") {
            set_context_session(session_id.to_str()?);
        }
        if let Some(distinct_id) = req.headers().get("X-POSTHOG-DISTINCT-ID") {
            identify_context(distinct_id.to_str()?);
        }

        // Add request tags
        tag("$request_method", req.method().as_str());
        tag("$request_path", req.uri().path());

        // Call next service
        self.inner.call(req).await
    }
}
```

For **Actix-Web**:
```rust
pub struct PosthogContext {
    capture_exceptions: bool,
}

impl<S, B> Transform<S, ServiceRequest> for PosthogContext
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
{
    async fn call(&self, req: ServiceRequest) -> Result<ServiceResponse<B>, Error> {
        let _guard = new_context(false, self.capture_exceptions);

        // Extract and set context
        if let Some(session_id) = req.headers().get("X-POSTHOG-SESSION-ID") {
            set_context_session(session_id.to_str()?);
        }

        // ... similar to Axum ...

        req.call(srv).await
    }
}
```

---

## 4. Exception Handling

### 4.1 Exception Capture Flow

```python
# In new_context:
try:
    yield
except Exception as e:
    if new_context.capture_exceptions:
        if new_context.client:
            new_context.client.capture_exception(e)
        else:
            capture_exception(e)
    raise
```

**Important**: Exception is captured THEN re-raised. This ensures:
1. Exception is tracked in PostHog
2. Normal Python error handling continues
3. Stack traces are preserved

### 4.2 Duplicate Prevention

```python
def exception_is_already_captured(error):
    if isinstance(error, BaseException):
        return hasattr(error, "__posthog_exception_captured")
    elif isinstance(error, tuple) and len(error) > 1:
        return error[1] is not None and hasattr(
            error[1], "__posthog_exception_captured"
        )
    return False

def mark_exception_as_captured(error, uuid):
    if isinstance(error, BaseException):
        setattr(error, "__posthog_exception_captured", True)
        setattr(error, "__posthog_exception_uuid", uuid)
    # ... handle tuples ...
```

**Mechanism**: Attaches attributes to exception objects to prevent duplicate capture.

**Rust Adaptation Challenge**: Rust doesn't allow adding arbitrary data to error types.

**Solutions**:
1. **Thread-local set**: Track exception pointers
   ```rust
   thread_local! {
       static CAPTURED_EXCEPTIONS: RefCell<HashSet<*const dyn Error>> = RefCell::new(HashSet::new());
   }
   ```

2. **Error wrapper**: Create wrapper type with capture flag
   ```rust
   pub struct CapturedError {
       inner: Box<dyn Error>,
       captured: bool,
       uuid: Option<String>,
   }
   ```

3. **Accept duplicates**: Simpler, rely on server-side deduplication

### 4.3 Exception Autocapture

```python
class ExceptionCapture:
    def __init__(self, client: "Client"):
        self.client = client
        self.original_excepthook = sys.excepthook
        sys.excepthook = self.exception_handler
        threading.excepthook = self.thread_exception_handler

    def exception_handler(self, exc_type, exc_value, exc_traceback):
        self.capture_exception((exc_type, exc_value, exc_traceback))
        self.original_excepthook(exc_type, exc_value, exc_traceback)

    def thread_exception_handler(self, args):
        self.capture_exception((args.exc_type, args.exc_value, args.exc_traceback))
```

**Mechanism**: Hooks into Python's global exception handlers.

**Rust Adaptation**:
- **No direct equivalent** (Rust panics aren't caught this way)
- Use **panic hooks**:
  ```rust
  std::panic::set_hook(Box::new(|panic_info| {
      if let Some(context) = get_current_context() {
          // Extract panic info and capture
          capture_panic(panic_info);
      }
  }));
  ```

### 4.4 Exception Data Format

```python
properties = {
    "$exception_type": all_exceptions_with_trace_and_in_app[0].get("type"),
    "$exception_message": all_exceptions_with_trace_and_in_app[0].get("value"),
    "$exception_list": all_exceptions_with_trace_and_in_app,
    **properties,
}
```

**Structure**:
- `$exception_type`: Class name (e.g., "ValueError")
- `$exception_message`: Error message
- `$exception_list`: Array of exception frames (for chained exceptions)

Each frame includes:
```python
{
    "type": "ValueError",
    "value": "Invalid input",
    "module": "myapp.views",
    "stacktrace": {
        "frames": [
            {
                "filename": "views.py",
                "abs_path": "/app/myapp/views.py",
                "function": "process_request",
                "lineno": 42,
                "pre_context": ["    # Previous lines"],
                "context_line": "    raise ValueError('Invalid input')",
                "post_context": ["    # Following lines"],
                "in_app": true
            }
        ]
    }
}
```

---

## 5. API Design Patterns

### 5.1 Public API Structure

```python
# Module-level functions wrap client methods
def capture(event: str, **kwargs):
    return _proxy("capture", event, **kwargs)

def tag(name: str, value: Any):
    return inner_tag(name, value)

def identify_context(distinct_id: str):
    return inner_identify_context(distinct_id)

# _proxy lazy-initializes default client
def _proxy(method, *args, **kwargs):
    setup()  # Creates default_client if needed
    fn = getattr(default_client, method)
    return fn(*args, **kwargs)
```

**Pattern**:
1. Module-level convenience functions
2. Lazy initialization of global client
3. Forward calls to client methods

**Rust Adaptation**:
```rust
// Module-level functions
pub fn capture(event: &str, properties: Option<Properties>) -> Result<()> {
    get_or_init_client().capture(event, properties)
}

pub fn tag(key: &str, value: impl Into<Value>) {
    get_current_context().map(|ctx| ctx.tag(key, value.into()));
}

// Lazy static client
static DEFAULT_CLIENT: OnceLock<Client> = OnceLock::new();

fn get_or_init_client() -> &'static Client {
    DEFAULT_CLIENT.get_or_init(|| {
        Client::new(get_api_key())
    })
}
```

### 5.2 Builder Pattern for Client

```python
# Settings as module-level variables
api_key = None
host = None
debug = False
# ...

# Client built from settings
def setup() -> Client:
    global default_client
    if not default_client:
        default_client = Client(
            api_key,
            host=host,
            debug=debug,
            # ... all settings ...
        )
    return default_client
```

**Rust Improvement**: Use proper builder pattern:
```rust
let client = Client::builder()
    .api_key("key")
    .host("https://app.posthog.com")
    .debug(true)
    .enable_local_evaluation(true)
    .poll_interval(Duration::from_secs(30))
    .build()?;
```

### 5.3 Optional Arguments Pattern

Python uses `TypedDict` with `NotRequired`:
```python
class OptionalCaptureArgs(TypedDict):
    distinct_id: NotRequired[Optional[ID_TYPES]]
    properties: NotRequired[Optional[Dict[str, Any]]]
    timestamp: NotRequired[Optional[Union[datetime, str]]]
    # ...

def capture(event: str, **kwargs: Unpack[OptionalCaptureArgs]):
    distinct_id = kwargs.get("distinct_id", None)
    properties = kwargs.get("properties", None)
    # ...
```

**Rust Options**:

1. **Builder pattern** (most idiomatic):
   ```rust
   client.capture("event")
       .distinct_id("user-123")
       .property("key", "value")
       .send()?;
   ```

2. **Options struct**:
   ```rust
   struct CaptureOptions {
       distinct_id: Option<String>,
       properties: Option<HashMap<String, Value>>,
       timestamp: Option<DateTime<Utc>>,
   }

   client.capture("event", CaptureOptions {
       distinct_id: Some("user-123".into()),
       ..Default::default()
   })?;
   ```

3. **Separate methods**:
   ```rust
   client.capture("event")?;
   client.capture_with_id("event", "user-123")?;
   client.capture_with_properties("event", properties)?;
   ```

---

## 6. Thread/Async Safety

### 6.1 contextvars Behavior

Python's `contextvars` provides:
1. **Thread isolation**: Each thread has its own context
2. **Async isolation**: Each async task has its own context
3. **Automatic copying**: Child tasks inherit parent's context

```python
import asyncio
from contextvars import ContextVar

ctx_var: ContextVar[int] = ContextVar('ctx', default=0)

async def child_task():
    print(f"Child sees: {ctx_var.get()}")  # Inherits parent's value
    ctx_var.set(20)
    print(f"Child changed to: {ctx_var.get()}")

async def parent_task():
    ctx_var.set(10)
    await child_task()
    print(f"Parent still has: {ctx_var.get()}")  # Still 10!

# Output:
# Child sees: 10
# Child changed to: 20
# Parent still has: 10
```

**Rust Equivalent**: `tokio::task_local!` + manual management

```rust
tokio::task_local! {
    static CONTEXT_SCOPE: Arc<ContextScope>;
}

// Need explicit spawning with context
let ctx = get_current_context();
tokio::spawn(async move {
    CONTEXT_SCOPE.scope(ctx.clone(), async {
        // Task code here
    }).await
});
```

### 6.2 Thread Safety in Python Implementation

```python
# ContextScope is NOT thread-safe itself
class ContextScope:
    def __init__(self, ...):
        self.tags: Dict[str, Any] = {}  # Plain dict, no locking
```

**Why it works**:
- Each thread has its own ContextVar
- No shared state between threads
- Modifications are thread-local

**Rust Implication**:
- Can use `thread_local!` for similar isolation
- No need for `Arc<Mutex<>>` for the scope itself
- Only need sync primitives for shared client state

### 6.3 Async Context Propagation

Python automatically propagates context to:
- `asyncio.create_task()`
- `asyncio.gather()`
- Async context managers

**Rust Challenge**: No automatic propagation. Must explicitly pass context.

**Solution**: Use `tokio::task_local!` with scoped spawning:
```rust
tokio::task_local! {
    static CONTEXT: Arc<ContextScope>;
}

pub async fn with_context<F, T>(ctx: Arc<ContextScope>, f: F) -> T
where
    F: Future<Output = T>,
{
    CONTEXT.scope(ctx, f).await
}

// Usage:
let ctx = Arc::new(ContextScope::new(...));
with_context(ctx.clone(), async {
    // This code has access to context
    tag("key", "value");
}).await;
```

---

## 7. Property Management

### 7.1 Special Properties

PostHog recognizes certain property prefixes:
- `$set`: Set person properties (persistent)
- `$set_once`: Set only if not already set
- `$context_tags`: List of context tag keys
- `$session_id`: Session identifier
- `$process_person_profile`: Whether to create/update person

```python
def set(self, **kwargs):
    properties = kwargs.get("properties", None) or {}
    properties = add_context_tags(properties)
    (distinct_id, personless) = get_identity_state(kwargs.get("distinct_id"))

    if personless or not properties:
        return None  # Personless set() does nothing

    msg = {
        "distinct_id": distinct_id,
        "$set": properties,
        "event": "$set",
    }
    return self._enqueue(msg)
```

**Key Insight**: Context tags are added to `$set` operations, so they become person properties!

### 7.2 Property Merging Precedence

From highest to lowest priority:
1. **Explicit properties** passed to `capture()`
2. **Context tags** from current scope
3. **Super properties** (global properties)
4. **System context** (SDK version, OS, etc.)

```python
def capture(self, event: str, **kwargs):
    properties = kwargs.get("properties", None)
    properties = {**(properties or {}), **system_context()}
    properties = add_context_tags(properties)  # Context tags go in, explicit props override

    # If super_properties enabled:
    if self.super_properties:
        properties = {**self.super_properties, **properties}
```

**Rust Implementation**:
```rust
fn build_properties(
    explicit: Option<Properties>,
    context_tags: Properties,
    super_props: &Properties,
) -> Properties {
    let mut props = Properties::new();

    // 1. System context (lowest priority)
    props.extend(system_context());

    // 2. Super properties
    props.extend(super_props.clone());

    // 3. Context tags
    props.extend(context_tags);

    // 4. Explicit properties (highest priority)
    if let Some(explicit) = explicit {
        props.extend(explicit);
    }

    props
}
```

### 7.3 Group Properties

```python
def group_identify(
    self,
    group_type: str,
    group_key: str,
    properties: Optional[Dict] = None,
):
    msg = {
        "event": "$groupidentify",
        "properties": {
            "$group_type": group_type,
            "$group_key": group_key,
            "$group_set": properties,
        },
        "distinct_id": get_identity_state(distinct_id)[0],
    }

    # NOTE - group_identify doesn't generally use context properties
    if get_context_session_id():
        msg["properties"]["$session_id"] = get_context_session_id()

    return self._enqueue(msg)
```

**Note**: Group operations don't include context tags by default, only session_id.

---

## 8. Testing Patterns

### 8.1 Context Isolation Tests

```python
def test_new_context_isolation(self):
    with new_context(fresh=True):
        tag("outer", "value")

        with new_context(fresh=True):
            # Inner context should start empty
            assert get_tags() == {}
            tag("inner", "value")
            assert get_tags()["inner"] == "value"
            self.assertNotIn("outer", get_tags())

        with new_context(fresh=False):
            # Should inherit outer tag
            assert get_tags() == {"outer": "value"}

        # After exiting, inner tag should be gone
        self.assertNotIn("inner", get_tags())
        assert get_tags()["outer"] == "value"
```

**Test Coverage**:
- Fresh context starts empty
- Non-fresh context inherits parent
- Exiting context restores previous state
- Child modifications don't affect parent

**Rust Test Pattern**:
```rust
#[test]
fn test_context_isolation() {
    let _guard1 = new_context(true, false);
    tag("outer", "value");

    {
        let _guard2 = new_context(true, false);
        assert_eq!(get_tags().len(), 0);
        tag("inner", "value");
        assert!(get_tags().contains_key("inner"));
        assert!(!get_tags().contains_key("outer"));
    }

    {
        let _guard3 = new_context(false, false);
        assert_eq!(get_tags().get("outer"), Some(&Value::String("value".into())));
    }

    assert!(!get_tags().contains_key("inner"));
    assert!(get_tags().contains_key("outer"));
}
```

### 8.2 Exception Capture Tests

```python
@patch("posthog.capture_exception")
def test_scoped_decorator_exception(self, mock_capture):
    test_exception = ValueError("Test exception")

    def check_context_on_capture(exception, **kwargs):
        current_tags = get_tags()
        assert current_tags.get("important_context") == "value"

    mock_capture.side_effect = check_context_on_capture

    @scoped()
    def failing_function():
        tag("important_context", "value")
        raise test_exception

    with self.assertRaises(ValueError):
        failing_function()

    mock_capture.assert_called_once_with(test_exception)
    assert get_tags() == {}  # Context cleaned up
```

**Test Verifies**:
1. Tags are available when exception is captured
2. Exception is re-raised after capture
3. Context is cleaned up after function exits

### 8.3 Mock Utilities

```python
class MockRequest:
    def __init__(self, headers=None, method="GET", path="/test"):
        self.headers = headers or {}
        self.method = method
        self.path = path

    def build_absolute_uri(self):
        return f"http://example.com{self.path}"
```

**Rust Test Helpers**:
```rust
pub mod test_utils {
    pub fn mock_context() -> Arc<ContextScope> {
        Arc::new(ContextScope::new(None, false, false))
    }

    pub fn with_test_context<F>(f: F)
    where F: FnOnce() {
        let _guard = new_context(true, false);
        f();
    }

    pub struct MockClient {
        pub captured_events: Arc<Mutex<Vec<Event>>>,
    }

    impl MockClient {
        pub fn new() -> Self {
            Self {
                captured_events: Arc::new(Mutex::new(Vec::new())),
            }
        }

        pub fn assert_event_count(&self, expected: usize) {
            assert_eq!(self.captured_events.lock().unwrap().len(), expected);
        }
    }
}
```

---

## 9. Rust-Specific Adaptations

### 9.1 Context Storage

**Python**: Uses `contextvars.ContextVar`
**Rust Options**:

1. **Thread-local storage** (for sync):
   ```rust
   thread_local! {
       static CONTEXT_STACK: RefCell<Option<Arc<ContextScope>>> = RefCell::new(None);
   }

   pub fn get_current_context() -> Option<Arc<ContextScope>> {
       CONTEXT_STACK.with(|stack| stack.borrow().clone())
   }
   ```

2. **Task-local storage** (for async):
   ```rust
   tokio::task_local! {
       static CONTEXT_STACK: Arc<ContextScope>;
   }

   pub async fn with_context<F, T>(ctx: Arc<ContextScope>, f: F) -> T
   where F: Future<Output = T> {
       CONTEXT_STACK.scope(ctx, f).await
   }
   ```

3. **Hybrid approach** (recommended):
   ```rust
   // Sync contexts use thread_local
   thread_local! {
       static SYNC_CONTEXT: RefCell<Option<Arc<ContextScope>>> = RefCell::new(None);
   }

   // Async contexts use task_local
   tokio::task_local! {
       static ASYNC_CONTEXT: Arc<ContextScope>;
   }

   // Unified API that checks both
   pub fn get_current_context() -> Option<Arc<ContextScope>> {
       // Try task-local first (for async)
       if let Ok(ctx) = ASYNC_CONTEXT.try_with(|c| c.clone()) {
           return Some(ctx);
       }
       // Fall back to thread-local (for sync)
       SYNC_CONTEXT.with(|c| c.borrow().clone())
   }
   ```

### 9.2 Guard Pattern for Context Management

```rust
pub struct ContextGuard {
    previous: Option<Arc<ContextScope>>,
    _phantom: PhantomData<*const ()>, // Make !Send to prevent moving across threads
}

impl ContextGuard {
    pub fn new(fresh: bool, capture_exceptions: bool) -> Self {
        let previous = get_current_context();
        let new_scope = Arc::new(ContextScope::new(
            previous.clone(),
            fresh,
            capture_exceptions,
        ));

        set_current_context(Some(new_scope));

        Self {
            previous,
            _phantom: PhantomData,
        }
    }
}

impl Drop for ContextGuard {
    fn drop(&mut self) {
        set_current_context(self.previous.take());
    }
}

// Usage:
{
    let _guard = ContextGuard::new(false, true);
    tag("key", "value");
    // Guard dropped here, context restored
}
```

### 9.3 Type-Safe Property Values

Python uses `Any` for flexibility:
```python
def tag(key: str, value: Any):
    # value can be anything
```

Rust can use `serde_json::Value` or custom enum:
```rust
#[derive(Clone, Debug, Serialize)]
#[serde(untagged)]
pub enum PropertyValue {
    Null,
    Bool(bool),
    Number(f64),
    String(String),
    Array(Vec<PropertyValue>),
    Object(HashMap<String, PropertyValue>),
}

impl From<&str> for PropertyValue {
    fn from(s: &str) -> Self {
        PropertyValue::String(s.to_string())
    }
}

impl From<i64> for PropertyValue {
    fn from(n: i64) -> Self {
        PropertyValue::Number(n as f64)
    }
}

// Usage:
tag("string_prop", "value");
tag("number_prop", 42);
tag("bool_prop", true);
```

### 9.4 Error Handling Strategy

Python uses `@no_throw` decorator:
```python
@no_throw()
def capture(self, ...):
    # Never raises, logs instead
```

Rust options:

1. **Return Result**:
   ```rust
   pub fn capture(&self, event: &str) -> Result<EventId, Error> {
       // Caller handles errors
   }
   ```

2. **Panic on error** (for critical failures):
   ```rust
   pub fn capture(&self, event: &str) -> EventId {
       // Panics are caught by panic hook
   }
   ```

3. **Log and return default** (matches Python):
   ```rust
   pub fn capture(&self, event: &str) -> Option<EventId> {
       match self.try_capture(event) {
           Ok(id) => Some(id),
           Err(e) => {
               error!("Failed to capture event: {}", e);
               None
           }
       }
   }
   ```

### 9.5 Async Context Propagation

```rust
// Wrapper that propagates context to spawned tasks
pub async fn spawn_with_context<F, T>(f: F) -> JoinHandle<T>
where
    F: Future<Output = T> + Send + 'static,
    T: Send + 'static,
{
    let ctx = get_current_context();
    tokio::spawn(async move {
        if let Some(ctx) = ctx {
            set_current_context(Some(ctx));
        }
        f.await
    })
}

// Usage:
spawn_with_context(async {
    // This task has access to parent's context
    tag("in_background", true);
}).await?;
```

---

## 10. Recommended Implementation Plan for Rust

### Phase 1: Core Context System (Week 1)

1. **Define ContextScope struct**:
   ```rust
   pub struct ContextScope {
       parent: Option<Arc<ContextScope>>,
       fresh: bool,
       capture_exceptions: bool,
       session_id: Option<String>,
       distinct_id: Option<String>,
       tags: HashMap<String, Value>,
   }
   ```

2. **Implement storage**:
   - Thread-local for sync
   - Task-local for async
   - Unified getter function

3. **Create guard type**:
   - `ContextGuard` with `Drop` implementation
   - `new_context()` function returning guard

4. **Basic API**:
   - `tag(key, value)`
   - `identify_context(distinct_id)`
   - `set_context_session(session_id)`
   - `get_tags()`
   - `get_context_distinct_id()`
   - `get_context_session_id()`

### Phase 2: Client Integration (Week 2)

1. **Identity resolution**:
   - `get_identity_state()` function
   - Personless event handling

2. **Property merging**:
   - `add_context_tags()` function
   - Proper precedence order

3. **Update capture method**:
   - Integrate context tags
   - Support `$context_tags` property

4. **Add `set` and `set_once` methods**:
   - Context tag integration
   - Personless handling

### Phase 3: Exception Handling (Week 3)

1. **Panic hook integration**:
   - Global panic hook
   - Context extraction in hook
   - Capture panic as exception

2. **Exception data formatting**:
   - Extract stack trace
   - Build exception properties
   - Match PostHog format

3. **Duplicate prevention**:
   - Thread-local tracking
   - UUID generation

### Phase 4: Web Framework Integration (Week 4)

1. **Axum middleware**:
   - Header extraction
   - Request metadata tagging
   - Exception capture

2. **Actix-web middleware**:
   - Similar to Axum
   - Actix-specific patterns

3. **Rocket fairing**:
   - Request/response hooks
   - Context management

### Phase 5: Testing & Documentation (Week 5)

1. **Unit tests**:
   - Context isolation
   - Tag inheritance
   - Identity resolution
   - Property merging

2. **Integration tests**:
   - With real client
   - Mock server responses
   - Web framework tests

3. **Documentation**:
   - API docs
   - Usage examples
   - Migration guide

---

## 11. Key Differences: Python vs Rust

| Aspect | Python | Rust |
|--------|--------|------|
| **Context Storage** | `contextvars.ContextVar` | `thread_local!` / `task_local!` |
| **Scope Management** | Context manager (`with`) | Guard type (`Drop` trait) |
| **Type System** | Dynamic (`Any`) | Static (`Value` enum or generics) |
| **Error Handling** | Decorator `@no_throw` | `Result<T, E>` or `Option<T>` |
| **Async** | Automatic propagation | Manual with `task_local!` |
| **Exception Capture** | `sys.excepthook` | `std::panic::set_hook` |
| **Inheritance** | Automatic copying | Explicit `Arc::clone()` |
| **Thread Safety** | ContextVar handles it | Need `Arc` for shared data |

---

## 12. Code Examples: Side-by-Side

### Basic Context Usage

**Python**:
```python
with posthog.new_context():
    posthog.identify_context("user-123")
    posthog.tag("request_id", "abc")
    posthog.capture("button_clicked")
```

**Rust**:
```rust
{
    let _guard = posthog::new_context(false, true);
    posthog::identify_context("user-123");
    posthog::tag("request_id", "abc");
    posthog::capture("button_clicked", None)?;
}
```

### Fresh Context

**Python**:
```python
with posthog.new_context(fresh=True):
    # Starts with no inherited state
    posthog.tag("isolated", True)
```

**Rust**:
```rust
{
    let _guard = posthog::new_context(true, true);
    posthog::tag("isolated", true);
}
```

### Decorator/Function Wrapper

**Python**:
```python
@posthog.scoped()
def process_payment(payment_id):
    posthog.tag("payment_id", payment_id)
    posthog.capture("payment_started")
```

**Rust** (macro):
```rust
#[posthog::scoped]
fn process_payment(payment_id: &str) {
    posthog::tag("payment_id", payment_id);
    posthog::capture("payment_started", None).ok();
}
```

**Rust** (closure):
```rust
posthog::with_context(|| {
    posthog::tag("payment_id", payment_id);
    posthog::capture("payment_started", None).ok();
});
```

### Exception Capture

**Python**:
```python
with posthog.new_context(capture_exceptions=True):
    posthog.tag("operation", "import")
    risky_operation()  # Exception auto-captured if raised
```

**Rust**:
```rust
{
    let _guard = posthog::new_context(false, true);
    posthog::tag("operation", "import");
    risky_operation()?; // Need explicit error handling
}
```

### Web Middleware

**Python (Django)**:
```python
class PosthogContextMiddleware:
    def __call__(self, request):
        with contexts.new_context(self.capture_exceptions):
            session_id = request.headers.get("X-POSTHOG-SESSION-ID")
            if session_id:
                contexts.set_context_session(session_id)

            contexts.tag("$current_url", request.build_absolute_uri())
            return self.get_response(request)
```

**Rust (Axum)**:
```rust
async fn posthog_middleware<B>(
    req: Request<B>,
    next: Next<B>,
) -> Response {
    let _guard = posthog::new_context(false, true);

    if let Some(session_id) = req.headers().get("X-POSTHOG-SESSION-ID") {
        posthog::set_context_session(session_id.to_str().unwrap_or(""));
    }

    posthog::tag("$current_url", req.uri().to_string());
    next.run(req).await
}
```

---

## 13. Conclusion

The PostHog Python SDK's context implementation is well-designed and can be effectively adapted to Rust with the following key strategies:

### What Translates Well:
1. ✅ **Hierarchical scope structure** (parent references)
2. ✅ **Tag collection with precedence** (recursive merging)
3. ✅ **Identity resolution with fallbacks**
4. ✅ **Property merging strategies**
5. ✅ **Guard/RAII pattern** (Python's context manager → Rust's Drop)

### What Needs Rust-Specific Approaches:
1. ⚠️ **Context storage**: Use `thread_local!` + `task_local!` instead of `contextvars`
2. ⚠️ **Async propagation**: Manual management required
3. ⚠️ **Exception capture**: Use panic hooks, not exception hooks
4. ⚠️ **Type safety**: Use `Value` enum instead of `Any`
5. ⚠️ **Error handling**: Prefer `Result` over "never throw"

### Recommended Next Steps:
1. Start with sync-only implementation using `thread_local!`
2. Add async support with `task_local!` in separate PR
3. Implement basic middleware for one web framework (Axum)
4. Add comprehensive tests before expanding
5. Document patterns for users migrating from other SDKs

### Performance Considerations:
- Python's `contextvars` has minimal overhead; Rust's `thread_local!` is even faster
- Arc cloning for parent references is cheap (just ref count increment)
- No locks needed for scope data (each thread/task has its own)
- Context tag collection is O(depth), typically very shallow

This analysis should provide everything needed to implement a production-ready context system for posthog-rs that matches the Python SDK's capabilities while leveraging Rust's strengths in type safety and performance.
