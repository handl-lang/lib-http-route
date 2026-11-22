# lib-http-route

`http-route` — the routing layer between `serve` (lib-serve /
serve-epoll) and Handl handler arrows, in pure handl. This is the
missing piece flagged in P-004: `serve`/`serve-epoll` provide the
accept loop and `http-rfc` the wire codec, but routing lived inline in
the handler. Now it is a library.

`serve` hands the handler a parsed request record
`{ method, target, path, query, version, headers, body, body_bytes }`
and expects a response record back. `http-route` dispatches on
method + path pattern:

```handl
from "http-route" import * as rt
from "serve-epoll" import { serve }

routes := [
  { method: "GET",  path: "/health",    handler: (_req) ~> do { { status: 200, body: "ok" } } },
  { method: "GET",  path: "/users/:id", handler: (req) ~> do {
      { status: 200, body: "user " + req.params.id } } },
  { method: "POST", path: "/users",     handler: (req) ~> do {
      { status: 201, body: type.to_string(rt.json_body(req)) } } },
  { method: "*",    path: "*",          handler: (_req) ~> do {
      { status: 404, body: "not found" } } }   -- custom fallback
]

serve(rt.router(routes), { port: 8080 })
```

## Surface

- `match_path(pattern, path)` — segment-wise match → params record or
  `null`. Literals match exactly; `:name` binds one segment into
  `req.params`; `*` as the last segment matches the remainder.
- `query(qs)` — query-string → record. Repeated keys accumulate into
  lists; `%XX` and `+` are decoded; bare `flag` gets `""`.
- `route(routes, req)` — first method+path match wins; the handler gets
  `req` merged with `params` and `query_params`. Miss → `not_found`.
- `router(routes)` — `(req) ~> resp`, plug straight into `serve`.
- `json_body(req)` — `json.parse(req.body)`, `null` when empty.
- `not_found` — the default `{ status: 404, body: "not found" }`.

Method mismatches are 404s (not 405s); matching is case-sensitive and
`/`-exact. Arity is strict — there are no optional parameters, so the
custom-fallback story is a trailing `{ method: "*", path: "*" }` route.

## Layout

```text
src/index.hnl        the router (pure handl; deps: str, char, typeof,
                     hash_map, json — all builtin kit modules)
src/index.test.hnl   45 tests: match_path, query, route/router, json_body
pkg.hnl              package manifest — modules, deps, license
ci.sh                bupha entry → scripts/hnl-gate.sh
```

## Gate

```bash
HANDL_EXT_DIR=../.ci-target/dogfood/release \
HNL=../host-cli/target/debug/hnl bash scripts/hnl-gate.sh
```

The implementation uses the arrow dialect (`~>`), which the canonical
typecheck/lint tools skip — the gate reports it as dialect-skipped. The
real check is `hnl test` over `src/index.test.hnl` (needs
`HANDL_EXT_DIR` pointing at the extension `.so`s, as CI sets it):

```bash
HANDL_EXT_DIR=../.ci-target/dogfood/release \
../host-cli/target/debug/hnl test src/index.test.hnl
```

## License

MIT.
