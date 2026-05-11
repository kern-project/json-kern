# json-bench

Micro-benchmarks for `json-kern`.

Run the generated workload:

```sh
craft run
```

Run one mode with an iteration count:

```sh
craft run -- 10000 parse
craft run -- 10000 validate
craft run -- 10000 compact
craft run -- 10000 lookup
craft run -- 10000 decode
```

Pass a file path as the third argument to benchmark external input:

```sh
craft run -- 2000 parse path/to/payload.json
```
