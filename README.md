# logging-nv

Structured logging for novo-lang: a record with typed fields, a filter,
a formatter family, and sinks behind one trait whose effect parameter
means a log call costs exactly what its destination costs.

**Status: NOT IMPLEMENTED — interface only.**  Every `pub fn` body is a
`todo()`, so the signatures, the effect rows and the tests are published
and nothing is implemented.  The first implementation is the `0.1.0`
published over this.

## What this is

A port of Rust's `log` facade and Python's `logging`, cut to the layer
design: a level, a target, a message and typed fields go in; a filter
decides; a formatter renders; a sink writes.

| module | holds | rows |
| --- | --- | --- |
| `lgrecord` | the level, the typed field value, the record | `[]` |
| `lgfilter` | the per-target rules and the specification grammar | `[]` |
| `lgformat` | human with colour, logfmt, JSON lines | `[]` |
| `lgsink` | **`LgSink[e]`**, the fault type, the null sink, the generic emits | `[e]` |
| `lgwrite` | the writer form, the stderr/stdout sink, the terminal question | `[e]`, `[io]` |
| `lgfile` | the file sink, and rotation as arithmetic | `[]`, `[fs]` |
| `lgring` | the last N records, as a value and as a sink | `[]`, `[mutate]`, `[e]` |
| `lgdeflog` | a device's decoded frames as records | `[]`, `[e]` |
| `lglog` | the logger facade, and the seam with `std.log` | `[]`, `[e]`, `[io]`, `[time]` |

## The load-bearing interface

```novo norun:pseudo
pub trait LgSink[e]
    fn emit(self, r: LgRecord) -> Result<Unit, LgSinkFault> [e]
    fn flush(self) -> Result<Unit, LgSinkFault> [e]
    fn accepts(self, l: LgLevel) -> Bool [e]
```

**`LgSink[e]` is the package.**  The effect parameter is what lets one
generic emit be written once and charged what the caller's own sink
costs — `[io]` over a terminal, `[fs]` over a rotating file, `[mutate]`
over a ring buffer, and **nothing at all** over the null sink or a
buffer a test drains.

The standard library's `Logger` shows what the alternative costs.  Its
`LogSink` is an enum, so — as [its own page](../../docs/stdlib/log.md)
says — `Logger.info` is charged the union over the variants, `[io]`,
"even when the installed sink is `SinkNull`".  A program that logs into
a buffer and asserts on the bytes is charged for a console it never
touches, and a caller with no `[io]` to give cannot log at all.

**The sink takes the record, not the line.**  The obvious surface —
`emit(self, line: Str)` — is wrong twice.  A ring buffer on a device
would have to format on the target, which is the entire cost the
deferred-logging story exists to avoid.  And a program writing to a
terminal and a collector would have to render the same record twice, in
two formats, in a caller that has no reason to know there are two.  So a
sink holds its own `LgFormat` and renders when and if it writes.

## The one example that will work

```novo
use lgfilter
use lgformat
use lglog
use lgrecord
use lgsink
use lgwrite

fn main() [io]
    let sink = lgwrite.stderr_sink(lgformat.human())
    let lg = lglog.filtered(lglog.logger("http.server"),
                            lgfilter.rule(lgfilter.filter(LgInfo), "http", LgDebug))

    match lglog.emit(lg, sink, LgInfo, "request served",
                     [lgrecord.field_int("status", 200),
                      lgrecord.field_float("seconds", 0.012)])
        Ok(written) => println("${written}")
        Err(e)      => println(e.message())
```

## Adding it, and checking it

```console
$ novo pkg add logging-nv
$ novo pkg build
$ novo test tests/lgsink_tests.nv
```

The suites are **red on purpose**: every body is a `todo()`, so every
assertion reaches `not implemented: logging-nv.<module>.<fn>`.  That is
what an interface release looks like from the outside, and it is how the
first implementation will know it is finished.

`novo pkg add` says `NOT IMPLEMENTED — interface only` on the way in,
because an interface resolves, downloads and builds exactly like an
implemented package and the difference only shows the first time
something calls it.

## The layer, and why

`host`, and the rows are not all the same — which is the disclosure
worth having.  Six of the nine modules declare **nothing**: the record,
the filter, the formatter family, the ring's arithmetic, the device
conversion and the logger's record builder are all `[]`.  What performs
is `lgwrite` (`[io]`), `lgfile` (`[fs]`), `lgring`'s sink impl
(`[mutate]`), `lglog.now` (`[time]`), and everything generic over
`LgSink[e]`, which costs what the caller's sink costs.

