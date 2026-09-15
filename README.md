# logging-nv

**Structured logging** records an event as a value with named fields rather than
as a sentence, so that a program reading the log can find a field by name. The
convention was made ordinary by Rust's [`log`](https://docs.rs/log) facade and
[`tracing-subscriber`](https://docs.rs/tracing-subscriber), and by Python's
[`logging`](https://docs.python.org/3/library/logging.html). This package brings
it to novo-lang, with the destinations that write: a standard stream, a rotating
file, a ring buffer in memory, and a bridge that turns a microcontroller's
deferred log into the same records.
[logging-core-nv](https://novo-lang.org/packages/logging-core-nv) publishes the
part that performs no input or output, for a library that must declare an empty
effect list.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What it is

A **record** is one thing that happened. It carries a level, a target, a
message, a time, a list of fields and an optional source location.

A **level** says how loud a record is. There are six, from `LgTrace` to
`LgError`, with `LgOff` above them as a threshold that silences everything.

A **target** is the dotted name of the subsystem that emitted the record, such
as `http.server`. The caller chooses it, and a filter matches on it.

A **field** is one key and one value. A field value keeps its type: text, whole
number, real number, true or false, or present-and-empty. A consumer that reads
`status` gets a number on every line.

A **filter** decides whether a record is worth writing, before it is rendered.
It holds a default level and a list of rules, each a target prefix and the level
that prefix governs.

A **format** decides how a record is written. The human format is for a person
reading a terminal. The logfmt format writes `key=value` pairs on one line. The
JSON lines format writes one JSON object per line, for a collector.

A **sink** is where a record goes. `LgSink[e]` is a trait with one effect
parameter. Each implementation supplies the effects its own writing costs, so a
function that emits through a sink is charged exactly that and no more. Writing
to a terminal costs `[io]`. Writing to a file costs `[fs]`. Pushing into a ring
buffer costs `[mutate]`. The sink that keeps nothing costs nothing.

A **logger** binds a target, a filter, a format, a theme and a list of sticky
fields that go on every line it builds. It resolves the filter's answer for its
target once, so the per-line check compares two numbers.

**Rotation** is starting a new file so that a program logging for a year does
not fill a disk. A policy says when: never, past a size, at a day boundary, or
both.

A **deferred log** is the shape a microcontroller uses. The device sends an
interned format-string index and raw argument bytes, and a host decodes them.
This package converts one of those decoded records into an `LgRecord`, so a
firmware's log and its host tool's log go through one filter and one file.

## Install

```
novo pkg add logging-nv
```

## Example

```novo
use lgfilter
use lgformat
use lglog
use lgrecord
use lgwrite

fn main() [io]
    // A sink over stderr, rendering each record in the human format.
    let console = lgwrite.stderr_sink(lgformat.human())

    // A logger for one subsystem, under a filter that raises http to debug.
    let lg = lglog.filtered(lglog.logger("http.server"),
                            lgfilter.rule(lgfilter.filter(LgInfo), "http", LgDebug))

    // Write one record through the sink. The call costs what the sink costs,
    // and the sink renders the record through the format it holds.
    match lglog.emit(lg, console, LgInfo, "request served",
                     [lgrecord.field_int("status", 200),
                      lgrecord.field_float("seconds", 0.012)])
        Ok(written) => println("${written}")
        Err(e)      => println(e.message())
```

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails on
purpose: every test reaches a `not implemented` panic.

## What the package contains

| Module | Contents |
| --- | --- |
| `lgrecord` | The record and its parts: the six levels, the typed field value, the field, the record, and the check for reserved and duplicate keys. |
| `lgfilter` | The filter: a default level, prefix rules, the longest-prefix lookup, the one-line specification grammar, and the report of unreachable rules. |
| `lgformat` | The three renderings: human with colour, logfmt, and JSON lines. Each writes into a string or into a buffer the caller owns. |
| `lgsink` | The `LgSink[e]` trait, the fault type, the sink that keeps nothing, and the generic emits that cost whatever the caller's sink costs. |
| `lgwrite` | The sinks over stderr and stdout, the writer form that takes any `Write`, and the two questions this package asks the machine. |
| `lgfile` | The rotating file sink, and the naming and keeping decisions as plain arithmetic over a size and a day number. |
| `lgring` | The last N records, as a value the caller threads and as a sink behind a slot the caller owns. |
| `lgdeflog` | A decoded device record as an `LgRecord`: the level mapping, the tick-to-second conversion, and what the conversion drops. |
| `lglog` | The logger: its target, filter, format and sticky fields, the emit calls, the clock reading, and the bridge to `std.log`. |

## How to choose an entry point

**A program uses `lglog`.** Build a logger once at startup, then call
`lglog.emit` with the sink the program owns. `lglog.build` answers the record a
logger would emit without writing anything, which is how a test checks a
logger's own behaviour.

**A library takes the sink as a bound type parameter and calls
`lgsink.emit_if`.** The function then costs whatever the caller's sink costs.

```novo ignore
fn served<S: lgsink.LgSink[e]>(to: S, f: LgFilter, status: Int) -> Result<Bool, LgSinkFault> [e]
    lgsink.emit_if(to, f, lgrecord.record(LgInfo, "http.server", "request served"))
```

**A library that writes to a destination the caller supplies calls
`lgwrite.write_record`.** It is generic over the standard library's `Write`, so
an in-memory buffer costs nothing and a file costs the file's effects. A test of
a library's log output is then an ordinary assertion on bytes.

**A program chooses a sink by where the output goes.** `lgwrite.stderr_sink` for
a terminal. `lgwrite.stdout_sink` only when the program's standard output *is*
the log. `lgfile.open` for a file that rotates. `lgring.ring_sink` for the last
N records held in memory. `lgsink.null_sink` for a test.

**A host tool for a device calls `lgdeflog.bridge_into`.** It takes a batch of
decoded device records and puts them into any sink, through the same filter as
everything else.

## The rules a user needs

1. **A higher level number is louder.** `LgTrace` is 0 and `LgOff` is 5. A
   threshold admits what is at least as loud as itself. Call
   `lgrecord.level_at_least` rather than writing the comparison.
2. **There are six levels, where `std.log` has five.** `LgTrace` is the extra
   one. A device trace line therefore arrives as a trace line, and a filter set
   to debug-but-not-trace can be expressed.
3. **`LgOff` is a threshold and never a record's level.** A record built at
   `LgOff` is filtered out by every filter.
4. **A filter's longest matching prefix wins, and prefixes match whole
   segments.** `"ml"` governs `ml` and `ml.core`. It does not govern `mlx`.
   Rule order in a specification is therefore not a hidden meaning.
5. **The specification grammar is `env_logger`'s.** A bare level sets the
   default. Comma-separated `target=level` clauses add rules. See the `RUST_LOG`
   section of [`env_logger`](https://docs.rs/env_logger)'s documentation.
6. **A sink fault is not a program error.** `emit` answers a `Result` so that a
   caller can look at it. Propagating it with `!` would make a failed log line
   abort the request it was describing.
7. **A sink renders the record itself, through the format it holds.** Nothing
   hands a sink a finished line. One record can therefore reach a terminal in
   colour and a collector as JSON.
8. **`level`, `logger`, `msg` and `ts` are reserved in the JSON format.** A
   caller's field with one of those keys appears twice in the object, and this
   package does not rename it. `lgrecord.reserved_or_duplicate_keys` reports it.
9. **The JSON line's key set is fixed. Its key order is not.** A JSON object is
   unordered, so a consumer must read by name. See
   [JSON Lines](https://jsonlines.org/).
10. **logfmt quotes a value containing a space, an `=`, a quote or a control
    byte, and leaves every other value bare.**
    `lgformat.logfmt_needs_quoting` is the rule a consumer's parser has to agree
    with. See the [logfmt convention](https://brandur.org/logfmt).
11. **The default stream is stderr.** A log line on standard output corrupts the
    output of any program whose standard output is data. Use
    `lgwrite.stdout_sink` only for a program whose output is the log itself.
12. **The clock is read in one place.** `lglog.now` is the only call here that
    reads it. `lglog.emit` takes no time. `lglog.emit_at` takes the value, so
    the clock's effect appears at the call site that read it.
13. **`lgfile.append` takes the day the caller is in.** Rotation reads no clock,
    which is what makes a month boundary testable. `lgfile.day_of` turns a Unix
    second into that day number.
14. **Rotation names and keeps by arithmetic that performs nothing.**
    `lgfile.should_rotate`, `.rotated_name`, `.dated_name` and `.retired_names`
    take numbers and answer with names. Only `open`, `append`, `rotate_now` and
    `retire` touch the filesystem.
15. **A file sink reports a failed write.** It does not swallow one, so a disk
    that filled at 03:00 does not look like a quiet night.
16. **Colour is asked once, by the party that owns the descriptor.**
    `lgwrite.is_terminal` and `lgwrite.colour_depth` are the two questions. The
    answers go into `LgHumanStyle`, and every render stays free of effects.
    `colour_depth` also honours `NO_COLOR` and a `TERM` of `dumb`.
17. **A full ring drops the oldest record and counts it.** The ring sink answers
    `Ok` even when it dropped one, because a ring that failed on a full buffer
    would be a queue. The count comes back on the ring's own `dropped` field.
18. **A device record does not convert without loss, and the loss is
    reported.** A 128-bit argument, a byte string, a list and a nested type
    arrive as text. The interned index and the raw tick count do not arrive.
    `lgdeflog.lossy_fields` names what was dropped, per record.
19. **A device stamp needs a tick rate and an origin.** A device counter's unit
    is the firmware's business. `lgdeflog.tick_base` takes the ticks per second,
    a host time and the device counter read at that moment.
    `lgdeflog.no_tick_base` leaves records unstamped rather than dating them to
    1970.
20. **The human format pads the level tag to column nine**, which is the column
    `std.log`'s own text format uses. Output from both surfaces lines up during
    a migration.

## What is not included

- **A replacement for `std.log`.** The standard library's global functions reach
  a working stderr with nothing threaded anywhere, and its `Logger` already has
  per-module levels, text and JSON output and five sinks. A program those serve
  should not take this dependency. `lglog.to_std_log` is the seam for one that
  is migrating a subsystem at a time.
- **A clock inside the emit path.** Reading the time is an effect, and an emit
  that read it would charge every caller. `lglog.emit_at` takes the value.
- **A global logger.** A filter and a logger are values the caller threads. One
  process-wide threshold cannot say "debug from the scheduler, error from the
  HTTP client".
- **A generic fan-out over two arbitrary sinks.** A function may bind exactly
  one effect parameter, so a signature over two sink types with two different
  effect sets cannot be written. `lglog.emit_to_console_and_file` is the
  concrete pair, at `[io, fs]`. Any other pair calls `emit_all` twice and keeps
  its own counts.
- **A message-text filter.** `env_logger` can match on the rendered message.
  Matching on text would make the gate depend on the rendering, which this
  package defers until after the gate.
- **Local time.** The RFC 3339 stamp is UTC. A local rendering needs a timezone
  database and a clock, and the formatter has neither.
- **A transport for a device.** Nothing here reads a serial port or an ELF file.
  [deflog-decoder](https://novo-lang.org/packages/deflog-decoder) does that, and
  this package converts what it produces.
- **Running on a microcontroller.** A record holds a message, a target and a
  list of fields, all of which allocate, and none of them link on a device.
- **Use beside logging-core-nv.** See "Related packages".

## Related packages

- [logging-core-nv](https://novo-lang.org/packages/logging-core-nv) publishes
  the record, the filter, the formats, the `LgSink[e]` declaration and the
  ring's value half at a layer a library with no effects can depend on. **A
  program takes one of the two packages, not both.** Both declare `LgRecord`,
  `LgSink` and the rest under the same names, and an assembly holding both is
  refused with `E2005`. Take logging-core-nv when your library must not perform
  anything. Take this package when your program owns the destination.
- [ansi-nv](https://novo-lang.org/packages/ansi-nv) supplies the colour
  attributes the human format uses, and the arithmetic that narrows a
  twenty-four-bit colour to the sixteen a real terminal has.
- [deflog-decoder](https://novo-lang.org/packages/deflog-decoder) and
  [deflog-parser](https://novo-lang.org/packages/deflog-parser) decode a
  device's deferred log. They bring `elf-nv`, `leb128-nv`, `zigzag-nv` and
  `rzcobs-nv` with them, so this package's closure is seven packages.
- [tracing-nv](https://novo-lang.org/packages/tracing-nv) records spans, which
  are events with a duration and a parent. A log line says what happened. A span
  says how long it took and what it was part of.
- `std.log` in the standard library is the no-dependency case. Its sink is an
  enum, so every logging call is charged the union over the variants, `[io]`,
  even when the installed sink discards. Its field values are text.

## Tests

```bash
novo test tests                            # every suite
novo test tests/lgrecord_tests.nv          # the level order and the typed field
novo test tests/lgfilter_tests.nv          # longest prefix, and the segment rule
novo test tests/lgformat_tests.nv          # the three renderings and their escaping
novo test tests/lgsink_tests.nv            # the trait, and what its parameter costs
novo test tests/lgwrite_tests.nv           # the stream sinks, and the stderr default
novo test tests/lgfile_tests.nv            # rotation, as arithmetic
novo test tests/lgring_tests.nv            # the last N records, and the drop count
novo test tests/lgdeflog_tests.nv          # a device's line as a host record
novo test tests/lglog_tests.nv             # the logger, and the std.log seam
```

`novo test` fails on purpose today. Every assertion reaches a `not implemented:
logging-nv.<module>.<fn>` panic, because every body is a `todo()`. The tests are
the specification the implementation will have to satisfy.

The expected text comes from the conventions the formats are named after: the
logfmt convention for the `key=value` line, JSON Lines for the
one-object-per-line form, RFC 3339 for the timestamp spelling, and
`env_logger`'s `RUST_LOG` grammar for the filter specification. The level names
and the nine-column tag are `std.log`'s.

The suite asserts that `"ml"` governs `ml.core` and does not govern `mlx`, that
the level comparison admits what is at least as loud as the threshold, that a
caller's `msg` field is reported as a reserved key rather than renamed, that
logfmt quotes exactly the values a consumer's parser expects, and that a full
ring counts what it discarded. `tests/lgfile_tests.nv` covers the three
off-by-one faults a rotation ships with: a `keep = 3` that keeps four, a size
rule that rotates at the cap rather than past it, and a daily rotation that
skips a day at a month boundary. `tests/lgdeflog_tests.nv` asserts that a device
trace line arrives as a trace line and that a dropped value is reported.
`tests/lgsink_tests.nv` also carries an assertion the compiler makes rather than
`test.assert`: a function declared with an empty effect list emits through the
null sink, and it would not compile if the effect parameter did not do what this
package claims.

## Implementation status

| Item | Implemented |
| --- | --- |
| `lgrecord.LgLevel`, `.LgValue`, `.LgField`, `.LgRecord` | declared |
| `lgrecord`'s eighteen functions, from `level_num` to `reserved_keys` | no |
| `lgfilter.LgRule`, `.LgFilter`, `.LgFilterFault` and `impl Error` | declared |
| `lgfilter`'s ten functions, from `filter` to `redundant_rules` | no |
| `lgformat.LgHumanStyle`, `.LgStamp`, `.LgFormat`, `.LgTheme` | declared |
| `lgformat`'s seventeen functions, from `human` to `tag_column` | no |
| `lgsink.LgSink[e]`, `.LgSinkFault` and `impl Error`, `.LgNullSink`, `.LgFanout` | declared |
| `impl LgSink[] for LgNullSink`: `emit`, `flush`, `accepts` | no |
| `lgsink.null_sink`, `.null_sink_at`, `.emit_if`, `.emit_all`, `.emit_and_flush` | no |
| `lgwrite.LgStream`, `.LgStdSink` | declared |
| `impl LgSink[io] for LgStdSink`: `emit`, `flush`, `accepts` | no |
| `lgwrite.stderr_sink`, `.stdout_sink`, `.with_min`, `.with_theme`, `.with_format` | no |
| `lgwrite.is_terminal`, `.colour_depth`, `.write_record`, `.write_records` | no |
| `lgfile.LgRotate`, `.LgFilePolicy`, `.LgFileSink` | declared |
| `impl LgSink[fs] for LgFileSink`: `emit`, `flush`, `accepts` | no |
| `lgfile.policy`, `.rotating`, `.flushing` | no |
| `lgfile.open`, `.append`, `.append_all`, `.rotate_now`, `.retire` | no |
| `lgfile.should_rotate`, `.rotated_name`, `.dated_name`, `.retired_names`, `.day_of` | no |
| `lgring.LgRing`, `.LgRingSink` | declared |
| `impl LgSink[mutate] for LgRingSink`: `emit`, `flush`, `accepts` | no |
| `lgring`'s eleven functions, from `ring` to `dump_into` | no |
| `lgdeflog.LgTickBase` | declared |
| `lgdeflog`'s eleven functions, from `level_of` to `bridge_into` | no |
| `lglog.LgLogger` | declared |
| `lglog`'s fourteen functions, from `logger` to `to_std_log` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
