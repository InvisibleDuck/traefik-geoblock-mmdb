# traefik-geoblock-mmdb

A tiny [Traefik](https://traefik.io/) **local middleware plugin** that allow-lists
HTTP requests by **country**, reading a **MaxMind GeoLite2-Country `.mmdb`** file
directly.

- ✅ Reuses an existing GeoLite2-Country.mmdb (e.g. maintained by a `geoipupdate` container)
- ✅ **No external API calls**, no per-request lookups over the network
- ✅ **No IP2Location** data (and therefore none of its licensing constraints)
- ✅ **Zero third-party dependencies** — a small, self-contained MaxMind DB reader
  is included, so it loads under Yaegi as a *local* plugin with no `vendor/` and no build step
- ✅ Config keys compatible with `nscuro/traefik-plugin-geoblock`
  (`enabled`, `databaseFilePath`, `allowPrivate`, `allowedCountries`)

> This product includes GeoLite2 data created by MaxMind, available from
> <https://www.maxmind.com>.

## Configuration

| Key                    | Type       | Default | Description |
|------------------------|------------|---------|-------------|
| `enabled`              | bool       | `false` | Master switch. When `false`, requests pass through untouched. |
| `databaseFilePath`     | string     | —       | Path *inside the container* to `GeoLite2-Country.mmdb`. Required. |
| `allowedCountries`     | []string   | —       | ISO 3166-1 alpha-2 codes that are allowed (e.g. `CH`, `LI`). Required. |
| `allowPrivate`         | bool       | `false` | Allow private / loopback / link-local source IPs without a lookup. |
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
country is present in `allowedCountries`. An unknown country (IP not in the DB)
or a lookup error results in a **deny** (fail closed).

## Using it as a Traefik *local* plugin

Local plugins are read by Traefik from `./plugins-local/src/<moduleName>`
relative to its working directory (`/` in the official image).

**1. Mount this repo into the Traefik container** (docker-compose):

```yaml
services:
  traefik:
    volumes:
      - /path/to/traefik-geoblock-mmdb:/plugins-local/src/github.com/smiso/traefik-geoblock-mmdb:ro
```

**2. Declare the local plugin** (static config, e.g. `traefik.yml`):

```yaml
experimental:
  localPlugins:
    geoblock:
      moduleName: github.com/smiso/traefik-geoblock-mmdb
```

**3. Define middlewares** (dynamic config, e.g. `config.yml`):

```yaml
http:
  middlewares:
    geoblock-ch:
      plugin:
        geoblock:
          enabled: true
          databaseFilePath: /plugins-storage/GeoLite2-Country.mmdb
          allowPrivate: true
          allowedCountries:
            - CH
            - LI
```

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
