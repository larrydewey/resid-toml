# resid-toml

[TOML](https://toml.io/) as a format for
[resid-serial](https://github.com/larrydewey/resid-serial), the serialization framework.

| File | What it is |
|---|---|
| `src/text.resid` | key and string quoting, number rendering |
| `src/parse.resid` | the grammar, parsed into a tree: tables, arrays of tables, dotted keys, inline tables, every string and number form, dates |
| `src/toml.resid` | the `Encoder` and the `Decoder`, and the tree the encoder's leaves make |

## What a TOML value is

A **document** is a table: `key = value` lines at the top and `[table]`
sections below, a table's own keys inside its section. An **array of tables**
is written `[[name]]`, one section per element. Keys may be bare, basic
strings or literal strings, and a dotted key reaches into a table from
anywhere.

Two consequences shape the code:

- Decoding parses the whole document first and then navigates the tree, so a
  field may be asked for by name in any order, a key may be written before
  or after the table it belongs to, and a dotted key may reach into a table
  from anywhere. A field the type did not ask for is skipped, and the next
  field is found by name rather than by position.
- Encoding collects one leaf per value with the path it was written at, and
  renders the tree at the end: a table's own keys, then a section per
  sub-table, and `[[name]]` per element of an array of tables. A length
  cannot be known while a section is being written, and a struct's fields
  arrive in the order the type declares them, so the leaves are gathered
  first and the sections come out of them.

A table is a **value** where a value belongs — as an array element, or as a
variant's own keys — and a **section** where a table belongs in a document.
Both spellings read back the same, so a document written by hand and one
written by `to_toml` decode alike.

## The data model

| Resid | TOML |
|---|---|
| a record, a map | a table, so a map's keys have to be strings |
| a sequence | an array, or `[[name]]` sections when the elements are records |
| `Str`, `Int`, `Float`, `Bool` | the values themselves |
| `Option(T)` | `Some(v)` is `v`; TOML has no null, so `None` is refused |
| `Str(N)` | streamed a code point at a time, like any other string |
| `Unit`, bytes, `Dec(N)` | no spelling; refused in both directions |
| `Int(128)`, `UInt(128)` | an integer while it fits in the 64 bits TOML has |
| a variant | a table with a `type` key: `{type = "Circle", value = 3}`, and a unit variant is `{type = "Dot"}` |
| a date-time | a string, since the data model has no date: the text is kept as written and reads back the same |

At the top of a document a table *is* the document, so a bare scalar, a
sequence or a unit has no spelling there and `to_toml` says
`a TOML document, whose top level is a table`. A record or a map is
written as the document's own keys. A variant at the top of a document has
the same shape: its `type` and `value` keys are the document's keys, since
`{type = "Dot"}` on its own is not a document.

## Entry points

```resid
import "src/toml.resid";

Result(Str, SerialError) text = to_toml(point) else { return 1; };
Result(Point, SerialError) back = from_toml(text) else { return 1; };
```

A document can also be read by hand: `toml_parse(text)` gives the tree and
`toml_decoder(root)` gives a `Decoder` over it,
and `tm_at`, `tm_key_at`, `tm_text`, `tm_int`, `tm_real`, `tm_flag`,
`tm_items` and `tm_table` read out of it.

## What the parser takes

Checked by `tests/vectors.resid`, which reads the specification's own
examples:

- keys: bare, `"quoted"`, `'literal'`, dotted, and the dotted and quoted
  forms of a table header
- tables: `[a]`, `[a.b]`, `[a.b.c]`, out of order, reopened, and under a
  table that already has pairs
- arrays of tables: `[[a]]`, several of them, nested under a table, and
  `[[a.b]]` below `[[a]]`
- arrays: empty, mixed, nested, trailing commas, newlines inside, arrays of
  inline tables
- inline tables: empty, one pair, several, a trailing comma, nested, with
  an array inside
- strings: basic, literal, multi-line in both forms, escapes, `\uXXXX` and
  `\UXXXXXXXX`, the line-ending backslash, the newline after the opening
  delimiter trimmed
- integers: decimal, hex, octal, binary, underscores, signs, and the 64-bit
  range
- floats: whole, fractional, exponents, underscores, `inf`, `-inf`, `nan`
- booleans, and comments at the end of a line or on a line of their own
- date-times, local dates and local times, kept as the text they were
  written as

## What it refuses

Checked by `tests/errors.resid`, every case an error value with a kind and a
path rather than an abort: a missing `=`, a value that is not there, a
string or table or array that is never closed, an empty key, a key or a
table defined twice, a header that would redefine a value as a table or the
other way round, an integer outside the 64 bits, a missing required field, a
field of the wrong type, a variant with no `type` key or an unknown one, and
reading past the end of the document.

## Tests

```
tests/run.sh            # build resid-derive, then every program
tests/run.sh --update   # rewrite the golden .out files
```

| Program | Checks | What it covers |
|---|---|---|
| `tests/vectors.resid` | 75 | the documents the specification writes, parsed and read back, and the text the encoder writes for them |
| `tests/roundtrip.resid` | 29 | values through the whole pipeline, with hand-written instances, so the test shows what one looks like |
| `tests/errors.resid` | 36 | what the format reports when the document is wrong |
| `tests/value.resid` | 46 | the format-neutral `Value` tree, which is where `read_peek` and `read_skip` are exercised |
| `tests/derive.resid` | 20 | instances generated by `resid-derive`, to show they need nothing from the format beyond the protocol |

## Requirements

A Resid compiler with the changes made alongside resid-serial (spec v3.8).
The framework is a dependency in `resid.toml`, and `src/*.resid` import it
by package name (`import "resid-serial/serial.resid";`), so it resolves the
same from a sibling `../resid-serial` checkout or from a registry, and
compiles inside its declared capability ceiling. Build through the
manifest tool (`resid-manifest build resid.toml residc`), or pass
`residc` the dependency map it writes (`resid-manifest depmap resid.toml
deps.txt`, then `residc file.resid -depmap deps.txt`). `./install.sh` in
the Resid checkout installs `resid-manifest` beside `residc`.

## License

MIT. See [LICENSE](LICENSE).
