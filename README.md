# meta-tracing

[![crates.io](https://img.shields.io/crates/v/meta-tracing.svg)](https://crates.io/crates/meta-tracing)
[![docs.rs](https://docs.rs/meta-tracing/badge.svg)](https://docs.rs/meta-tracing)
[![License](https://img.shields.io/badge/license-MIT%2FApache--2.0-blue.svg)](#license)

A small companion to the [`tracing`](https://crates.io/crates/tracing) crate
that collects structured metadata as you go and hands you a single
JSON-serializable record at the end.

## Why

`tracing` is great at producing a stream of events for humans, log
aggregators, and OTel pipelines. But sometimes you also want a **single
structured object** describing what a unit of work did — sizes, timings,
named summary sections, warnings — to return alongside an API response,
attach to a job record, or write to S3.

`meta-tracing` runs alongside `tracing`. Every call also emits a span or
event, so your logs stay consistent; and at the end you call `.build()`
and get a `CollectedMeta` value you can serialize.

## At a glance

```text
MetaCollector  ──►  (sections + issues + timings)  ──►  CollectedMeta { JSON }
```

## Quick start

```toml
# Cargo.toml
[dependencies]
meta-tracing = "0.1"
serde        = { version = "1", features = ["derive"] }
serde_json   = "1"
```

```rust
use meta_tracing::MetaCollector;
use serde::Serialize;

#[derive(Serialize)]
struct LoadStats { input_rows: usize, output_rows: usize }

let mut meta = MetaCollector::new();
meta.set_input_rows(1_000);

// Timed section — opens a tracing span, captures elapsed_ms when finished.
{
    let section = meta.timed_section("load");
    // ... do work ...
    section.finish_with_data(&LoadStats { input_rows: 1_000, output_rows: 950 });
}

// Free-form warnings; prefix with [WARN] / [ERROR] to drive log levels.
meta.add_issue("[WARN] 50 rows dropped");
meta.set_output_rows(950);

let result = meta.build();
println!("{}", serde_json::to_string_pretty(&result).unwrap());
```

The output is a `CollectedMeta` with `sections`, `issues`, `input_rows`,
`output_rows`, and `processing_time_ms` — all skip-serialize-if-empty so
the JSON stays tight.

## What's in the box

- **`MetaCollector`** — the builder you hand around in your processing
  code.
- **Sections** (`add_section`, `merge_section`) — named JSON blobs from
  any `Serialize` type. `merge_section` deep-merges into an existing
  object, which is useful when multiple stages contribute to the same
  section (e.g. `"validation"`).
- **Issues** (`add_issue`, `add_issues`) — free-form strings; `[WARN]` /
  `[ERROR]` prefixes drive the corresponding `tracing` level, and
  `CollectedMeta::warning_count()` / `error_count()` count them.
- **Row tracking** (`set_input_rows`, `set_output_rows`, `set_rows`) —
  common enough to deserve dedicated fields.
- **Timed sections** (`timed_section` returns a `TimedSection` guard) —
  opens an INFO `meta_section` span and, on `finish*`, writes
  `{ elapsed_ms, data? }` into the section under that name.

Convenience free functions (`record_input_rows`, `record_section`,
`record_issue`, …) take `Option<&mut MetaCollector>` so optional
collection sites can stay one-liner.

## What this is *not*

- Not a tracing layer / subscriber. It calls into `tracing` but doesn't
  hook the dispatch — bring your own subscriber.
- Not opinionated about transport. `CollectedMeta` implements
  `Serialize` / `Deserialize`; do what you like with the JSON.
- Not a metrics library. If you need counters, histograms, and an
  exporter, look at `metrics` or `opentelemetry`.

## License

Dual-licensed under either of:

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE))
- MIT license ([LICENSE-MIT](LICENSE-MIT))

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally
submitted for inclusion in the work by you, as defined in the Apache-2.0
license, shall be dual-licensed as above, without any additional terms or
conditions.
