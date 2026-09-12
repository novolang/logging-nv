# Changelog

All notable changes to logging-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `lgrecord` — `LgLevel` with six levels, `LgValue` with five typed
  arms, `LgField` and `LgRecord`, plus the reserved-and-duplicate key
  check the JSON format needs.
- `lgfilter` — per-target rules matched by longest prefix, segment
  aware, with the `info,http=error` specification grammar, its faults,
  a round trip back to text, and the cheap global gate a hot loop
  hoists.
- `lgformat` — human with ansi-nv colour, logfmt with its quoting rule
  published as a predicate, and one JSON object per line, all `[]`, with
  a writer form that appends into a buffer the caller owns.
- `lgsink` — `LgSink[e]`, `LgSinkFault`, the null sink at `[]`, and the
  generic emits that cost what the caller's sink costs.
- `lgwrite` — `write_record<W: Write[e]>`, the stderr and stdout sink at
  `[io]`, and the terminal and colour-depth questions.
- `lgfile` — a rotating file sink at `[fs]` with every naming and
  keeping decision published as `[]` arithmetic.
- `lgring` — the last N records as a pure value and as a `[mutate]`
  sink, with `dump_into` at `[e]`.
- `lgdeflog` — a `DeflogRecord` as an `LgRecord`, the tick base, and
  what the conversion drops.
- `lglog` — the logger facade, the `std.log` seam, and the concrete
  console-and-file fan-out.

### Known

- **`LgSink[e]` is the load-bearing interface**, and the effect
  parameter is the whole argument: one generic emit, charged what the
  destination costs, where an enum sink charges every caller the union.
- **A `core` package cannot reach it**, because a `core` package may
  depend only on `core` packages. The README's § The missing row names
  the `logging-core-nv` split that would close it and says exactly which
  modules move.
- **A fan-out over two arbitrary sinks cannot be written**: a function
  binds exactly one effect parameter. `lglog.emit_to_console_and_file`
  is the concrete pair.
- **`LgFileSink` claims `[fs]` and not `[io, fs]`**, which is the honest
  claim rather than what the standard library's `File` would supply.
- **Seven packages in the closure**, four of them brought by the device
  bridge; a `deflog-sink-nv` adapter is the alternative.
- **No device claim.** `Str`, `Result` and the record's list of fields
  do not link at `@tier(embedded)`, so there is no probe.
