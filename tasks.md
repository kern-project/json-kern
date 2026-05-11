# json-kern Tasks

`json-kern` should become the default Kern package for JSON validation,
borrowing, decoding, owning, indexing, and writing. The package replaces the old
incubator experiment with a redesigned public surface, not a file move.

## Priority 0: Package Shape

- Create a standalone Craft package named `json` with `src/lib.rn`, focused
  tests, README, MIT license, and no dependency on the old incubator source.
- Start from a small public API that can grow coherently:
  - `source.parse()` returns a borrowed `Value`.
  - `source.validate()` checks a complete document.
  - `value.array()`, `value.object()`, and scalar methods expose typed views.
  - `value.clone_document(alloc)` builds an owned document when ownership is
    needed.
  - `document.root()` and `document.root_mut()` are the owned tree action
    surface.
- Avoid preserving parallel low-level variants just because the incubator had
  them. Add packed/indexed representations only after their receiver model and
  ownership story are clearer than the default document API.

## Priority 1: Borrowed Parser And Values

- Implement allocation-free validation and complete-value parsing.
- Preserve byte offsets for structured parse errors and diagnostic location
  helpers.
- Expose borrowed scalar views:
  - `is_null`, `bool_value`, `number_text`
  - `string_raw`, decoded string sizing/writing/cloning
  - strict integer and float decoding
- Expose borrowed container cursors:
  - array cursor with `next()`
  - object cursor with entries that expose key and value views
- Define duplicate-key behavior explicitly. Optional lookup may return the first
  match, but required schema helpers should be able to reject duplicates.

## Priority 2: Owned Document

- Implement an owned `Document` whose storage strategy is internal to the
  document. Users should not manually choose per-node storage for ordinary use.
- Provide `DocumentValue` and `DocumentValueMut` handles for traversal and
  mutation.
- Model allocation failures with structured errors, including capacity overflow
  where it is distinguishable.
- Support builder-style mutation:
  - object append/set/remove
  - array push/pop/remove
  - scalar replacement
- Keep cleanup handle-oriented: `document.deinit(alloc)`.

## Priority 3: Rendering And Writing

- Provide compact rendering into `&mut [u8]` and into `base.io.Write`.
- Add a fluent writer/builder for constructing JSON without manually counting
  separators.
- Keep escaping rules explicit and documented next to writer APIs.
- Make buffer sizing and short-output failures structured.

## Priority 4: Documentation And Tests

- README should show borrowed parsing, owned document mutation, and rendering.
- Module docs should teach ownership, duplicate-key policy, string decoding, and
  numeric strictness.
- Public items need `///` contracts for allocation, borrowing, cleanup, and
  error boundaries.
- Tests must cover:
  - full JSON grammar and parse offsets
  - borrowed object/array traversal
  - string escape decoding
  - integer and float errors
  - duplicate-key behavior
  - owned mutation and rendering
  - README-shaped compile tests

## Priority 5: Expansion Path

- Indexed object views when repeated lookup is a measured need.
- Packed/arena-backed documents if they simplify ownership rather than adding
  a second public universe.
- JSON Pointer and patch helpers.
- Schema-style helpers once the field/error model is stable.
- Benchmarks against common corpora after the API is stable.

## Done For First Publishable Version

- `craft fmt --check --verbose --color never` passes.
- `craft test --color never` passes.
- `craft style --verbose --color never` reports no missing public docs for the
  hand-written public API.
- README examples have matching compile-only tests.
- The old incubator implementation is deleted from `kern` or clearly no longer
  referenced by workspace docs/tests.
