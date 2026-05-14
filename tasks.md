# json-kern Tasks

`json-kern` should become the default Kern package for JSON validation,
borrowing, decoding, owning, indexing, and writing. The package replaces the old
incubator experiment with a redesigned public surface, not a file move.

## Priority 0: Package Shape

- [x] Create a standalone Craft package named `json` with `json/src/lib.kn`,
  focused tests, README, MIT license, and no dependency on the old incubator
  source.
- [x] Promote the repository root to a Craft workspace that manages `json`,
  `json-test`, and `json-bench` together.
- [~] Start from a small public API that can grow coherently:
  - [x] `source.parse_json()` returns a borrowed `Value`.
  - [x] `source.validate_json()` checks a complete document.
  - [x] `value.array()`, `value.object()`, and scalar methods expose typed views.
  - [x] `value.clone_document(alloc)` builds an owned document when ownership is
    needed.
  - [~] `document.root()` exists; `document.root_mut()` and value-level mutation
    handles are still pending.
- [x] Avoid preserving parallel low-level variants just because the incubator had
  them. Add packed/indexed representations only after their receiver model and
  ownership story are clearer than the default document API.
- [x] Split `json/src/lib.kn` into focused modules:
  - `bytes.kn`
  - `parser.kn`
  - `decode.kn`
  - `value.kn`
  - `document.kn`
  - `render.kn`
  - `builder.kn`
  - `format.kn`

## Priority 1: Borrowed Parser And Values

- [x] Implement allocation-free validation and complete-value parsing.
- [x] Preserve byte offsets for structured parse errors.
- [x] Add diagnostic location helpers.
- [~] Expose borrowed scalar views:
  - [x] `is_null`, `bool_value`, `number_text`
  - [x] `string_raw`, decoded string sizing/writing/cloning
  - [x] strict integer decoding through `i64_value`
  - [x] strict float decoding
- [x] Expose borrowed container cursors:
  - array cursor with `next()`
  - object cursor with entries that expose key and value views
- [x] Define duplicate-key behavior explicitly. Optional lookup returns the first
  decoded key match; `find_unique()` and `reject_duplicate_keys()` reject
  duplicate decoded keys for required schema-style workflows.

## Priority 2: Owned Document

- [~] Implement an owned `Document` whose storage strategy is internal to the
  document. Current implementation owns compact JSON text and exposes borrowed
  root views.
- [ ] Provide `DocumentValue` and `DocumentValueMut` handles for traversal and
  mutation.
- [x] Model allocation failures with structured errors, including capacity
  overflow where it is distinguishable.
- [ ] Support builder-style mutation:
  - object append/set/remove
  - array push/pop/remove
  - scalar replacement
- [x] Keep cleanup handle-oriented: `document.deinit(alloc)`.

## Priority 3: Rendering And Writing

- [x] Provide compact rendering into `&mut [u8]` and into `base.io.Write`.
- [x] Add a fluent writer/builder for constructing JSON without manually
  counting separators.
- [x] Keep escaping rules explicit and documented next to writer APIs.
- [x] Make buffer sizing and short-output failures structured.

## Priority 4: Documentation And Tests

- [~] README shows borrowed parsing, owned document cloning/replacement, and
  rendering/writing. Fine-grained owned mutation is pending with the API.
- [x] Module docs teach ownership, duplicate-key policy, string decoding, and
  numeric strictness for the implemented modules.
- [~] Public items need `///` contracts for allocation, borrowing, cleanup, and
  error boundaries. Current public API is documented; run `craft style` during
  review to keep coverage visible.
- [~] Tests cover:
  - [x] full JSON grammar smoke and JSONTestSuite-shaped valid/invalid cases
  - [x] parse offsets in focused smoke tests
  - [x] borrowed object/array traversal
  - [x] string escape decoding
  - [x] integer errors
  - [x] float errors
  - [x] duplicate-key first-match and duplicate rejection behavior
  - [x] streaming writer separators, escaping, and state errors
  - [~] owned mutation and rendering; root replacement and rendering covered,
    fine-grained mutation pending
  - [x] README-shaped compile tests
- [x] Add `json-test` as an in-repository extended conformance runner.
- [x] Add `json-bench` as an in-repository benchmark runner.

## Priority 5: Expansion Path

- Indexed object views when repeated lookup is a measured need.
- Packed/arena-backed documents if they simplify ownership rather than adding
  a second public universe.
- JSON Pointer and patch helpers.
- Schema-style helpers once the field/error model is stable.
- Benchmarks against common corpora after the API is stable.

## Done For First Publishable Version

- [x] `craft fmt --check --verbose --color never` passes.
- [x] `craft test --color never` passes.
- [x] `craft style --verbose --color never` reports no missing public docs for the
  hand-written public API.
- [x] README examples have matching compile-only tests.
- [ ] The old incubator implementation is deleted from `kern` or clearly no longer
  referenced by workspace docs/tests.
