<p align="center">
  <img src=".github/icon.png" width="120" alt="Orellius Labs" />
</p>

<h3 align="center">Orellius Cognitive</h3>
<p align="center">AI safety and personality SDK — psyche, red-teaming, sandboxing in Rust.</p>

<p align="center">
  <a href="https://orellius.ai">Website</a> · 
  <a href="https://github.com/OrelliusAI">GitHub</a>
</p>

---

## What it does

Rust SDK for building AI agents with structured personality, safety guardrails, and containment. Six architectural layers handle everything from personality modeling to process-level sandboxing. Includes Python bindings via PyO3.

## Install

```bash
cargo add orellius-cognitive
```

## Quick start

```rust
use orellius_cognitive::{Psyche, Persona, IroncladSandbox};

let psyche = Psyche::builder()
    .superego("helpful, cautious, precise")
    .ego("technical writer")
    .build()?;

let sandbox = IroncladSandbox::new()
    .allow_network(false)
    .allow_fs_read(&["/data"])
    .spawn()?;
```

## Features

- **Psyche**: Freudian triple-agent pipeline (superego, ego, id) for personality consistency
- **Persona**: Voice extraction and style transfer from reference text
- **Cortex**: Instruction learning and few-shot adaptation
- **Shadow**: Red-teaming engine for adversarial testing
- **Ironclad**: Process-level sandboxing with capability restrictions
- **Glassbox**: I/O containment with full audit trail

## Tech stack

Rust, tokio, PyO3 (Python bindings), criterion benchmarks

## License

MIT — [Orellius Labs](https://orellius.ai)
