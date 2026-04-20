<div align="center">

<img src="assets/laminae-git.png" alt="Laminae" width="720" />

# Laminae

**Six composable Rust layers between raw LLMs and production AI. Personality. Safety. Red-teaming. Containment.**

[![Crates.io](https://img.shields.io/crates/v/laminae.svg)](https://crates.io/crates/laminae)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/Rust-stable-000?logo=rust)](https://www.rust-lang.org)
[![Docs](https://docs.rs/laminae/badge.svg)](https://docs.rs/laminae)

</div>

> [!NOTE]
> **Open for contributions.** Laminae is mature at **v0.4.2** (11 crates published to crates.io), but a few meaningful gaps remain (Linux seccomp sandbox, Python PyPI packaging, docs site deploy). See [Help Wanted](#help-wanted).

## What is Laminae?

Laminae is a Rust SDK that sits between your app and a raw LLM call. It's split into six composable layers; you pick the ones you need.

- **Persona** - extract voice from weighted samples; enforce it with a 6-detector filter (AI vocabulary, meta-commentary, trailing questions, em-dashes, length, paragraph structure)
- **Cortex** - learn from user edits; detect 8 pattern types; produce a "learned instructions" prompt block
- **Psyche** - multi-agent cognitive pipeline (Id / Superego / Ego) with auto-tiering (Skip / Light / Full); streaming events
- **Shadow** - red-team analyzer. Static (SQLi, XSS, path traversal, weak crypto) + secrets (10 patterns incl. GitHub/AWS/JWT) + dependency (pipe-to-shell, compromised packages) + LLM reviewer, async pipeline
- **Ironclad** - process sandbox. `sandbox-exec` on macOS (full), namespaces on Linux, Job Objects on Windows; CPU/memory watchdog with SIGKILL
- **Glassbox** - deterministic I/O containment. Prompt-injection detection (~400ns), system-prompt-leak detection, command blocklists, immutable-zone write protection, symlink-bypass-resistant, Unicode NFKC normalization, rate limiter

First-class backends: **Claude** (`laminae-anthropic`), **OpenAI + compatible** (`laminae-openai` with built-in `groq()`, `together()`, `deepseek()`, `local()`), **Ollama** (`laminae-ollama`).

## Install

```toml
# Cargo.toml
[dependencies]
laminae = "0.4.2"                 # umbrella
# or pick individual layers:
# laminae-glassbox = "0.4.2"
# laminae-persona  = "0.4.2"
# laminae-psyche   = "0.4.2"
# laminae-shadow   = "0.4.2"
# laminae-ironclad = "0.4.2"
# laminae-cortex   = "0.4.2"
```

## Quick examples

```rust
use laminae::glassbox::Glassbox;

let guard = Glassbox::new();
guard.validate_input("Ignore previous instructions...")?; // returns Err on injection
```

```rust
use laminae::psyche::PsycheEngine;

let engine = PsycheEngine::new(ego_backend).with_tiering();
let response = engine.reply("Write a SQL query for...").await?;
// streaming variant: engine.reply_streaming(...) -> Receiver<PsycheEvent>
```

```rust
use laminae::shadow::ShadowEngine;

let shadow = ShadowEngine::new();
let mut events = shadow.analyze_async(code).await;
while let Some(event) = events.recv().await {
    // ShadowEvent::Finding / ShadowEvent::Done
}
```

Full examples + recipes live in `docs/` (mdbook; not yet deployed to the web - see [Help Wanted](#help-wanted) task #4).

## Architecture

Workspace at root; 11 crates in tiered order:

- **Tier 1** (leaf/foundational): `laminae-glassbox`, `laminae-persona`, `laminae-cortex`, `laminae-ollama`
- **Tier 2**: `laminae-ironclad` (uses glassbox), `laminae-psyche` (uses ollama)
- **Tier 3**: `laminae-shadow` (uses psyche + glassbox)
- **Tier 4** (provider SDKs): `laminae-anthropic`, `laminae-openai` - both impl the `EgoBackend` trait
- **Tier 5** (umbrella): `laminae` re-exports everything
- **Tier 6**: `laminae-python` (PyO3 bindings to Glassbox, VoiceFilter, Cortex)

Each layer engine returns either `anyhow::Result<T>` or a `tokio::sync::mpsc::Receiver<Event>` for streaming. Trait-based extension points: `EgoBackend` (LLMs), `Analyzer` (Shadow), `GlassboxLogger` (custom logging).

**WASM support**: Glassbox, Persona, Cortex work on WASM (no native OS calls). Psyche, Shadow, Ironclad, Ollama need native runtime.

## Build / test

```bash
# whole workspace
cargo build --workspace
cargo test --workspace --exclude laminae-python

# benches (HTML reports in target/criterion/)
cargo bench --workspace

# Python bindings (requires maturin)
cd crates/laminae-python
maturin develop
```

MSRV: Rust 1.83. CI runs test + doc-test on every push.

## Publishing

Release workflow triggers on `v*` tags, runs tests, then `cargo publish` in dependency order (30s waits between tiers). Requires `CARGO_REGISTRY_TOKEN` in repo secrets.

## Help Wanted

Concrete, scoped tasks:

1. **Python PyPI packaging** (medium, ~2-3h) - `crates/laminae-python/README.md` says "PyPI package coming soon." Add `.github/workflows/publish-python.yml` that builds maturin wheels on `v*` tags and uploads to PyPI via twine. Unlocks Laminae for Python users.
2. **Linux seccomp-bpf / landlock sandbox** (hard, ~8-10h) - Ironclad on Linux uses `unshare` namespaces only today. Add the `seccomp` or `landlock` crate for syscall / LSM filtering. `crates/laminae-ironclad/README.md:79` tracks this.
3. **Windows filesystem / network isolation** (hard, ~6-8h) - Job Objects give resource limits today, no filesystem or network isolation. Add filter-driver or Job-Object-level restrictions. `crates/laminae-ironclad/README.md:81`.
4. **mdbook docs deploy** (easy, ~1-2h) - `docs/` has 40+ structured pages but no CI to publish them. Add `.github/workflows/docs.yml` that builds mdbook and deploys to `gh-pages`.
5. **Shadow container-backed execution** (medium, ~4-5h) - `docs/src/layers/shadow.md:11` mentions "ephemeral container testing (optional)" but the implementation isn't wired. Add a `sandbox_execute()` path that runs findings inside Docker/Podman.
6. **Benchmark tracking in CI** (easy, ~1-2h) - `BENCHMARKS.md` has concrete M4 Max numbers but CI doesn't publish results. Add `cargo bench --workspace` to CI, publish results to a GitHub Pages artifact to track regressions.

Good first PRs: #4 and #6.

## Contributing

Read [CONTRIBUTING.md](CONTRIBUTING.md).

- Semver is strict. Bump correctly.
- `cargo fmt`, `cargo clippy -- -D warnings`, `cargo test --workspace` all green
- New features go in the right tier; prefer adding a new crate over widening an existing one
- Add benchmarks for anything on a hot path (Glassbox input validation, Persona filter)

## License

[MIT](LICENSE). Copyright 2026 Orel Ohayon.
