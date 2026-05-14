# json-kern

JSON parsing, borrowed traversal, and compact rendering utilities for Kern.

The Craft package name is `json`; the repository name is `json-kern`. The
current public surface starts with the fast borrowed layer: parsing and
validation do not allocate, `Value` borrows the original input, and
array/object cursors walk the source text directly. `Document` owns one compact
JSON buffer and exposes borrowed root views from that storage.

## Usage

```toml
[dependencies]
json = { git = "https://github.com/softfault/json-kern.git" }
```

Local development can use a path dependency:

```toml
[dependencies]
json = { path = "../json-kern/json" }
```

```kern
use json;
use base.coll.string;
use base.io.Write;
use base.mem.alloc.gpa;
use std.mem.Page;

enum AppError {
    Parse: json.ParseError,
    Key: json.KeyError,
    Render: json.RenderError,
    Write: json.WriteError,
}

fn app(alloc: &mut base.mem.alloc.Allocator) void!AppError {
    let .{ Ok: root } = "{\"name\":\"kern\",\"enabled\":true}".parse_json(alloc) else {
        .{ Err: err } => return .{ Err: .{ Parse: err } },
    };

    let .{ Some: object0 } = root.object() else {
        return .{ Err: .{ Parse: .EmptyInput } };
    };
    let object = object0..&;
    let .{ Ok: enabled } = object.find(alloc, "enabled") else {
        .{ Err: err } => return .{ Err: .{ Key: err } },
    };
    let .{ Some: flag } = enabled else {
        return .{ Err: .{ Parse: .EmptyInput } };
    };

    let .{ Some: enabled_flag } = flag.bool_value() else {
        return .{ Err: .{ Parse: .EmptyInput } };
    };
    if (!enabled_flag) {
        return .{ Err: .{ Parse: .EmptyInput } };
    }

    let mut out: [32]u8 = undef;
    _ = root.render_compact(alloc, out..&[0...32])
        .map_err([](err: json.RenderError) AppError { return .{ Render: err }; })
        .?;

    let mut compact = string();
    defer compact..&.deinit(alloc);
    let mut string_writer = compact..&.writer(alloc);
    let writer = (string_writer..& as &mut Write);
    _ = root.write_compact(alloc, writer)
        .map_err([](err: json.RenderError) AppError { return .{ Render: err }; })
        .?;

    let mut built = string();
    defer built..&.deinit(alloc);
    let mut built_writer = built..&.writer(alloc);
    let built_sink = (built_writer..& as &mut Write);
    let json_writer = json.writer(alloc, built_sink)..&;
    defer json_writer.deinit();
    json_writer.begin_object().map_err([](err: json.WriteError) AppError {
        return .{ Write: err };
    }).?;
    json_writer.key("name").map_err([](err: json.WriteError) AppError {
        return .{ Write: err };
    }).?;
    json_writer.string("kern").map_err([](err: json.WriteError) AppError {
        return .{ Write: err };
    }).?;
    json_writer.end_object().map_err([](err: json.WriteError) AppError {
        return .{ Write: err };
    }).?;
    json_writer.finish().map_err([](err: json.WriteError) AppError {
        return .{ Write: err };
    }).?;

    let doc = root.clone_document(alloc)
        .map_err([](_: json.DocumentError) AppError {
            return .{ Render: .{ Parse: .EmptyInput } };
        })
        .?;
    let mut owned = doc;
    defer owned..&.deinit(alloc);
    let root_mut = owned..&.root_mut()..&;
    root_mut.object_set_json(alloc, "enabled", "false")
        .map_err([](_: json.DocumentError) AppError {
            return .{ Render: .{ Parse: .EmptyInput } };
        })
        .?;
    return .{ Ok: {} };
}

fn main() i32 {
    let page = Page.{}..&;
    let alloc = gpa().on(page)..&;
    defer alloc.deinit();

    let .{ Ok: _ } = app(alloc) else return 1;
    return 0;
}
```

## Borrowed API

- `source.validate_json(alloc)` validates a complete document.
- `source.parse_json(alloc)` returns a borrowed `Value`.
- `value.is_null()`, `value.bool_value()`, `value.number_text()`,
  `value.i64_value()`, `value.f64_value()`, and `value.string_raw()` expose
  scalar views.
