# lib-http-router

`http-router` — the HTTP routing layer between `serve` (lib-serve /
serve-epoll) and Handl handler arrows, in pure handl, built on
[`lib-route`](https://github.com/handl-lang/lib-route) (the generic
route-pattern library: trie compile, specificity-ranked dispatch,
`match_path`, `compile_pat`). This is the missing piece flagged in
P-004: `serve`/`serve-epoll` provide the accept loop and `http-rfc`
the wire codec, but routing lived inline in the handler. Now it is a
library.

`serve` hands the handler a parsed request record
`{ method, target, path, query, version, headers, body, body_bytes }`
(lib-http-rfc shape) and expects a response record back
`{ status?, headers?, body? | body_bytes? }`. `http-router` dispatches
on method + path pattern, remix-style.

## v2 — named route maps

`route/1` builds a route-map record `{ entries, by_name }` from a
record of named leaves and nested maps. The key path becomes the URL
path; `"."` (or `""`) is the index/root route.

```handl
from "http-router" import * as rt
from "serve-epoll" import { serve }

app := rt.route({
  ".":      index_h,                          -- GET/POST/... "/"  (any method)
  health:   { get: health_h },                -- method map -> GET "/health"
  users: {
    ":id":   user_h,                          -- nested -> "/users/:id"
    "new":   { get: new_h, post: create_h }   -- method map -> "/users/new"
  },
  login:    rt.form("/login")({               -- remix GET+POST pair
               get: login_form_h, post: login_submit_h }),
  files:    { "*rest": file_h }               -- "/files/*rest" -> params.rest
})

serve(rt.router(app), { port: 8080 })
```

Leaf forms:

- **handler arrow** — any method (`"*"`), path derived from keys.
- **method map** `{ get: h, post: h, ... }` — one entry per method key
  (case-insensitive, stored uppercased) at the same path. Must be
  *pure*: only HTTP-method keys plus `path`/`pattern`/`method`/
  `methods` config keys — anything else makes the record a nested map.
  Method names are reserved keys inside route maps.
- **spec** `{ method?, methods?, path?, pattern?, handler }` —
  explicit; `method`/`methods` take a string or list (uppercased),
  absent means `"*"`; `path`/`pattern` overrides the derived path.
- **nested record** — recurses, keys join with `/`.

Entry `name` = key path joined by `/` (`"index"` for root) — the
`by_name`/`href` lookup key.

The legacy `{ method?, path, handler }` **list form still works**:
`route(list, req)` and `router(list)` accept it unchanged (a list
passed to `route/1` wraps as `{ entries, by_name: {} }`).

## Dispatch semantics (v2)

- **Specificity ranking** — among path matches, only method-compatible
  entries are eligible; the winner is the highest score: static segment
  = 2, `:param` = 1, `*`/` *name` = 0, compared left-to-right; a strict
  score-prefix wins (exact `/a/b` beats `/a/b/*`). Equal scores keep
  list order. Order in the source no longer decides.
- **405 Method Not Allowed** — path matches but method doesn't →
  `{ status: 405, headers: { allow: "GET, POST" }, body: ... }` where
  `allow` is the deduped, encounter-ordered methods of the
  path-matching entries (comma-joined string — the wire codec's header
  shape). No path match at all → `not_found` (404).
- **`*name` captures** — a trailing `*name` segment binds the
  `/`-joined remainder to `req.params.name` (`*` alone still binds
  nothing). `*`/` *name` anywhere but last never matches.

## Surface

- `match_path(pattern, path)` — segment-wise match → params record or
  `null`. Literals match exactly; `:name` binds one segment into
  `req.params`; trailing `*`/`*name` consumes the remainder.
- `query(qs)` — query-string → record. Repeated keys accumulate into
  lists; `%XX` and `+` are decoded; bare `flag` gets `""`.
- `route(x)` — arity 1: build a route-map record from `{name: leaf |
  nested}` (or wrap a legacy list).
- `route(routes, req)` — dispatch; most-specific method+path match
  wins, handler gets `req` merged with `params` + `query_params`.
  Miss → `not_found` (404) or 405 as above.
- `router(routes)` — `(req) ~> resp`, plug straight into `serve`.
  Accepts a legacy list, a route-map record, or a raw map (built once).
- `href(pattern|entry, params)` — reverse URL generation: `:var` and
  `*name` substitution, e.g. `href("/users/:id", {id:"42"})` →
  `"/users/42"`. Missing param → `error.raise`; extra params ignored;
  substitution is raw (callers percent-encode).
- `href(map, name, params)` — named lookup via `by_name`, e.g.
  `href(app, "users/:id", {id:"9"})`.
- `form(pattern)` — returns `(handlers) ~> { path: pattern, ...hs }`,
  a method-map leaf pairing `get`/`post` at one URL (remix convention);
  `form(pattern, hs)` merges immediately.
- `json_body(req)` — `json.parse(req.body)`, `null` when empty.
- `not_found` — the default `{ status: 404, body: "not found" }`.

Path patterns support the full lib-route grammar: literals, `:param`,
trailing `*`/`*name` splats, v3 optional groups `(/seg)`, suffix/prefix
vars (`:file.:ext`), and `?k=v` search constraints (a constraint miss
is a 404, not a 405). `match_path(pattern, path)` delegates to
lib-route's matcher.

The custom-fallback story is a trailing
`{ method: "*", path: "*" }` entry — note it now ranks *least*
specific, so it only catches what nothing else did, and a method
mismatch on it still 405s only when its own method doesn't match.

## Layout

```text
src/index.hnl        the router (pure handl; deps: route (lib-route),
                     str, char, typeof, hash_map, json — all builtin
                     kit modules except route)
src/index.test.hnl   89 tests: match_path, query, route/router,
                     route maps, href, form, specificity, 405, json_body
pkg.hnl              package manifest — modules, deps, license
workflows/gate.yaml                bupha entry (handl-gates) → scripts/hnl-gate.sh
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
