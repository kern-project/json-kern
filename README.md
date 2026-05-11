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
json = { path = "../json-kern" }
```

```kern
use json;
use base.coll.string;
use base.io.Write;
use base.mem.alloc.gpa;
use sys.mem.page;

enum AppError {
    Parse: json.ParseError,
    Key: json.KeyError,
    Render: json.RenderError,
}

fn app() void!AppError {
    let .{ Ok: root } = "{\"name\":\"kern\",\"enabled\":true}".parse_json() else {
        .{ Err: err } => return .{ Err: .{ Parse: err } },
    };

    let .{ Some: object0 } = root.object() else {
        return .{ Err: .{ Parse: .EmptyInput } };
    };
    let object = object0..&;
    let .{ Ok: enabled } = object.find("enabled") else {
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

    let mut out = [32]u8.{undef};
    _ = root.render_compact(out..&[0 .. 32])
        .map_err([](err: json.RenderError) AppError { return .{ Render: err }; })
        .!;

    let page = page()..&;
    let alloc = gpa().on(page)..&;
    defer alloc.deinit();

    let mut compact = string();
    defer compact..&.deinit(alloc);
    let mut string_writer = compact..&.writer(alloc);
    let writer = &mut Write.{ string_writer..& };
    _ = root.write_compact(writer)
        .map_err([](err: json.RenderError) AppError { return .{ Render: err }; })
        .!;

    let doc = root.clone_document(alloc)
        .map_err([](_: json.DocumentError) AppError {
            return .{ Render: .{ Parse: .EmptyInput } };
        })
        .!;
    defer doc..&.deinit(alloc);
    return .{ Ok: {} };
}
```

## Borrowed API

- `source.validate_json()` validates a complete document.
- `source.parse_json()` returns a borrowed `Value`.
- `value.is_null()`, `value.bool_value()`, `value.number_text()`,
  `value.i64_value()`, and `value.string_raw()` expose scalar views.
- `value.write_string(out)` and `value.clone_string(alloc)` decode JSON string
  escapes into UTF-8 on demand.
- `value.array()` returns an `ArrayCursor`.
- `value.object()` returns an `ObjectCursor`.
- `entry.write_key(out)` and `entry.clone_key(alloc)` decode object keys.
- `object.find(key)` returns the first decoded key match. Duplicate keys are
  preserved by cursor iteration.
- `value.render_compact(out)` and `source.render_json_compact(out)` write
  compact JSON into caller-owned output.
- `value.write_compact(writer)` and `source.write_json_compact(writer)` stream
  compact JSON into a `base.io.Write` sink.
- `value.clone_document(alloc)` and `source.parse_json_document(alloc)` build
  an owned compact `Document`.
- `document.root()` returns a borrowed view over the owned compact storage.
- `document.replace_root_json(alloc, text)` validates and replaces the whole
  document root.

String decoding is explicit: parsing validates JSON escape syntax, while
`write_string`, `clone_string`, `write_key`, and `clone_key` decode Unicode
escapes into UTF-8 at the caller's chosen allocation boundary.

## Performance Notes

The parser validates with a single recursive scan over the source text.
Borrowed cursors scan the selected array or object as they advance; they do not
allocate an index. This is the intended default for one-pass protocol handling.
Repeated object lookup should use an indexed view later when measurements show
the repeated lookup cost matters.

## Development

The library source is split by responsibility:

- `src/parser.rn` and `src/bytes.rn`: allocation-free grammar scanning.
- `src/value.rn`: borrowed `Value`, array cursor, object cursor, and key APIs.
- `src/decode.rn`: JSON string and key escape decoding.
- `src/render.rn`: compact rendering into buffers and `base.io.Write`.
- `src/document.rn`: owned compact `Document`.
- `src/format.rn`: diagnostic formatting and exact error equality.

Run the published package checks from the repository root:

```sh
craft fmt --check --verbose --color never
craft test --color never
craft style --verbose --color never
```

The heavier conformance runner lives in `json-test`:

```sh
cd json-test
craft run --color never
```

It carries RFC 8259 and JSONTestSuite-shaped `y_`, `n_`, and `i_` cases. A
mirrored external corpus such as `nst/JSONTestSuite` can be placed under
`json-test/fixtures/JSONTestSuite` as the data-driven runner grows.

The benchmark tool lives in `json-bench`:

```sh
cd json-bench
craft run --color never -- 10000 parse
craft run --color never -- 10000 validate
craft run --color never -- 10000 compact
craft run --color never -- 10000 lookup
craft run --color never -- 10000 decode
```

Pass a JSON file path as the third argument to benchmark a larger corpus.

## License

`json-kern` is distributed under the MIT License. See `LICENSE`.
