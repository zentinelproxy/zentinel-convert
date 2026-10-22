<p align="center">
  <a href="https://zentinelproxy.io">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/zentinelproxy/design/main/marks/lockup/zentinel-lockup-dark.svg">
      <img alt="Zentinel" src="https://raw.githubusercontent.com/zentinelproxy/design/main/marks/lockup/zentinel-lockup-light.svg" height="52">
    </picture>
  </a>
</p>

<h3 align="center">Config converter</h3>

<p align="center">Converts nginx, HAProxy, Traefik, Caddy and Envoy configurations into Zentinel KDL, for anyone moving an existing proxy setup to Zentinel.</p>

<p align="center">
  <a href="https://zentinelproxy.io/converter/">Converter in the browser</a> &middot;
  <a href="https://docs.zentinelproxy.io">Docs</a> &middot;
  <a href="https://github.com/zentinelproxy">All repositories</a>
</p>

`zentinel-convert` detects the source format, parses it into an intermediate
representation (listeners, routes, upstreams), and writes Zentinel KDL. It also
reports where the source config does something a Zentinel agent would do (rate
limiting, auth, WAF-like rules) and can declare that agent for you. The same
code, compiled to WebAssembly, runs the [converter on
zentinelproxy.io](https://zentinelproxy.io/converter/).

The output is a starting point, not a finished config: check it with
`zentinel test -c <file>` and read it before you deploy it (see
[Limitations](#limitations)).

## Quick start

```sh
cargo install --git https://github.com/zentinelproxy/zentinel-convert
zentinel-convert convert nginx.conf -o zentinel.kdl
zentinel test -c zentinel.kdl
```

Or paste a config into [zentinelproxy.io/converter](https://zentinelproxy.io/converter/);
nothing leaves the browser.

## Install

The crate is not on crates.io. Install from Git:

```sh
cargo install --git https://github.com/zentinelproxy/zentinel-convert
```

or build from source:

```sh
git clone https://github.com/zentinelproxy/zentinel-convert
cd zentinel-convert
cargo build --release   # binary at target/release/zentinel-convert
```

## Usage

### convert

```sh
zentinel-convert convert nginx.conf                       # auto-detect, KDL to stdout
zentinel-convert convert --format haproxy haproxy.cfg     # name the format
zentinel-convert convert nginx.conf -o zentinel.kdl       # write to a file
zentinel-convert convert --follow-includes nginx.conf     # follow nginx include
zentinel-convert convert --agents auto nginx.conf         # declare detected agents
zentinel-convert convert --dry-run nginx.conf             # show, don't write
```

| Option | Default | Meaning |
|---|---|---|
| `-f`, `--format` | auto-detect | `nginx`, `haproxy`, `traefik`, `caddy`, `envoy` |
| `-o`, `--output` | stdout | output file |
| `--follow-includes` | off | follow `include` directives |
| `--max-include-depth` | `10` | include nesting limit |
| `--strict` | off | fail on unknown directives |
| `--agents` | `suggest` | `suggest` (report only), `auto` (declare high-confidence agents), `none` |
| `--agent-socket-dir` | `/run/zentinel` | directory used for declared agents' sockets |
| `--comments` | off | add explanatory comments |
| `--source-refs` | off | add source file references |
| `--dry-run` | off | print instead of writing |

`--format apache` is accepted by the CLI but there is no Apache parser yet; it
fails with "Failed to detect configuration format".

### analyze

Report agent opportunities without converting:

```sh
zentinel-convert analyze nginx.conf
zentinel-convert analyze --json haproxy.cfg
```

### detect

```sh
zentinel-convert detect mystery-config.txt
# nginx
```

## Example

nginx directives are read inside an `http { }` block, as nginx itself requires;
a file with bare `upstream`/`server` blocks at top level converts to an empty
config.

```nginx
http {
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

    upstream backend {
        least_conn;
        server 10.0.0.1:8080 weight=5;
        server 10.0.0.2:8080 weight=3;
        server 10.0.0.3:8080 backup;
        keepalive 32;
    }

    server {
        listen 443 ssl http2;
        server_name api.example.com;
        ssl_certificate     /etc/ssl/certs/api.crt;
        ssl_certificate_key /etc/ssl/private/api.key;

        location /api/v1/ {
            limit_req zone=api_limit burst=20;
            proxy_pass http://backend;
        }
    }
}
```

`zentinel-convert convert nginx.conf` produces:

```kdl
schema-version "1.0"

system {
}

listeners {
    listener "listener_443" {
        address "0.0.0.0:443"
        protocol "h2"
        tls {
            cert-file "/etc/ssl/certs/api.crt"
            key-file "/etc/ssl/private/api.key"
        }
    }
}

routes {
    route "route_api_v1" {
        matches {
            path-prefix "/api/v1/"
            host "api.example.com"
        }
        upstream "backend"
    }
}

upstreams {
    upstream "backend" {
        target "10.0.0.1:8080" weight=5
        target "10.0.0.2:8080" weight=3
        target "10.0.0.3:8080" backup=#true
        load-balancing "least_connections"
        connection-pool {
            max-idle 32
        }
    }
}
```

and reports two high-confidence rate-limit opportunities (`limit_req_zone`, and
`limit_req` on `route_api_v1`). Note `backup=#true`: Zentinel does not read a
`backup` property on targets, so `10.0.0.3` becomes an ordinary target that
receives traffic.

## Agent detection

| Source pattern | Agent |
|---|---|
| `limit_req`, `limit_req_zone` and equivalent rate limits | [Rate limit](https://registry.zentinelproxy.io/ratelimit/) |
| `auth_basic`, `Authorization` header checks (Basic, Bearer) | [Auth](https://registry.zentinelproxy.io/auth/) |
| IP allow/deny lists, scanner User-Agent blocks (sqlmap, nikto, ...), Referer/Origin and Content-Type checks | [WAF](https://registry.zentinelproxy.io/waf/) |

With `--agents auto` the converter writes an `agents { }` block for each
high-confidence detection, but does not attach the agent to any route; add the
route's `agents` reference yourself, or the agent is never called.

## Supported formats

| Format | Files | What is read |
|---|---|---|
| nginx | `.conf` | `http` block: includes, upstreams, servers, locations, TLS, rate limiting |
| HAProxy | `.cfg` | frontends, backends, ACLs, stick tables |
| Traefik | `.yaml`, `.toml` | routers, services, middlewares (v2/v3) |
| Caddy | `Caddyfile` | sites, `reverse_proxy`, TLS, matchers |
| Envoy | `.yaml` | listeners, clusters, routes, filters |

## Limitations

Converting each fixture in `tests/fixtures/` and running `zentinel test -c` on
the result shows the gaps a converted config can have:

- TLS listeners can come out with a `tls { }` block that has no certificate,
  which Zentinel rejects: Caddy and Traefik sites that rely on automatic
  certificates, and HAProxy `bind ... ssl crt <file.pem>` (the combined PEM is
  not carried over). Add `cert-file`/`key-file` or an `acme` block.
- Envoy listeners can come out with `protocol "h2c"`, which Zentinel does not
  accept (`http`, `https`, `h2`, `h3`).
- nginx `root` locations become static-file routes that require the directory
  to exist on the machine that runs `zentinel test`.

## WebAssembly

The `crates/wasm` crate (`zentinel-convert-wasm`) is what the site's converter
loads:

```sh
cd crates/wasm
wasm-pack build --target web --release
```

It exports `convert(config, format?)`, `detect_format(config)`,
`validate(config, format?)`, `get_supported_formats()` and `get_version()`.

## Library

```rust
use zentinel_convert::{convert, AgentMode, ConvertOptions};
use std::path::Path;

let result = convert(
    Path::new("nginx.conf"),
    ConvertOptions { agent_mode: AgentMode::Auto, ..Default::default() },
)?;
println!("{}", result.kdl_output);
for s in &result.diagnostics.agent_suggestions {
    println!("{:?} agent ({:?} confidence)", s.agent_type, s.confidence);
}
```

## Development

```sh
cargo build
cargo test          # unit tests plus insta snapshot tests in tests/golden_tests.rs
cargo insta review  # after changing parser or emitter output
```

## Related

- [Converter on zentinelproxy.io](https://zentinelproxy.io/converter/)
- [Configuration reference](https://docs.zentinelproxy.io/)
- [Zentinel](https://github.com/zentinelproxy/zentinel)

## License

MIT OR Apache-2.0, see [`LICENSE-MIT`](LICENSE-MIT) or [`LICENSE-APACHE`](LICENSE-APACHE), at your option.

---

<p align="center">
  <sub>Part of <a href="https://zentinelproxy.io">Zentinel</a>, the security-first reverse proxy. The shisa guards the gate.</sub>
</p>
