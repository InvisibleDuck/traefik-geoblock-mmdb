# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Traefik middleware plugin (`github.com/InvisibleDuck/traefik-geoblock-mmdb`) that allow-lists HTTP
requests by country, reading a local MaxMind GeoLite2-Country `.mmdb` directly. The entire
implementation is `geoblock.go`; `geoblock_test.go` is the only test file.

## Hard constraint: Yaegi compatibility

Traefik loads plugins by interpreting their source with **Yaegi**, not by compiling them.
Everything follows from that:

- **Stdlib only.** `go.mod` must stay dependency-free — no `vendor/`, no build step. This is why
  the MaxMind DB reader is hand-written in the second half of `geoblock.go` instead of using
  `oschwald/maxminddb-golang`.
- The package is `traefik_geoblock_mmdb` (underscores); the entry points Traefik requires are
  `CreateConfig() *Config` and `New(ctx, next, *Config, name) (http.Handler, error)`.
- Prefer plain, conservative Go. Generics, newer stdlib APIs, and clever reflection risk tripping
  the interpreter.
- `.traefik.yml` is the catalog manifest. Its `testData` is instantiated by the Traefik catalog
  validator on a machine with **no `.mmdb` present**, which is why it sets `enabled: false`.

## Commands

```sh
go vet ./...
gofmt -l .

# Full test run. The reader test needs a real database and self-skips without it.
GEOBLOCK_TEST_DB=/path/to/GeoLite2-Country.mmdb go test -v ./...

# Single test
go test -run TestNoDatabaseFollowsAllowOnError -v      # no database needed
GEOBLOCK_TEST_DB=/path/to/GeoLite2-Country.mmdb go test -run TestLookup -v

# Run the suite the way Traefik actually executes this code. REQUIRED before
# any release: native `go test` cannot catch interpreter-only failures.
go run github.com/traefik/yaegi/cmd/yaegi@latest test -v .
```

`TestLookupEUCountry` runs against `testdata/GeoIP2-Country-Test.mmdb`, MaxMind's
public Apache-2.0 test fixture (`test-data` in `maxmind/MaxMind-DB`) — unrelated to the
licensed GeoLite2 database `TestLookup` wants, and checked in so the suite has a
database-backed test that always runs.

Without a local Go toolchain, run the same thing in a throwaway container (see the header comment
of `geoblock_test.go` for the exact `docker run` invocation).

## Architecture

`geoblock.go` has two clearly separated halves, split by the banner comment around line 263:

1. **Middleware** — `Config`, `New`, `ServeHTTP`, `decide`, `database`, `clientIP`, `isPrivate`.
2. **Embedded mmdb reader** — `countryDB`, `openCountryDB`, `lookupCountry`, `readNode`, `decode`.
   A country-lookup-only subset of the MaxMind DB format (https://maxmind.github.io/MaxMind-DB/):
   it walks the binary search tree and decodes just enough of the data section to reach
   `country.iso_code`, falling back to `registered_country.iso_code`. Unneeded types (double,
   float, int32, bytes…) are skipped rather than decoded. `record_size` 24/28/32 are supported.

### Decision semantics (the core invariant)

`decide(req)` is a **pure predicate**: it performs no response I/O and never calls `next`. That is
what makes wrapping it in `recover()` safe — a panic can flip the answer without risking a
double-serve. Keep it that way; `ServeHTTP` stays the only place that writes.

The allow-list is strict, and `allowOnError` only covers the *undetermined* case:

| Situation | Result |
|---|---|
| Country resolved and in `allowedCountries` | allow |
| Country resolved but not allow-listed | **block, always** — `allowOnError` is ignored here |
| Private/loopback/link-local IP with `allowPrivate` | allow, no lookup |
| Country undetermined (IP not in DB, unparseable IP, decode error, panic, **or DB missing**) | `allowOnError` |

### Startup and database availability

A missing or unreadable `.mmdb` must **never** return an error from `New` — that makes Traefik
treat the middleware as nonexistent and breaks every router referencing it. `New` logs a warning
and continues with `db == nil`. `database()` then retries `openCountryDB` at most once per
`dbReloadInterval` (30s) under a mutex, so the plugin self-heals once the file appears (e.g. after
a `geoipupdate` sidecar's first run) without a Traefik restart. Genuine authoring mistakes — empty
`databaseFilePath`, empty `allowedCountries` — still fail fast in `New`.

### Source IP and its security property

`clientIP` defaults to the TCP peer (`req.RemoteAddr`), which is unspoofable on a directly-exposed
Traefik. A header is consulted **only** when the operator explicitly sets `clientIPHeader`. Do not
change this default or add automatic `X-Forwarded-For` handling: trusting a client-supplied header
on a directly-exposed Traefik turns the allow-list into a one-header bypass. The README's
security note documents this contract.

## Yaegi-only failure modes

Native `go test` passing proves nothing about how the plugin behaves in Traefik. The known
trap, and the reason `TestLookupEUCountry` exists:

- **Never return a concrete `bool` through `decode`'s `interface{}`.** Native Go compiles it as
  an ordinary interface conversion; Yaegi routes it through `reflect.Value.SetBool` on an
  `Interface`-kind value and panics. `case 14` therefore returns `nil` and skips the value, like
  the unused types in `default`. Booleans carry no payload bytes, so the offset is unaffected.
- This stayed hidden for a long time because `is_in_european_union` is the only boolean in a
  GeoLite2-Country record, and MaxMind omits the field entirely when it would be false. Only
  **EU-country** lookups reach that code path — a Swiss test IP never does. The symptom was not
  a crash but a silent one: `decide`'s `recover()` caught the panic, so every EU request quietly
  fell through to `allowOnError` instead of resolving its real country.

Treat any new value flowing out through an `interface{}` as suspect and re-run `yaegi test`.

## Conventions

- Logging goes through `log.Printf` prefixed `geoblock(<name>):`. Keep it sparse — `New` runs once
  per router instance, so per-instance logs get repeated and Traefik surfaces them as ERR
  regardless of severity (see commit 52a1168).
- Any change to `Config` needs matching updates in three places: the struct's doc comments, the
  README configuration table, and `.traefik.yml` `testData` if the new key is illustrative.
- Traefik resolves the plugin from a **public repo with a semver tag** plus the `traefik-plugin`
  GitHub topic. The README's install snippet pins a version, so bump it in the same commit as the
  release tag.
