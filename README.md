# Mini Redis

A Redis-inspired in-memory key-value store built in C++17 with an HTTP/JSON REST API, TTL expiration, append-only persistence, and a browser dashboard.

Mini Redis explores how a database handles concurrent requests, expires temporary data, and recovers state after a process crash. The server uses the C++ standard library and POSIX APIs, with no third-party library dependencies.

This is an educational systems project inspired by Redis. It communicates using **HTTP and JSON**, not **RESP (Redis Serialization Protocol)**. It works with HTTP clients such as `curl`, but is **not compatible with `redis-cli` or other RESP-based Redis clients**. Its append-only log also uses a custom disk format.

## Quick start

Requires a C++17 compiler and Make on macOS or Linux. No downloaded libraries, database installation, package manager, or frontend build is needed. Windows users can build under WSL; native Windows is not implemented.

From the project directory:

```sh
./run.sh
```

Open **http://127.0.0.1:7379**. The dashboard can create, inspect, edit, delete, and filter keys. It also shows current key count, value bytes, read hit rate, and successful PUT count. Statistics refresh every three seconds; refresh the explorer to update its rows.

Stop with **Ctrl+C**. Start again with the same data directory to recover your data.

Options:

```sh
./run.sh --port 8080 --threads 8 --data-dir ./my-data
```

`run.sh` changes to the project directory before running, so relative data paths are relative to the project. Running the executable directly instead uses your current working directory:

```sh
make
./build/mini-redis --port 7379 --threads 4 --data-dir data
```

Alternatively, if CMake is installed:

```sh
cmake -S . -B build-cmake
cmake --build build-cmake
./build-cmake/mini-redis
```

## REST API

Save a workout for one hour:

```sh
curl -i -X PUT http://127.0.0.1:7379/keys/workout:last \
  -H 'Content-Type: application/json' \
  -d '{"value":"Pull day","ttl_seconds":3600}'
```

Read it:

```sh
curl http://127.0.0.1:7379/keys/workout:last
```

```json
{"key":"workout:last","value":"Pull day","ttl_seconds":3600,"expires_at_ms":1790800000000}
```

The expiration timestamp above is illustrative; real values depend on when the request runs.

Save without expiration, list keys, view statistics, or delete:

```sh
curl -X PUT http://127.0.0.1:7379/keys/climbing:best \
  -H 'Content-Type: application/json' -d '{"value":"V6"}'
curl 'http://127.0.0.1:7379/keys?prefix=climbing%3A&limit=50'
curl http://127.0.0.1:7379/stats
curl -X DELETE http://127.0.0.1:7379/keys/climbing:best
```

| Method | Endpoint | Result |
| --- | --- | --- |
| GET | `/` | Browser dashboard |
| GET | `/health` | `200` when writable; `503` after a disk error |
| GET | `/stats` | Key count, value bytes, counters, log size, uptime, worker count |
| GET | `/keys?prefix=&limit=100&after=` | Sorted key metadata and `next_after` cursor |
| PUT | `/keys/{key}` | `201` for a new key, `200` for replacement |
| GET | `/keys/{key}` | Key, string value, remaining TTL, absolute expiration; `404` if absent |
| DELETE | `/keys/{key}` | `200` with `{"deleted":true}`; `404` if absent |

### Semantics & limits

- Keys: 1–512 bytes of UTF-8. Percent-encode special characters in URLs. Decoding happens exactly once; `+` in a path stays `+`.
- Values: strings of 0–65,536 UTF-8 bytes. To store an object, serialize it into a string yourself. Unicode and escaped control characters are supported.
- PUT accepts only `value` and optional `ttl_seconds`. Duplicate or unknown fields are errors.
- TTL: an integer from 1 to 315,360,000 seconds. Omit it or use `null` for no expiration. Every PUT **replaces the previous TTL**, including removing it when omitted. Zero, negative, fractional, and exponent-form TTLs are rejected.
- GET rounds remaining seconds upward. Non-expiring keys return `null` for both TTL fields. Keys cannot be read after their deadline, even if background cleanup has not run yet.
- Listing returns metadata, not values. `limit` is 1–1000. Pass `next_after` as the next request's `after`. Pages are not a transactionally frozen snapshot if keys change between requests.
- JSON errors use `{"error":"message"}` and an appropriate HTTP status.
- Header limit: 16 KiB. Request body limit: 512 KiB, allowing even a maximum-sized value encoded entirely as JSON escapes.
- One request per connection; responses include `Connection: close`. Requests with bodies use Content-Length. Chunked transfer, Expect/100-continue, HEAD, and keep-alive are not supported.
- Request reading and response writing each have a five-second deadline once handled by a worker. At most 128 connections wait in the application queue; additional connections receive `503`.
- The service binds only to `127.0.0.1`. Use `localhost:PORT` or `127.0.0.1:PORT` as Host. Other Hosts and cross-origin browser requests are rejected. There is no authentication or TLS; this is a local development service, not a public deployment.