### The missing row

**A `core` package cannot reach `LgSink[e]`.**  A `core` package may
depend only on `core` packages, so the trait that was designed to let a
core library log at `[]` is, at this layer, out of its reach — the one
thing in this design that does not fit its box.

What would close it is a second row, `logging-core-nv` at `core`,
holding `lgrecord`, `lgfilter`, `lgformat`, the `LgSink[e]` declaration
and `lgring`'s value half — every one of which is already `[]` — with
this package keeping `lgwrite`, `lgfile`, `lgring`'s sink, `lgdeflog`
and `lglog`.  Nothing would have to be redesigned to make the split; the
line is already drawn in the table above.

The alternative, and the reason it was not taken here: `docs/publishing.md`
§ A package with a core and a host half lets one package declare
`layer = "core"` with `host_modules = ["lgwrite", "lgfile"]`.  That
would work, and it would move this package's cell on the Orbit map away
from where the must-have plan put it.  Which of the two is right is a
decision for the milestone review rather than for this lane.

## What `std.log` keeps

This package does not replace the standard library's logging and should
not be taken by a program the standard library already serves.

- **The global functions stay.**  `log.info("…")` reaches a working
  stderr with no value threaded anywhere.  A script, a tutorial example
  and a forty-line `main` all want that, and this package does not wrap
  them.
- **`std.log`'s `Logger` keeps the whole no-dependency case.**  It costs
  nothing to resolve, and it already has per-module levels with
  longest-prefix matching, text and JSON, five sinks and a pure
  `render_at`.
- **`render_at` keeps being the reference for the text format.**  The
  human format here pads the level tag to the same column nine, so a
  program migrating one subsystem at a time produces output that still
  lines up.
- **`lglog.to_std_log` is the seam**, so a program can start building
  records and filtering them properly while its output still goes
  through the standard library's own logger.

What this package adds, and why none of the four belongs in a standard
library every program links: the effect parameter on the sink; typed
field values, so the renderer rather than the call site decides how a
number is spelled; a rotation policy; and a bridge to a device decoder
the standard library has no reason to know about.

## The deferred-logging bridge

`lgdeflog` converts a `DeflogRecord` — one line a device sent as an
interned index and raw bytes, decoded on the host by `deflog-decoder` —
into an `LgRecord`, and `bridge_into` puts a batch of them into any
`LgSink[e]`.  A firmware's log and its host tool's log then go through
one filter, one format and one file.

Two honest costs, named here rather than discovered:

- **The closure.**  Depending on `deflog-decoder` and `deflog-parser`
  brings `elf-nv`, `leb128-nv`, `zigzag-nv` and `rzcobs-nv` with them,
  so a program that only wants a log line on stderr assembles seven
  packages.  A `deflog-sink-nv` adapter — depending on both and depended
  on by neither — is the alternative, and it is the milestone review's
  to weigh.
- **The conversion is lossy**, and `lossy_fields` says so per record: a
  128-bit argument, a byte string, a list and a nested user type arrive
  as text, and the interned index and the raw tick count do not arrive
  at all.

## What widened, and what did not

- **A fan-out over two arbitrary sinks cannot be written.**  A function
  binds exactly one effect parameter (`E3005`), so
  `fan_out<A: LgSink[e], B: LgSink[f]>` is refused with a diagnostic
  that says as much.  What is published instead is
  `lglog.emit_to_console_and_file`, where both sink types are named and
  the row is the concrete `[io, fs]` that pair costs.  Any other pair
  calls `emit_all` twice.
- **`LgFileSink` claims `[fs]` and not `[io, fs]`.**  The standard
  library's own `File` declares `impl Write[io, fs]`, so an
  implementation written the obvious way would inherit both.  `[fs]` is
  the honest claim for "append to a named file" and the implementation
  lane's job is to make it true; if it cannot, the row widens and this
  section says so rather than the claim quietly growing.
- **A struct with no fields is a parse error.**  `LgNullSink` carries a
  level floor because it had to carry something.  That turned out to
  improve it — `accepts` now has an answer a test can exercise — but the
  field was not the design's idea.

## Reference

The reference implementations are Rust's [`log`](https://docs.rs/log)
facade, [`tracing-subscriber`](https://docs.rs/tracing-subscriber)'s
filter grammar, and Python's
[`logging`](https://docs.python.org/3/library/logging.html).  The
specification-shaped pieces are their own: the logfmt convention, and
one JSON object per line.

## Licence

Apache-2.0.
