# lib-http-router

`http-router` — the HTTP routing layer between `serve`
(lib-serve/serve-epoll) and Handl handler arrows, in pure handl, built
on `lib-route` (trie compile, specificity-ranked dispatch,
`match_path`, `compile_pat`). Named route maps, method+path dispatch
with `:params`/`*wildcards`/optional groups, 405s, href generation,
query parsing, JSON body decode.

## Layout

| path | role |
| --- | --- |
| `src/index.hnl` | the router |
| `src/index.test.hnl` | tests |
| `pkg.hnl` | manifest — deps `json`, `route` (`branch:main`) |
| `scripts/hnl-gate.sh` | `.hnl` gate |
| `workflows/gate.yaml` | CI — shared `handl-gates` campaign |

## Contract

- `serve` hands the handler a parsed `lib-http-rfc` request record;
  the router dispatches to handler arrows and returns the response
  record — the shapes are the interop contract.
- Pure `.hnl`, no Rust crate.

## Verification

```bash
HNL=../host-cli/target/debug/hnl scripts/hnl-gate.sh
```

<!-- myco:agent-awareness -->

## Agent awareness

- Session start / any work discussion: run `myco status --prompt`
  before proposing next steps — know the board first.
- User describes new work → offer to capture it (`myco todo add`,
  `issue new`, `proposal new`). Don't wait to be asked.
- Picking up a task: `myco resume <ID>` is the briefing.
- Wrapping up: `myco handoff <ID>` anything resumable; keep statuses
  honest (claimed → in-progress → blocked/done).

<!-- /myco:agent-awareness -->