## How it works

```text
Browser / curl / application
            |
        HTTP + JSON
            |
    bounded connection queue
            |
     fixed worker pool
            |
   mutex-protected storage engine
       |                 |
  hash table       append-only log
       |                 |
  expiry index     fsync before success
       |
 background cleanup every 250 ms
```

### Storage and concurrency

An `unordered_map` gives average O(1) key lookup. A sorted expiration index tracks only the current deadline for each expiring key, so repeatedly overwriting a key does not accumulate stale timers. Inserting or changing an expiring key costs O(log E), where E is the number of expiring keys.

A single mutex serializes store operations, including logging. The HTTP worker pool handles concurrent connections; storage operations themselves run one at a time. This deliberately favors understandable ordering and correctness. Reads can wait behind a write's disk sync. Key listing scans and sorts matching keys, so it is intended for small datasets.

### Persistence and recovery

`data/appendonly.aof` begins with the eight-byte version marker `MRAOF001`. Each subsequent record contains:

```text
payload length: uint32 little-endian
payload CRC32:  uint32 little-endian
payload:
  operation:    1 byte ('S' for set, 'D' for delete)
  expiration:   uint64 little-endian, Unix milliseconds; 0 = forever
  key length:   uint32 little-endian
  value length: uint32 little-endian
  key bytes
  value bytes
```

Mutations are logged and synced with `fsync` before updating memory and acknowledging success. On startup, records are replayed in order. Expiration uses absolute deadlines, so restarting does not renew TTLs. An incomplete final record is truncated to the last complete record. A complete record with an invalid checksum, invalid header, or invalid structure stops startup rather than silently discarding data.

An exclusive file lock prevents two processes from writing the same log. After a disk-write or sync error, further mutations are blocked and health becomes `503`; rollback of the failed append is best effort. Resolve the disk issue before restarting. A disconnected client may have an uncertain result even if the write committed; GET can check the resulting state.

The tests exercise abrupt **process termination**, not physical power loss. Actual power-loss guarantees also depend on the OS, filesystem, and storage hardware honoring sync requests. CRC32 detects accidental corruption, not malicious tampering.

### Statistics

`value_bytes` counts current value payloads, not process memory or key/container overhead. `log_bytes` includes historical records. Hits and misses count individual GET-key requests only; listing does not affect them. Writes, deletes, expirations, hits, and misses reset on restart. Keys and payload bytes reflect recovered state.

## Tests and benchmark

Python 3.9+ is needed for the test and benchmark scripts. They create isolated temporary databases and local-only servers, leaving your application data alone.

```sh
make test
make sanitize
python3 tools/benchmark.py --requests 500 --clients 8 --output benchmark-results.json
```

The integration suite covers CRUD, Unicode, size limits, TTL replacement, background expiry, TTL across downtime, process-crash recovery, incomplete log tails, corruption rejection, exclusive locking, concurrent writes/replay, pagination, malformed JSON/HTTP, origin checks, and shutdown/restart.

`make sanitize` uses AddressSanitizer and UndefinedBehaviorSanitizer. It requires a compiler/runtime supporting those tools.

If AddressSanitizer itself cannot initialize on your OS/compiler combination, run `make sanitize SANITIZERS=undefined` for the undefined-behavior checks alone. This is not a replacement for an AddressSanitizer run on a supported machine.

The benchmark runs read, durable-write, and 80%-read workloads against 100 keys with 256-byte values. It reports throughput plus p50/p95/p99 end-to-end latency. It includes Python, TCP-connection, HTTP, and JSON overhead, and is **not** a comparison to production Redis. Report your machine, workload, and durability settings alongside any numbers.

## Project structure

| File | Responsibility |
| --- | --- |
| `src/main.cpp` | Command-line options and signal handling |
| `src/store.hpp`, `src/store.cpp` | Hash table, expiration, locking, append log, recovery |
| `src/http.hpp`, `src/http.cpp` | Socket server, HTTP parsing, worker queue, API routes |
| `src/json.hpp` | Small strict JSON parser and string encoder |
| `src/dashboard.hpp` | Embedded dashboard HTML, CSS, and JavaScript |
| `tests/integration.py` | Black-box tests, including restart/crash scenarios |
| `tools/benchmark.py` | Reproducible isolated local benchmark |

## Limitations and future work

The database and log have no configured size cap, eviction policy, or log compaction. Old values remain in the append-only log after updates/deletes until a future compaction implementation removes them. Use small, non-sensitive development datasets and monitor memory and disk space. Do not delete or modify an active log.

This version has no replication, transactions spanning requests, Redis wire protocol, lists/sets, or distributed coordination. Good follow-on exercises are crash-safe log compaction, an LRU memory limit, atomic increment/compare-and-swap operations, and reducing contention without weakening durability.

See [VALIDATION.md](VALIDATION.md) for the recorded test environment, results, and uncompleted checks. Linux and the optional CMake build have not yet been validated.
