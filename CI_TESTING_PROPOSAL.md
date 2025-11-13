# CI Testing: Implementation Proposal

## Problem Statement
Current CI workflow only tests:
- Default features during unit tests
- Only `e2e-test` feature for E2E tests
- Clippy linting on default features only

**Risk:** Feature combinations may fail to compile or have lint warnings without detection, leading to production issues.

## Current Features
From [Cargo.toml](https://github.com/fern-demo/posthog-rs/blob/19ba132ffd3d886aa30e10f2d8293442f1c3b028/Cargo.toml#L27):
- `default = ["async-client"]`
- `async-client` - Asynchronous client implementation
- `e2e-test` - End-to-end testing flag

## Proposed Solutions

We have two implementation approaches, each with distinct trade-offs:

### Approach 1: Sequential Testing (Simple & Direct)
**Implementation:** Direct update to [.github/workflows/pr.yaml](https://github.com/fern-demo/posthog-rs/blob/main/.github/workflows/pr.yaml)

**Status:** 📝 Proposed

#### What Changed:
1. **Builds** - Test all feature combinations:
   - Default features
   - No default features
   - All features
   - Only async-client

2. **Unit Tests** - Run against all feature combinations:
   - Default features
   - No default features
   - All features
   - Only async-client

3. **Clippy Linting** - Verify all feature combinations:
   - Default features
   - No default features
   - All features
   - Only async-client

#### Full Implementation:
```yaml
name: Test PR

on: [pull_request]

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Rust
      run: |
        rustup update stable
        rustup default stable
        rustup component add rustfmt clippy

    - name: Print Rust version
      run: rustc --version

    # Build all feature combinations
    - name: Build (default features)
      run: cargo build --verbose

    - name: Build (no default features)
      run: cargo build --verbose --no-default-features

    - name: Build (all features)
      run: cargo build --verbose --all-features

    - name: Build (only async-client)
      run: cargo build --verbose --no-default-features --features async-client

    # Test all feature combinations
    - name: Unit test (default features)
      run: cargo test --verbose

    - name: Unit test (no default features)
      run: cargo test --verbose --no-default-features

    - name: Unit test (all features)
      run: cargo test --verbose --all-features

    - name: Unit test (only async-client)
      run: cargo test --verbose --no-default-features --features async-client

    # E2E tests
    - name: E2E test
      run: cargo test --verbose --features e2e-test --no-default-features
      env:
        POSTHOG_RS_E2E_TEST_API_KEY: "${{ secrets.POSTHOG_RS_E2E_TEST_API_KEY }}"

    # Linting
    - name: Check formatting
      run: cargo fmt -- --check

    - name: Clippy (default features)
      run: cargo clippy -- -D warnings

    - name: Clippy (all features)
      run: cargo clippy --all-features -- -D warnings

    - name: Clippy (no default features)
      run: cargo clippy --no-default-features -- -D warnings

    - name: Clippy (only async-client)
      run: cargo clippy --no-default-features --features async-client -- -D warnings
```

#### Pros:
- Simple, easy to understand
- Minimal changes to existing workflow
- Clear step-by-step execution
- Easy to debug - sequential output
- No learning curve for team members

#### Cons:
- Slower - runs sequentially (~15-20 minutes estimated)
- Harder to maintain as features grow
- Verbose YAML with repetitive steps
- First failure stops entire pipeline
- Cannot easily see which specific feature combo failed at a glance

---

### Approach 2: Matrix-Based Parallel Testing (Modern & Scalable)
**Implementation:** New workflow structure using GitHub Actions matrix strategy

**Status:** 📝 Proposed (example available at [.github/workflows/pr-matrix.yaml.example](.github/workflows/pr-matrix.yaml.example))

#### Architecture:
Three separate parallel jobs:
1. **Matrix Test Job** - Build & test all feature combinations in parallel
2. **E2E Test Job** - Isolated E2E testing
3. **Linting Job** - Format and clippy checks

#### Full Implementation:
```yaml
name: Test PR

on: [pull_request]

jobs:
  # Matrix-based parallel feature testing
  test:
    name: Test (${{ matrix.name }})
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        include:
          - name: "default features"
            flags: ""
          - name: "no default features"
            flags: "--no-default-features"
          - name: "all features"
            flags: "--all-features"
          - name: "only async-client"
            flags: "--no-default-features --features async-client"

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Rust
      run: |
        rustup update stable
        rustup default stable

    - name: Print Rust version
      run: rustc --version

    - name: Build
      run: cargo build --verbose ${{ matrix.flags }}

    - name: Unit test
      run: cargo test --verbose ${{ matrix.flags }}

  # Isolated E2E testing
  e2e-test:
    name: E2E Test
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Rust
      run: |
        rustup update stable
        rustup default stable

    - name: E2E test
      run: cargo test --verbose --features e2e-test --no-default-features
      env:
        POSTHOG_RS_E2E_TEST_API_KEY: "${{ secrets.POSTHOG_RS_E2E_TEST_API_KEY }}"

  # Comprehensive linting
  linting:
    name: Linting
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        include:
          - name: "default features"
            flags: ""
          - name: "all features"
            flags: "--all-features"
          - name: "no default features"
            flags: "--no-default-features"
          - name: "only async-client"
            flags: "--no-default-features --features async-client"

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Rust
      run: |
        rustup update stable
        rustup default stable
        rustup component add rustfmt clippy

    # Only run formatting once (not in matrix)
    - name: Check formatting
      if: matrix.name == 'default features'
      run: cargo fmt -- --check

    - name: Clippy (${{ matrix.name }})
      run: cargo clippy ${{ matrix.flags }} -- -D warnings
```

#### Pros:
- **40-60% faster** - Parallel execution (~6-8 minutes estimated)
- Highly maintainable - Add features by adding one line to matrix
- Clear separation of concerns (test/e2e/lint as separate jobs)
- `fail-fast: false` shows all failures, not just the first
- GitHub UI shows clean matrix visualization
- Easy to identify which feature combo failed
- Industry best practice for multi-configuration testing

#### Cons:
- Slightly more complex YAML structure
- Uses more CI minutes concurrently (may hit limits on free tier)
- Requires understanding of GitHub Actions matrix strategy
- More jobs to monitor in CI dashboard

---

## Comparison Table

| Aspect | Sequential (Approach 1) | Matrix (Approach 2) |
|--------|------------------------|---------------------|
| **Execution Time** | ~15-20 min | ~6-8 min |
| **Maintainability** | Low - repetitive | High - DRY principle |
| **Readability** | High - explicit | Medium - requires matrix knowledge |
| **Failure Visibility** | Shows first failure only | Shows all failures |
| **Scalability** | Poor - grows linearly | Excellent - grows logarithmically |
| **CI Minutes Used** | Lower total | Higher concurrent usage |
| **Team Onboarding** | Easy | Requires brief explanation |


## References
- Current Implementation: [.github/workflows/pr.yaml](https://github.com/fern-demo/posthog-rs/blob/main/.github/workflows/pr.yaml)
- Features Definition: [Cargo.toml](https://github.com/fern-demo/posthog-rs/blob/19ba132ffd3d886aa30e10f2d8293442f1c3b028/Cargo.toml#L27)
