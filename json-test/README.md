# json-test

Extended conformance tests for `json-kern`.

The built-in cases follow the public JSONTestSuite convention:

- `y_` cases must parse.
- `n_` cases must reject.
- `i_` cases document implementation-defined pressure cases.

Run from the workspace root with:

```sh
craft run --project-path json-test
```

Or from this directory with:

```sh
craft run
```

External corpora such as
[`nst/JSONTestSuite`](https://github.com/nst/JSONTestSuite) can be mirrored
under `fixtures/JSONTestSuite`; the package layout is kept separate from the
published library so exhaustive data-driven checks can grow without slowing
ordinary `json` package tests.
