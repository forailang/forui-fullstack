# forui-fullstack

A starter template for fullstack apps built with [forai](https://github.com/forailang/forai)
and [forui](https://github.com/forailang/forui). One source tree, a wasm
browser client, a native server, RPC across the boundary, an auth
skeleton with cookie-backed sessions, and a SQLite data layer.

## What you get

- **`src/platforms/web/`** — browser entry: router init, root mount,
  HTML shell, session restore on boot.
- **`src/platforms/server/`** — server entry: SQLite, RPC dispatcher,
  session middleware, SSR fallback, static serving from `build/web/`.
- **`src/auth/`** — `signup`, `login`, `logout`, `restoreSession`
  remote defs; client-side `setSession` / `clearSession` /
  `isLoggedIn`; `requireSession()` for guarded endpoints.
- **`src/data/`** — `users.fai`, `db.fai`, plus a `tasks/` sample
  showing the canonical `remote def` + same-module import pattern.
- **`src/pages/`** — `home`, `about`, `tasks`, `task_detail`, plus
  `auth/{login, signup, logout}`.
- **`src/components/`** — `TopBar`, `Nav`, `Section` — small reusable
  layout pieces.
- **`db/migrations/`** — `users`, `sessions`, `tasks` schema.
- **`.env.*.example`** — committed placeholder env files. Real
  `.env.dev` / `.env.prod` / `.env.testing` are gitignored — copy the
  `.example` versions before first run.

## Quick start

This template currently expects to be cloned next to `forai/`,
`forui/`, `html-forui/`, and `forsqlite/`. The `[dependencies]` in
`fai.toml` use `file://../forui` style relative paths.

```bash
cp .env.dev.example .env.dev   # one-time: copy placeholders to local env

fai check                   # format + type check
fai test                    # run all inline tests

# Run server (binds 0.0.0.0:3040, serves build/web/ + RPC at /fai/rpc)
fai run server

# Build the web bundle into build/web/
fai run web                 # or: fai build web
```

Then open <http://localhost:3040>.

## Customizing

1. Rename the project in `fai.toml` (`name = "MyApp"`) — the build
   uses this name for the wasm and server artifacts.
2. Replace the `tasks` sample with your own resource:
   - migration in `db/migrations/`
   - `remote type` + `remote def`s in `src/data/<resource>/main.fai`
   - page in `src/pages/<resource>.fai`
   - route entry in `src/platforms/web/routes.fai`
3. Update the brand label in `src/components/topbar.fai` and the page
   title in `src/platforms/web/index.html`.

## Project shape

Read the full reasoning behind the layout in
[plans/forui-project-structure.md](https://github.com/forailang/forai/blob/main/plans/forui-project-structure.md)
and the framework reference in
[forui/forui.md](https://github.com/forailang/forui/blob/main/forui.md).

## License

Apache-2.0. See [LICENSE](LICENSE).
