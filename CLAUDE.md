# MyApp — forui-fullstack starter

This project was created from the `forui-fullstack` template:
[github.com/forailang/forui-fullstack](https://github.com/forailang/forui-fullstack).

It is a fullstack [forai](https://github.com/forailang/forai) app
using the [forui](https://github.com/forailang/forui) framework. One
`src/` tree, two platform entry points (`web` + `server`), RPC
across the boundary, SQLite for storage.

## Build, check, test

```bash
fai check                   # format + type check
fai test                    # run all inline tests
fai run server              # start the server (binds 0.0.0.0:3040)
fai run web                 # build the web bundle into build/web/
```

`fai check` runs `fmt` then `check`. `fai test` runs `check` first,
then every `test` block.

## Source layout

```
src/
  pages/         # screen components — TopBar + body, return ViewNode
  components/    # reusable layout pieces (Section, Nav, TopBar)
  data/          # server data: connection, queries, mutations (remote def)
  auth/          # signup / login / logout / restoreSession + client state
  util/          # pure helpers
  platforms/
    web/         # browser entry: routes.fai, main.fai, index.html, shell
    server/      # server entry: HTTP, RPC dispatcher, session middleware
db/
  migrations/    # SQL schema changes (users, sessions, tasks…)
public/          # static assets the user authors
build/           # generated artifacts (web/, server/)
tests/e2e/       # end-to-end tests; unit tests live inline in src/
```

A directory under `src/` is one forai module. Every `.fai` file in
the directory contributes to the same namespace, imported by the
directory name (e.g. `use { Section } from components`).

## RPC pattern

`remote def` functions live alongside ordinary defs in `src/data/` and
`src/auth/`. The client wasm never executes those bodies — the build
rewrites them to JSON RPC calls under `/fai/rpc`. On the server,
`requireSession()` enforces auth and reads the per-request session
populated by the `http:beforeRequest` subscriber from the
`fai_session` cookie.

```fai
# src/data/tasks/main.fai
remote type Task
  id Int
  text String
  done Bool
end

remote def addTask
    @param text String
    @return Task
do
  let uid = requireSession().userId
  ...
end

# src/pages/tasks.fai
use { Task, addTask } from data.tasks
addTask(input.value)        # → JSON RPC on the client; direct call on the server
```

Network errors throw with categorized messages (`network error: ...`,
`HTTP 500`, `invalid JSON in response`); auth/business errors throw
the message the server returned. Wrap calls in `try ... catch` when
you need user-facing error UI.

## Auth

`src/auth/server.fai` owns the cookie-backed session lifecycle.
`src/auth/state.fai` owns the client-side current-user signal. On
boot, `src/platforms/web/main.fai` calls `restoreSession()` so a page
refresh keeps the user signed in.

`requireSession()` is the gatekeeper for protected `remote def`s —
call it at the top of any handler that needs an authenticated user.
The `tasks` sample shows the pattern.

## Common edits

- **Add a page**: drop `src/pages/<name>.fai`, add a route in
  `src/platforms/web/routes.fai`. Each page renders its own `TopBar()`
  and wraps the body in a `VStack`.
- **Add a resource**: add a migration under `db/migrations/`, create
  `src/data/<resource>/main.fai` with `remote type` + `remote def`s,
  and add a page that imports them by their real module path.
## Conventions

- forui has no wildcard re-exports — import the symbols you use by
  name from the right module.
- Every named function needs a `test` block. For pages, a one-line
  root-kind check (`assert.equals(node.kind, 'VStack')`) is usually
  enough; richer tests build a tree with `testMount` and assert on
  shape with `findByKind` / `getProp`.
- Layout helpers (`Section`, `Card`, etc.) take a `Children` closure,
  **not** a `ViewNode` — children build inside the helper's frame.
- Signals must be declared at the top of a component function, not
  inside a conditional or loop — the framework tracks them by call
  order.

## Dependencies

`fai.toml` points at sibling source trees:

```toml
"file://../forui" = "0.1.0"
"file://../html-forui" = "0.1.0"
"file://../forsqlite" = "0.1.0"
```

Adjust if you move the project, or pin to git URLs once that is
supported.

## Reference

- forai language: <https://github.com/forailang/forai/blob/main/language.md>
- forui framework: <https://github.com/forailang/forui/blob/main/forui.md>
- Project structure rationale: <https://github.com/forailang/forai/blob/main/plans/forui-project-structure.md>
