# http

An HTTP client for [Mettle](https://github.com/The-Mettle-Project/Mettle). It sits on top of `std/http` (the parsers and writers) and adds:

- **Connection reuse.** Idle keep-alive connections are kept per scheme, host and port.
- **Redirects.** 301, 302, 303, 307 and 308, with the right method and body rules.
- **Cookies.** A cookie jar that follows RFC 6265.
- **gzip and deflate.** Bodies are decoded with [Press](https://github.com/Lex-Dellimere/Press), with a size limit.
- **https.** TLS through [mtls](https://github.com/Lex-Dellimere/Mtls). Certificates are checked.

`std/http`, `std/http2`, `std/websocket` and the rest of the standard library stay where they are. This package only needs `std/http`, `std/socket` and `std/time`.

## Install

In your `mbuild.mettle`:

```mettle
fn deps() {
  mb_dep_path("http", "../http");
}
```

http depends on mtls and Press. They are path dependencies for now (`../mtls` and `../press` next to this package). They become registry dependencies once they are listed.

## Use it

```mettle
import "http/http";
import "std/arena";
import "std/io";

fn main() -> int32 {
  var arena: Arena* = arena_init(0);
  var jar: HttpCookieJar;
  http_cookie_jar_init(&jar);

  var c: HttpClient;
  http_client_init(&c);
  c.jar = &jar;                       // optional: keep cookies

  match (http_client_get(&c, arena, "https://example.org/")) {
    case Ok(r): { println("{r->status} {r->body}"); }
    case Err(e): { println("failed: {e}"); }
  }

  http_client_free(&c);
  http_cookie_jar_free(&jar);
  arena_free(arena);
  return 0;
}
```

`import "http/http"` brings in everything. Import one module, such as `"http/cookie"`, to keep a build small.

| Module | What it has |
|---|---|
| `http/client` | `HttpClient`, `http_client_request`, `http_client_get`, `HttpReply` |
| `http/pool` | the idle connection pool: `HttpPool`, `HttpConn` |
| `http/redirect` | redirect rules and `http_url_resolve` |
| `http/cookie` | `HttpCookieJar`, `HttpCookie`, Set-Cookie parsing |
| `http/tls` | `HttpTlsConfig`, the https connection |
| `http/stream` | `HttpStream`, the byte stream the client reads and writes |
| `http/error` | the error codes the client adds to `HTTP_ERR_*` |

Every name starts with `http_`, `Http` or `HTTP_`. The names from `std/http` are unchanged, so you can use both.

## Documentation

- [docs/client.md](docs/client.md): the client, pooling, redirects, gzip, https
- [docs/cookies.md](docs/cookies.md): the cookie jar

## Things to know

- An `HttpClient` serves one thread at a time. Use one client per thread.
- Everything a request returns lives in the arena you pass.
- Idle connections are closed when they have been idle too long, but only when the client is next used or when you call `http_client_prune`. There is no background thread.
- `HttpStream` has the shape of the `NetStream` that is planned for `std`. When `std` has it, the client will use that type instead.
- There is no wss:// helper for the WebSocket client yet, and no HTTP/2 client. The client speaks HTTP/1.1.
- There is no public suffix list. The cookie jar refuses a `Domain` with no dot, such as `com`, but it cannot know that `co.uk` is a suffix.

## Tests

```bash
mbuild test       # debug build
mbuild test -r    # release build
```

The tests run against servers in the same process: a listening socket and a thread per connection (`tests/server.mettle`). They count accepted connections to show reuse, check the requests the server received, and use a local mtls server with test certificates for https (`tests/data/tls`). Nothing leaves the machine.

## License

Apache-2.0. See [LICENSE](LICENSE).
