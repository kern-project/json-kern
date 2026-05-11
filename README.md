# json-kern

JSON parsing, borrowed traversal, and compact rendering utilities for Kern.

The Craft package name is `json`; the repository name is `json-kern`. The
current public surface is the fast borrowed layer: parsing and validation do
not allocate, `Value` borrows the original input, and array/object cursors walk
the source text directly. Owned document editing will build on this layer.

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

enum AppError {
    Parse: json.ParseError,
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
        .{ Err: err } => return .{ Err: .{ Parse: err } },
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
    return .{ Ok: {} };
}
```

## Borrowed API

- `source.validate_json()` validates a complete document.
- `source.parse_json()` returns a borrowed `Value`.
- `value.is_null()`, `value.bool_value()`, `value.number_text()`,
  `value.i64_value()`, and `value.string_raw()` expose scalar views.
- `value.array()` returns an `ArrayCursor`.
- `value.object()` returns an `ObjectCursor`.
- `object.find(key)` returns the first raw key match. Duplicate keys are
  preserved by cursor iteration.
- `value.render_compact(out)` and `source.render_json_compact(out)` write
  compact JSON into caller-owned output.

String APIs currently expose raw string contents. Escape decoding into caller
storage is planned for the next layer so callers can choose allocation and
Unicode policy explicitly.

## Performance Notes

The parser validates with a single recursive scan over the source text.
Borrowed cursors scan the selected array or object as they advance; they do not
allocate an index. This is the intended default for one-pass protocol handling.
Repeated object lookup should use an indexed view later when measurements show
the repeated lookup cost matters.

## License

`json-kern` is distributed under the MIT License. See `LICENSE`.