- `value.require_f64()` reports explicit `json.NumberError` failures when a
  number is malformed or outside the finite `f64` range.
- `parse_error.offset()` and `parse_error.location_in(source)` expose byte
  offsets and one-based line/column diagnostics when a parse error carries an
  exact position.
- `value.write_string(out)` and `value.clone_string(alloc)` decode JSON string
  escapes into UTF-8 on demand.
- `value.array()` returns an `ArrayCursor`.
- `value.object()` returns an `ObjectCursor`.
- `entry.write_key(out)` and `entry.clone_key(alloc)` decode object keys.
- `object.find(alloc, key)` returns the first decoded key match. Duplicate keys are
  preserved by cursor iteration.
- `object.find_unique(alloc, key)` and `object.reject_duplicate_keys(alloc)` reject
  duplicate decoded keys with `json.KeyError.DuplicateKey` for schema-style reads.
- `value.render_compact(alloc, out)` and `source.render_json_compact(alloc, out)` write
  compact JSON into caller-owned output.
- `value.write_compact(alloc, writer)` and `source.write_json_compact(alloc, writer)` stream
  compact JSON into a `base.io.Write` sink.
- `json.writer(alloc, writer)` constructs JSON into a `base.io.Write` sink,
  inserting separators and escaping string/key bytes.
- `value.clone_document(alloc)` and `source.parse_json_document(alloc)` build
  an owned compact `Document`.
- `document.root(alloc)` returns a borrowed view over the owned compact storage.
- `document.replace_root_json(alloc, text)` validates and replaces the whole
  document root.
- `document.root_mut()` returns a compact-text mutation handle with scalar
  replacement, array push/pop, and object append/set/remove helpers.

String decoding is explicit: parsing validates JSON escape syntax, while
`write_string`, `clone_string`, `write_key`, and `clone_key` decode Unicode
escapes into UTF-8 at the caller's chosen allocation boundary.

String construction is explicit too: `JsonWriter.string()` and
`JsonWriter.key()` escape quotes, backslashes, control bytes, and common JSON
control escapes while writing directly into the caller's sink. `raw_number()`
accepts only one complete JSON number; floating-point formatting policy is left
to callers until Kern's freestanding number formatting grows a stable float
surface.

## Performance Notes

The parser validates with an explicit stack allocated through the caller's
allocator. This keeps nesting depth out of the machine call stack while keeping
allocation visible at the API boundary. Borrowed cursors scan the selected array
or object as they advance; they do not allocate an index. Repeated object lookup
should use an indexed view later when measurements show the repeated lookup cost
matters.

## Development

The library source is split by responsibility:

- `json/src/parser.kn` and `json/src/bytes.kn`: allocation-free grammar
  scanning.
- `json/src/value.kn`: borrowed `Value`, array cursor, object cursor, and key
  APIs.
- `json/src/decode.kn`: JSON string and key escape decoding.
- `json/src/render.kn`: compact rendering into buffers and `base.io.Write`.
- `json/src/builder.kn`: streaming JSON construction and string escaping.
- `json/src/document.kn`: owned compact `Document`.
- `json/src/format.kn`: diagnostic formatting and exact error equality.

The repository root is a Craft workspace for the published package and local
tools:

- `json`: publishable library package.
- `json-test`: extended conformance runner.
- `json-bench`: benchmark runner.

Run workspace checks from the repository root:

```sh
craft fmt --check --verbose --color never
craft test --color never
craft style --verbose --color never
```

The heavier conformance runner lives in `json-test`:

```sh
craft run --project-path json-test --color never
```

It carries RFC 8259 and JSONTestSuite-shaped `y_`, `n_`, and `i_` cases. A
mirrored external corpus such as `nst/JSONTestSuite` can be placed under
`json-test/fixtures/JSONTestSuite` as the data-driven runner grows.

The benchmark tool lives in `json-bench`:

```sh
craft run --project-path json-bench --color never -- 10000 parse
craft run --project-path json-bench --color never -- 10000 validate
craft run --project-path json-bench --color never -- 10000 compact
craft run --project-path json-bench --color never -- 10000 lookup
craft run --project-path json-bench --color never -- 10000 decode
```

Pass a JSON file path as the third argument to benchmark a larger corpus.

## License

`json-kern` is distributed under the MIT License. See `LICENSE`.
