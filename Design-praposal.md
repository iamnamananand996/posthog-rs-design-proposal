# Feature Flags Design Summary for posthog-rs

## What are PostHog Feature Flags?

PostHog feature flags enable controlled feature rollouts, A/B testing, and targeted user experiences. They support:

### Flag Types

- **Boolean flags**: Simple on/off toggles for gradual rollouts or kill switches
- **Multivariate flags**: Multiple variants for A/B testing (e.g., "control", "variant-a", "variant-b")
- **Payloads**: Additional JSON data attached to flags for configuration

### Key Capabilities

- **Percentage rollouts**: Gradually release to 10%, 50%, 100% of users
- **Property targeting**: Target based on user properties (plan, location, etc.)
- **Deterministic assignment**: Users consistently get same variant via hashing
- **Local evaluation**: Cache flags locally to eliminate network latency

## Overview

Adding feature flag support with local evaluation to posthog-rs SDK ([PR #36](https://github.com/PostHog/posthog-rs/pull/36)).

## Why This Matters

- **Community need**: Most requested feature in [RFC #29](https://github.com/PostHog/posthog-rs/issues/29)
- **Performance**: Local evaluation eliminates network latency (sub-ms vs 50-200ms)
- **Reliability**: Works during network outages via caching
- **Feature parity**: Matches other PostHog SDKs

## Core Design

### Architecture

```
Application → Client → LocalEvaluator → FlagCache
                ↓                         ↑
            Remote API              FlagPoller (background)
```

### Key Components

1. **`feature_flags.rs`**: Core evaluation logic

   - Boolean and multivariate flags
   - Property matching (8 operators)
   - SHA1-based distribution

2. **`local_evaluation.rs`**: Caching layer

   - Thread-safe cache with `Arc<RwLock>`
   - Background polling (default: 30s)
   - Both sync and async support

3. **Client methods**: Simple API
   ```rust
   client.is_feature_enabled("flag", "user-id", properties)?
   client.get_feature_flag("flag", "user-id", properties)?
   client.get_feature_flags("user-id", properties)?
   ```

## Implementation Checklist

### Must Have (Week 1)

- [x] Core flag evaluation logic
- [x] Local caching with polling
- [x] Basic client integration
- [ ] Fix security warnings (API key in logs)
- [ ] Add error handling for network failures
- [ ] Basic documentation in README

### Nice to Have (If Time)

- [ ] Exponential backoff for failed requests
- [ ] Metrics for flag evaluations
- [ ] More comprehensive examples

## Key Decisions

### Following RFC Principles

✅ **Backwards compatible** - No existing APIs changed
✅ **Simple** - No complex architectures (queues, actors)
✅ **Minimal deps** - Only sha1, regex added
✅ **Parity** - Sync and async APIs match

### Trade-offs Made

- **In-memory only**: No disk persistence (simplicity)
- **Pull-based**: Polling instead of WebSockets (simpler)
- **No metrics**: Flag usage not auto-tracked (can add later)

## Usage Example

```rust
// Setup
let client = posthog_rs::client("api-key")
    .personal_api_key("personal-key")  // Required for local eval
    .enable_local_evaluation(true);

// Use
if client.is_feature_enabled("new-feature", "user-123", None)? {
    // Feature enabled
}
```

---

---

# Things we will do inorder to make it mergable

## Security Fixes Needed

1. **Remove sensitive data from logs**

   - Don't log API keys or user IDs in examples
   - Use debug/trace levels appropriately

2. **Validate API endpoints**
   - Ensure HTTPS only
   - Add certificate validation

## Testing Requirements

### Essential Tests

- [ ] Property matching operators work correctly
- [ ] Hash distribution is uniform
- [ ] Cache updates don't block reads
- [ ] Graceful degradation on network failure

---

---

### Quick Validation

```bash
# Run existing tests
cargo test --features async-client

# Test examples
cargo run --example feature_flags
cargo run --example local_evaluation
```

## Migration Impact

**Zero breaking changes**. Existing users unaffected:

```rust
// Old code still works
client.capture(event)?;

// New capability available
client.is_feature_enabled("flag", "user", None)?;
```

## Next Steps

### Immediate

1. Address security warnings from GitHub scan
2. Add network error recovery
3. Update README with feature flag section

### Before Merge

1. Add integration tests
2. Validate with real PostHog instance
3. Performance benchmark (local vs remote)

### Final review

1. Final review and cleanup
2. Update CHANGELOG
3. Ready for merge

## Risks & Mitigations

| Risk          | Impact | Mitigation                    |
| ------------- | ------ | ----------------------------- |
| API changes   | High   | Pin to v2 API, version checks |
| Memory leaks  | Medium | Bounded cache, proper cleanup |
| Thread panics | Low    | Catch and restart poller      |

## Success Criteria

- ✅ Tests pass
- ✅ No security warnings
- ✅ <1ms local evaluation
- ✅ Graceful network failure handling
- ✅ Clear documentation

## References

- [PR #36](https://github.com/PostHog/posthog-rs/pull/36)
- [RFC #29](https://github.com/PostHog/posthog-rs/issues/29)
- [PostHog Docs](https://posthog.com/docs/feature-flags)
