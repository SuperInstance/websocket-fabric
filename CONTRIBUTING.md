# Contributing to websocket-fabric

Thank you for your interest in contributing to websocket-fabric! This is a high-performance WebSocket library for Rust with connection pooling, reconnection, and backpressure handling.

## Table of Contents

- [Quick Start](#quick-start)
- [Development Environment Setup](#development-environment-setup)
- [Development Workflow](#development-workflow)
- [Code Standards](#code-standards)
- [Testing Guidelines](#testing-guidelines)
- [Documentation Standards](#documentation-standards)
- [Submitting Changes](#submitting-changes)
- [Getting Help](#getting-help)

---

## Quick Start

```bash
# Clone the repository
git clone https://github.com/SuperInstance/websocket-fabric.git
cd websocket-fabric

# Build the project
cargo build --workspace

# Run tests
cargo test --workspace

# Make your changes
# ... edit files ...

# Run tests and linting
cargo test --workspace
cargo clippy --workspace --all-targets -- -D warnings
cargo fmt --check

# Submit a pull request
```

---

## Development Environment Setup

### Prerequisites

**Required:**
- Rust 1.75+ (stable toolchain)
- Git 2.30+

**Recommended:**
- 4GB+ RAM
- 2+ CPU cores
- SSD storage

### Installing Rust

```bash
# Install Rust using rustup
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Configure stable toolchain
rustup default stable
rustup update

# Verify installation
rustc --version
cargo --version
```

### Repository Setup

```bash
# Clone with SSH (recommended if you have SSH keys configured)
git clone git@github.com:SuperInstance/websocket-fabric.git

# Or clone with HTTPS
git clone https://github.com/SuperInstance/websocket-fabric.git

# Enter the directory
cd websocket-fabric

# Build all crates in development mode
cargo build --workspace

# Run all tests to verify setup
cargo test --workspace
```

### IDE Configuration

**VS Code (Recommended):**
1. Install the **rust-analyzer** extension
2. Install the **CodeLLDB** extension for debugging
3. Configure workspace settings:
   ```json
   {
     "rust-analyzer.cargo.loadOutDirsFromCheck": true,
     "rust-analyzer.cargo.features": "all",
     "rust-analyzer.checkOnSave.command": "clippy"
   }
   ```

**IntelliJ IDEA / CLion:**
1. Install the **Rust** plugin
2. Enable rustfmt: Settings → Languages & Frameworks → Rust
3. Use cargo as build system

**Vim/Neovim:**
- Install `rust-analyzer` via your plugin manager
- Configure LSP with `nvim-lspconfig` or `coc-rust-analyzer`

---

## Development Workflow

### Finding Something to Work On

- Check [GitHub Issues](https://github.com/SuperInstance/websocket-fabric/issues) for open tasks
- Look for labels: `good first issue`, `help wanted`, `enhancement`, `bug`
- Comment on the issue to claim it (avoid duplicate work)

### Creating a Branch

```bash
# Ensure you're on main and up to date
git checkout main
git pull origin main

# Create a feature branch
git checkout -b feature/your-feature-name

# Or a bug fix branch
git checkout -b fix/issue-number-brief-description

# Branch naming conventions:
# feature/ - New features
# fix/ - Bug fixes
# docs/ - Documentation changes
# refactor/ - Code refactoring
# test/ - Test improvements
# perf/ - Performance improvements
```

### Making Changes

- Write code following our [Code Standards](#code-standards)
- Add tests for new functionality (see [Testing Guidelines](#testing-guidelines))
- Update documentation as needed
- Keep commits atomic and well-described

### Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
type(scope): description

[optional body]

[optional footer]
```

**Types:**
- `feat` - New feature
- `fix` - Bug fix
- `docs` - Documentation changes
- `style` - Code style changes (formatting, no logic change)
- `refactor` - Code refactoring
- `test` - Adding or updating tests
- `chore` - Maintenance tasks
- `perf` - Performance improvements

**Examples:**
```bash
git commit -m "feat(pool): implement connection pooling"
git commit -m "fix(reconnect): resolve infinite loop on connection failure"
git commit -m "docs(readme): update installation instructions"
```

---

## Code Standards

### Rust Code Style

**General Principles:**
- Follow [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)
- Use `rustfmt` for all formatting (default settings)
- Use `clippy` for linting (zero warnings)
- Prefer clear, readable code over clever code
- Use descriptive names (no single-letter variables except loop counters)

**Naming Conventions:**
```rust
// Modules: snake_case
mod connection_pool;
mod message_queue;

// Types: PascalCase
struct ConnectionPool;
struct MessageQueue;

// Functions: snake_case
fn connect_to_server(url: &str) -> Result<Connection, Error>;

// Constants: SCREAMING_SNAKE_CASE
const MAX_RETRIES: u32 = 3;
const DEFAULT_TIMEOUT: Duration = Duration::from_secs(30);

// Generic types: PascalCase, single letter (T, E, K, V)
fn process<T: Display>(value: T) -> String {
    value.to_string()
}
```

**Error Handling:**
```rust
// Use Result for recoverable errors
pub fn connect(url: &str) -> Result<Connection, Error> {
    // ...
}

// Use Option for optional values
pub fn get_connection(&self, id: &str) -> Option<&Connection> {
    // ...
}

// Use ? for error propagation
pub async fn send_message(&self, msg: Message) -> Result<(), Error> {
    let connection = self.get_connection()?;
    connection.send(msg).await?;
    Ok(())
}
```

**Thread Safety:**
```rust
// Use Arc for shared ownership across threads
use std::sync::Arc;

// Use tokio::sync::Mutex in async code (NOT std::sync::Mutex)
use tokio::sync::Mutex;

// Example: Shared state in async context
#[derive(Clone)]
pub struct SharedState {
    inner: Arc<Mutex<StateData>>,
}

// Use AtomicBool for simple boolean flags
use std::sync::atomic::{AtomicBool, Ordering};

pub struct Connection {
    is_connected: Arc<AtomicBool>,
}
```

**Async/Await Best Practices:**
```rust
// NEVER hold MutexGuard across await points
// ❌ WRONG
let lock = mutex.lock().await;
async_function().await; // DEADLOCK!
drop(lock);

// ✅ CORRECT
let lock = mutex.lock().await;
let result = sync_operation(&lock);
drop(lock); // Release lock before await
async_function().await; // Safe now
```

### Documentation

**Public APIs:** Every public item must have documentation:

```rust
/// Creates a new WebSocket connection with automatic reconnection.
///
/// # Arguments
///
/// * `url` - The WebSocket URL to connect to
/// * `options` - Connection configuration options
///
/// # Returns
///
/// A `Connection` that can be used to send and receive messages
///
/// # Errors
///
/// Returns `Error::InvalidUrl` if the URL is not a valid WebSocket URL
///
/// # Examples
///
/// ```
/// use websocket_fabric::{Connection, ConnectionOptions};
///
/// # tokio_test::block_on(async {
/// let options = ConnectionOptions::default();
/// let connection = Connection::new("ws://localhost:8080", options).await?;
/// # Ok::<(), websocket_fabric::Error>(())
/// });
/// ```
pub async fn connect(url: &str, options: ConnectionOptions) -> Result<Connection, Error> {
    // implementation
}
```

---

## Testing Guidelines

### Test Organization

**Unit Tests:**
- Place in the same file as the code
- Use `#[cfg(test)]` module
- Test public APIs and private helpers
- Use descriptive test names

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_connection_url_parsing() {
        let url = "ws://localhost:8080";
        let result = parse_url(url);
        assert!(result.is_ok());
    }

    #[tokio::test]
    async fn test_connection_establishment() {
        let connection = connect_to_test_server().await;
        assert!(connection.is_connected());
    }
}
```

**Integration Tests:**
- Place in `tests/` directory
- Test component interactions
- Use `tokio::test` for async tests

```rust
// tests/integration/full_connection_flow.rs
#[tokio::test]
async fn test_full_connection_flow_with_reconnect() {
    // Setup
    let pool = ConnectionPool::new().await;

    // Execute
    let connection = pool.connect("ws://localhost:8080").await.unwrap();

    // Verify
    assert!(connection.is_connected());
}
```

### Test Coverage Goals

**Critical Areas (100% coverage required):**
- Connection establishment logic
- Reconnection mechanisms
- Message serialization/deserialization
- Error handling paths
- Security-critical code

**High Coverage (80%+):**
- Connection pool management
- Backpressure handling
- Heartbeat/ping-pong logic

**Standard Coverage (60%+):**
- Utility functions
- Configuration parsing
- Logging/tracing code

### Running Tests

```bash
# Run all tests
cargo test --workspace

# Run tests in parallel (faster)
cargo test --workspace -- --test-threads=8

# Run tests with output
cargo test --workspace -- --nocapture

# Run tests with logging
RUST_LOG=debug cargo test --workspace

# Run specific test
cargo test test_connection_url_parsing

# Run tests for specific package
cargo test --package websocket_fabric

# Run benchmarks
cargo bench
```

---

## Documentation Standards

### Code Documentation

**Public APIs:** Every public item must have documentation:
- Functions: What it does, parameters, return value, errors, examples
- Structs: Purpose, field descriptions, usage examples
- Enums: Variants, when to use each
- Traits: Purpose, required methods, implementor notes

### Project Documentation

**When to Update Documentation:**
- User-facing features → Update `README.md`
- Architectural changes → Update `ARCHITECTURE.md` (if exists)
- API changes → Update crate docs
- New development patterns → Update `CONTRIBUTING.md`

---

## Submitting Changes

### Before Submitting

- Ensure all tests pass: `cargo test --workspace`
- Ensure no warnings: `cargo clippy --workspace --all-targets -- -D warnings`
- Ensure formatting is correct: `cargo fmt --check`
- Update documentation if needed
- Self-review your changes

### Creating the Pull Request

1. **Push your branch**:
   ```bash
   git push origin feature/your-feature-name
   ```

2. **Go to GitHub** and create a PR from your branch

3. **Fill out the PR template**:
   ```markdown
   ## Description
   Brief description of changes made.

   ## Type of Change
   - [ ] Bug fix
   - [ ] New feature
   - [ ] Breaking change
   - [ ] Documentation update
   - [ ] Performance improvement

   ## Testing
   - [ ] Tests added/updated
   - [ ] All tests passing locally
   - [ ] Benchmarks run (if applicable)

   ## Checklist
   - [ ] Code follows style guidelines
   - [ ] Self-review performed
   - [ ] Documentation updated
   - [ ] No new warnings generated
   - [ ] Commits follow conventional commits
   ```

4. **Link related issues** (e.g., "Fixes #123")

### During Review

- Respond to feedback promptly
- Make requested changes
- Push updates to the same branch
- Ask questions if anything is unclear

---

## Getting Help

### Documentation

- [README.md](README.md) - Project overview and quick start
- [API Documentation](https://docs.rs/websocket-fabric) - Rust API docs

### Community Resources

- **GitHub Issues**: Bug reports and feature requests
- **GitHub Discussions**: Questions and ideas
- **Email**: dev@superinstance.ai

### Common Issues

**Build Errors:**
```bash
# Update Rust toolchain
rustup update stable

# Clean build artifacts
cargo clean

# Rebuild
cargo build --workspace
```

**Test Failures:**
```bash
# Run tests with output to see what's failing
cargo test --workspace -- --nocapture

# Run specific failing test
cargo test test_name

# Run with logging
RUST_LOG=debug cargo test --workspace
```

---

## Performance Guidelines

When making performance-related changes:

1. **Benchmark Before**: Establish a baseline
2. **Make Changes**: Implement your optimization
3. **Benchmark After**: Compare to baseline
4. **Document**: Explain the performance impact

```bash
# Run benchmarks
cargo bench

# Compare with baseline
cargo bench -- --baseline main
```

---

## License

By contributing to websocket-fabric, you agree that your contributions will be licensed under the **MIT** license.

---

## Recognition

We value all contributions! Contributors will be:

- Listed in `CONTRIBUTORS.md` (if applicable)
- Mentioned in release notes for significant contributions

Thank you for contributing to websocket-fabric!

---

*Last Updated: 2026-01-10*
