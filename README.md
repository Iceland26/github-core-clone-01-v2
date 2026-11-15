# github-core-clone-01

[![make.fun bounty](https://img.shields.io/badge/make.fun-bounty-7C3AED)](https://make.fun/bounty/github-core-clone-01)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

Built for the [github-core-clone-01 bounty on make.fun](https://make.fun/bounty/github-core-clone-01) — a standalone, GitHub-compatible git forge competition ($500 prize pool, 20 paid places).

A small, self-contained, GitHub-compatible git forge written in TypeScript for
Node.js 22+. It uses **only the Node.js standard library** (`node:http`,
`node:zlib`, `node:crypto`): no npm dependencies, no `git` binary, and no
filesystem storage. All state lives in memory for the lifetime of the process.

## Surfaces

The forge serves four API surfaces from a single HTTP server:

| Surface | Path | Module |
| --- | --- | --- |
| REST v3 (GitHub-compatible) | `/api/v3/...` | `app.ts` |
| GraphQL | `/api/graphql` | `graphql.ts` |
| Git smart HTTP (clone / fetch / push) | `/{owner}/{repo}.git/...` | `git.ts` |
| Model Context Protocol (MCP) | `/mcp` | `mcp.ts` |

`GET /healthz` returns `{"ok":true}` and needs no authentication. Every other
route requires `Authorization: Bearer $MAKEFUN_SHOP_TOKEN` when that variable is
set; otherwise the server replies `401 {"message":"Bad credentials"}`.

### REST v3 coverage

- Root, `meta`, `rate_limit`, `user`, `user/repos` (list with `Link`
  pagination, create)
- Repositories: get, update, delete
- Labels: create, list, get, delete
- Issues: create, list (state / sort / direction / pagination), get, update,
  comments, labels
- Contents: get file or directory, create / update (with stale-sha `409`),
  delete. Contents writes create real git blobs, trees and commits, so the
  repository can be cloned over git afterwards.
- Git data: refs (create / get / list / update / delete), blobs, trees, commits
- Branches and commits (ancestry walk, pagination)
- Pull requests: create, list, get, update, files, commits, merge (squash by
  default, with three-way conflict detection returning `405`)

Errors are always JSON objects of the form `{"message": "..."}`, and every
REST response carries an `X-GitHub-Request-Id` header.

## Running

```sh
MAKEFUN_ENV_URL=http://controller.example \
MAKEFUN_SHOP_TOKEN=secret \
npm start
# equivalent to: node --experimental-strip-types main.ts
```

On start-up `main.ts`:

1. Listens on an ephemeral port (`listen(0)`).
2. Registers itself with `POST $MAKEFUN_ENV_URL/register {"port": N}`,
   retrying with exponential backoff (250 ms doubling, capped at 2 s).
3. Polls `GET $MAKEFUN_ENV_URL/done` with the same backoff until it answers
   `{"done": true}`, then shuts down cleanly.

The source uses only erasable TypeScript syntax (no enums, namespaces or
parameter properties) so it runs directly with `--experimental-strip-types`.

## Determinism

Responses are deterministic for a given sequence of requests:

- No wall-clock time and no randomness is used in any returned value.
- Timestamps come from a logical clock: a base epoch (`MAKEFUN_TRIAL_NOW` when
  set, otherwise `1735689600`, i.e. 2025-01-01T00:00:00Z) plus a counter that
  advances once per created or mutated object.
- Ids come from monotonically increasing counters (repositories start at
  1001), and node ids are base64 encodings of the object kind and id.
- Git object ids are real SHA-1 hashes of canonical git objects, and commit
  timestamps use the logical clock, so identical request sequences produce
  identical commit shas.

## License

MIT — see [LICENSE](LICENSE).
