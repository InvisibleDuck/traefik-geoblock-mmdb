# traefik-geoblock-mmdb

A tiny [Traefik](https://traefik.io/) **middleware plugin** that allow-lists
HTTP requests by **country**, reading a **MaxMind GeoLite2-Country `.mmdb`** file
directly.

- ✅ Reads a local MaxMind GeoLite2-Country `.mmdb` directly
- ✅ Allow-lists by ISO 3166-1 alpha-2 country code
- ✅ **No external API calls**, no per-request lookups over the network
- ✅ **Zero third-party dependencies** — a small, self-contained MaxMind DB
  reader is included, so it loads under Yaegi with no `vendor/` and no build step

> This product includes GeoLite2 data created by MaxMind, available from
> <https://www.maxmind.com>.

## Configuration

| Key                    | Type       | Default | Description |
|------------------------|------------|---------|-------------|
| `enabled`              | bool       | `false` | Master switch. When `false`, requests pass through untouched. |
| `databaseFilePath`     | string     | —       | Path *inside the container* to `GeoLite2-Country.mmdb`. Required. |
| `allowedCountries`     | []string   | —       | ISO 3166-1 alpha-2 codes that are allowed (e.g. `CH`, `LI`). Required. |
| `allowPrivate`         | bool       | `false` | Allow private / loopback / link-local source IPs without a lookup. |
| `allowOnError`         | bool       | `false` | Behaviour when the source country **cannot be determined** — IP not in the DB, unparseable client IP, a lookup/decoder error, **or the database file is missing/unreadable**: allow the request (`true`, fail-open) or block it (`false`, fail-closed). A country that **is** determined but not in `allowedCountries` is always blocked. |
| `disallowedStatusCode` | int        | `403`   | HTTP status returned for blocked requests. |
| `clientIPHeader`       | string     | `""`    | **Leave empty unless Traefik is behind a trusted upstream proxy.** If set (e.g. `CF-Connecting-IP`), the source IP is taken from this header instead of the TCP peer. See the security note below. |

### Source-IP / security note

By default the plugin uses the **TCP peer address** (`req.RemoteAddr`) as the
client IP. This is correct and unspoofable when Traefik is directly exposed to
the internet. **Do not** set `clientIPHeader` to `X-Forwarded-For` on a
directly-exposed Traefik — a client could then spoof its country and bypass the
allow-list. Only set `clientIPHeader` when a trusted reverse proxy / CDN sits in
front of Traefik and injects a reliable client-IP header.

Allow-list semantics: a request is allowed **only** if the resolved source
country is present in `allowedCountries`. When the country **cannot be
determined** (IP not in the DB, unparseable client IP, or a lookup error) the
request is **blocked by default** (`allowOnError: false`, fail closed); set
`allowOnError: true` to let such requests through instead (fail open). A country
that *is* resolved but not allow-listed is always blocked.

A **missing or unreadable database** is handled the same way and, importantly,
**never fails Traefik startup** (which would otherwise make the middleware "not
exist" and break every router using it). The plugin serves per `allowOnError`
and keeps retrying to load the database in the background, so it recovers
automatically once the file appears — e.g. after a `geoipupdate` sidecar's first
run.

## Installation

Traefik downloads the plugin from GitHub (via the Traefik Plugin Catalog) — there
is **no volume mount of the plugin** to set up. Prerequisites for Traefik to
resolve it: a **public** repository with a **semver tag**, the `.traefik.yml`
manifest (included here), and the `traefik-plugin` topic so the catalog indexes it.

**1. Declare the plugin** in the **static** configuration (`traefik.yml`):

```yaml
experimental:
  plugins:
    geoblock:
      moduleName: github.com/smiso/traefik-geoblock-mmdb
      version: v0.3.0
```

**2. Define the middleware(s)** in the **dynamic** configuration (`config.yml`),
pointing `databaseFilePath` at the `.mmdb` file inside the container:

```yaml
http:
  middlewares:
    geoblock-ch:
      plugin:
        geoblock:
          enabled: true
          databaseFilePath: /geoip/GeoLite2-Country.mmdb
          allowPrivate: true
          allowedCountries:
            - CH
            - LI
```

**3. Attach** the middleware to a router (via labels or file provider) as usual.

> The plugin reads the `.mmdb` **at runtime** from `databaseFilePath`, so the
> database file must be present in the Traefik container regardless of how the
> plugin code is loaded. The next section shows one clean way to keep it fresh.

## Keeping the database fresh with `geoipupdate` (example)

The plugin does not download or update the database itself. A simple, clean
setup is to run MaxMind's official [`geoipupdate`](https://github.com/maxmind/geoipupdate)
container alongside Traefik and share a volume that holds the `.mmdb` file. You
need a free [MaxMind account](https://www.maxmind.com/en/geolite2/signup) with a
license key.

```yaml
services:
  traefik:
    image: traefik:latest
    # ... your usual Traefik config ...
    volumes:
      # The shared GeoIP volume — the plugin reads the .mmdb from here.
      - ./geoip:/geoip:ro
    # databaseFilePath in the middleware then points at /geoip/GeoLite2-Country.mmdb

  geoipupdate:
    image: maxmindinc/geoipupdate:latest
    restart: unless-stopped
    environment:
      GEOIPUPDATE_ACCOUNT_ID: "<your-account-id>"
      GEOIPUPDATE_LICENSE_KEY: "<your-license-key>"
      GEOIPUPDATE_EDITION_IDS: "GeoLite2-Country"
      GEOIPUPDATE_FREQUENCY: "168"   # refresh interval in hours (weekly); 0 = run once
    volumes:
      # geoipupdate writes GeoLite2-Country.mmdb into this same directory.
      - ./geoip:/usr/share/GeoIP
```

How it fits together:

- `geoipupdate` writes/refreshes `GeoLite2-Country.mmdb` into `./geoip` on the host.
- Both containers mount `./geoip`, so Traefik sees the file at `/geoip/GeoLite2-Country.mmdb`.
- The middleware's `databaseFilePath` points at that path.
- On the very first start, give `geoipupdate` a moment to fetch the database
  before the plugin needs it (e.g. start `geoipupdate` first, or set
  `GEOIPUPDATE_FREQUENCY: 0` for a one-shot download).

Secrets (`GEOIPUPDATE_ACCOUNT_ID`, `GEOIPUPDATE_LICENSE_KEY`) are best supplied
via an `.env` file / `env_file:` rather than committed into compose.

## Testing the reader

A test validates the mmdb reader against a real database (see `geoblock_test.go`):

```sh
GEOBLOCK_TEST_DB=/path/to/GeoLite2-Country.mmdb go test -run TestLookup -v
```

…or, with no local Go toolchain, via a throwaway container:

```sh
docker run --rm \
  -e GEOBLOCK_TEST_DB=/db/GeoLite2-Country.mmdb \
  -v "$PWD:/src" -v "/path/to/geoip:/db" \
  -w /src golang:1.23 go test -run TestLookup -v
```

## License

MIT (see `LICENSE`). GeoLite2 data is © MaxMind and used under its own license.
