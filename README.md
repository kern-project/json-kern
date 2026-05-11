# json-kern

JSON parsing, decoding, document, and rendering utilities for Kern.

The Craft package name is `json`; the repository name is `json-kern`. This
package replaces the old incubator JSON experiment with a redesigned API based
on borrowed views, owned document handles, and explicit error boundaries.

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
}

fn app() void!AppError {
    let .{ Ok: value } = "true".parse_json() else {
        .{ Err: err } => return .{ Err: .{ Parse: err } },
    };
    _ = value;
    return .{ Ok: {} };
}
```

Planned entry points:

- `source.validate_json()` validates a complete document.
- `source.parse_json()` returns a borrowed `Value`.
- `value.array()`, `value.object()`, and scalar methods expose typed borrowed
  views.
- `value.clone_document(alloc)` builds an owned `Document`.
- `document.root()` and `document.root_mut()` provide the owned tree action
  surface.
- Writer and renderer handles produce compact JSON with structured errors.

## Design

The borrowed layer should remain allocation-free. Owning APIs should hide their
internal storage strategy behind `Document`, so ordinary users do not choose
between raw node pools, packed documents, or indexing variants before they know
they need them.

Indexed and packed representations can be added later when their receiver model
is clearer than the default document API.

## License

`json-kern` is distributed under the MIT License. See `LICENSE`.
