# Injective Core Development

## Build Commands
- **Go (injective-core)**: `make install` - builds and installs injectived and peggo binaries
- **Rust (swap-contract)**: `cargo build --release --target wasm32-unknown-unknown` - builds WASM contract

## Test Commands
- **All tests**: `make test` - runs unit and integration tests
- **Unit tests only**: `make test-unit` - runs Go unit tests with ginkgo
- **Single module test**: `make test-exchange` - tests exchange module specifically
- **Single Go test**: `ginkgo -r --race ./path/to/package` - run tests for specific package
- **Rust tests**: `cargo test` - run all Rust contract tests

## Lint Commands
- **Go lint**: `make lint` - runs golangci-lint on all code
- **Rust lint**: `cargo clippy --tests -- -D warnings` - runs clippy with warnings as errors
- **Rust format check**: `cargo fmt --all -- --check` - checks rustfmt formatting

## Code Style Guidelines
- **Go**: Line length ≤140 chars, cognitive complexity ≤20, use revive linter rules
- **Imports**: Standard library first, then third-party, then local packages (alphabetized)
- **Naming**: PascalCase for exported, camelCase for unexported, ALL_CAPS for constants
- **Error handling**: Use descriptive error messages, wrap errors with context
- **Rust**: Max width 150 chars, 4 spaces indentation, Unix newlines
- **Types**: Use strong typing, avoid `any`/`interface{}`, prefer specific types</content>
<parameter name="file_path">CRUSH.md