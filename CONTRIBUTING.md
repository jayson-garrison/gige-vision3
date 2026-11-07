# Contributing

## Definition of Done

A change is done only when all of these hold:

1. `cargo fmt --check` passes.
2. `cargo clippy --all-targets -- -D warnings` passes with no warnings.
3. `cargo build` and `cargo test` pass; new behaviour has tests.
4. CI is green on ubuntu-latest and macos-latest (stable Rust).
5. The pull request has been reviewed and approved by QA (Rick).
6. Nothing is pushed directly to `main`. All changes go through a pull request from a feature branch.
