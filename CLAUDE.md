# MyApp — forui-fullstack starter

This project was created from the `forui-fullstack` template:
[github.com/forailang/forui-fullstack](https://github.com/forailang/forui-fullstack).

It is a fullstack [forai](https://github.com/forailang/forai) app
using the [forui](https://github.com/forailang/forui) framework. One
`src/` tree, two platform entry points (`web` + `server`), RPC
across the boundary, SQLite for storage.

## forai is NOT (foreign-syntax bites)

Patterns from JS/Rust/Swift/Python that don't parse here:

- **No inline `if`.** `if cond then a else b` and `let x = if ...`
  both fail. For a value, lift to a `var` and assign inside a
  multi-line `if ... else ... end` block.
- **No inline `try`.** `try expr()` fails. Wrap fallible calls in
  `try ... catch e ... end`.
- **One `use` per line.** `use a, b, c` fails. Multiple modules →
  multiple lines. Multiple symbols from one module →
  `use { x, y } from M`.
- **Single quotes are strings, double quotes are templates.**
  `'hello'` is a plain string; `"hello, ${name}"` interpolates.
  Backslash-escaped quotes (`\"...\"`) don't work in either.
- **`useSignal(null)` / `useSignal([])` won't infer the element
  type.** Pass a typed default that matches the loader's return:

  ```fai
  let defaultPost = Post(id: 0, title: '', body: '', slug: '', created_at: '', updated_at: '')
  var post = useSignal(defaultPost) do
      getPost(slug)
  end
  ```

- **`assert` only exports `equals`, `isTrue`, `isFalse`, `isNull`,
  `isNotNull`.** No `notEquals` / `atLeast` / etc. — run
  `fai doc assert` for the surface.
- **Doc comments are required.** Every `def` / `remote def` /
  `test` needs `# Description.` directly above. Scan every file
  before saving — missing one is a hard type error.

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

## forai workflow

forai works best with **one function per file**. Files are free —
even ten files for ten trivial functions is fine. Group only when
the functions are genuinely tied (e.g. small helpers always used
together).

### Documentation

Every named `def`, `remote def`, and `test` block needs a doc
comment. Missing one is a type error: `Function 'X' is missing a
required doc comment`. `main` is the only exemption.

A one-line `# Description.` is the floor. Prefer richer docs that
answer:

- **What** does the function do?
- **How** does it do it (briefly — when the algorithm isn't obvious)?
- **What is it used for** in the project?
- **Example usage**, especially for tricky inputs.

You're writing for future-you and other agents. Generous docs pay
off the next time someone (you) is grepping for how a thing works.

`fai doc <query>` is the lookup tool — it indexes the language,
stdlib, every dependency, and your project's own doc comments.
Use it before guessing whether a function exists or what it
returns.

```bash
fai doc lang.modules     # how imports work
fai doc Forui.signal     # the Signal type and its hooks
fai doc std.string       # string helpers (trim, replace, …)
fai doc Forsqlite        # the SQLite driver this project uses
fai doc formatName       # your own functions, once you write them
```

### fai commands and when to use each

| Command | Pipeline | Use when |
| --- | --- | --- |
| `fai fmt` | fmt | Just to format. Rarely needed by hand. |
| `fai check` | fmt → check | Quick syntax / type sanity while editing. |
| `fai test` | fmt → check → test | **Default for inner-loop work** — runs every `test` block. |
| `fai run` | fmt → check → test → run | Boot the server / run the app locally. |
| `fai build server` / `fai build web` | fmt → check → test → build | Catches per-target codegen issues `fai test` misses. Run before declaring done. |
| `fai doc <query>` | — | Look up an API. Always cheap. |

`fai check` alone passes more often than the project actually
builds. **Definition of done = all four green:** `fai check`,
`fai test`, `fai build server`, `fai build web`.

### TDD example: building `formatName`

Cadence is **red → green → refactor**, one function at a time.
Worked example.

**1. Research the APIs you'll need.** Walk down the namespace tree
with `fai doc` before writing code. Faster than guessing function
names and finding out they don't exist via `fai check`.

```bash
fai doc                       # list every root namespace
fai doc std                   # list std submodules
fai doc std.string            # list string functions and signatures
fai doc std.string.toLower    # see one function's full signature + import line
fai doc std.string.trim       # confirm the helper you plan to call exists
```

The output of `fai doc std.string.toLower` literally shows the
`use { toLower } from std.string` line you need — copy it.

**2. Stub the function with docs and a failing test.**

```fai
# src/util/formatName.fai
use std.string

# Collapse repeated spaces and trim a name for display.
# Trims leading/trailing whitespace, then replaces any run of two
# spaces with one until none remain. Used by topbar greetings and
# post bylines so user-typed names render cleanly regardless of
# stray whitespace.
# Example: formatName('  Brian   Bal ') == 'Brian Bal'.
def formatName
    @param raw String
    @return String
do
  raw                                  # placeholder
end

test formatName
it 'collapses repeated spaces between words'
  assert.equals(formatName('Brian   Bal'), 'Brian Bal')
end
end
```

**3. Run the gate. Test fails — red.**

```
$ fai test
  formatName
    ✗ collapses repeated spaces between words
        expected 'Brian Bal', got 'Brian   Bal'
```

**4. Write the implementation. Test passes — green.**

```fai
do
  var s = string.trim(raw)
  while string.contains(s, '  ')
    s = string.replace(s, '  ', ' ')
  end
  s
end
```

**5. Add edge-case tests. They fail — red again.**

```fai
it 'trims surrounding whitespace'
  assert.equals(formatName('  Brian Bal  '), 'Brian Bal')
end
it 'returns empty string for whitespace-only input'
  assert.equals(formatName('   '), '')
end
```

**6. Refine the implementation until edge cases pass.**

In this example the trim + while-loop already handles those edges
— `fai test` goes green on its own. When tests pass on the first
try after a real change, your initial design was broader than the
first test exercised; keep adding tests until you've genuinely seen
each failure mode go red before going green.

Ship a function only when `fai test` is green for every case.

## Testing

A `fai test` run executes every `test` block in one wasm pass against
a **single in-memory SQLite database**. Three consequences:

1. **State is shared across `test` blocks.** Rows another test
   inserted are visible to yours. Don't assume an empty db.
2. **Randomize unique fields.** Email, slug, title — anything with a
   `UNIQUE` constraint should mix in a random suffix so reruns and
   sibling tests don't collide:

   ```fai
   let n Int = math.floor(math.random() * 1000000.0)
   let email = 'login-' + toString(n) + '@example.com'
   ```

   See `src/auth/server.fai` and `src/data/users/findUserByEmail.fai`
   for the canonical pattern.
3. **`.env.testing` is not auto-loaded.** Tests run with no env
   loaded, so `getDb()` falls back to `sqlite::memory:`. Override
   manually with `env.load('.env.testing')` only if you need a
   file-backed test db.

### Component tests with remote-def signals

A component that does `var x = useSignal(initial) do remoteFn() end`
will trap inside `testMount` because the test runtime can't dispatch
RPC. For these, write a minimal `'is covered by routed app rendering'`
test (see `src/pages/task_detail.fai` for the pattern) and rely on
the `App` test in `routes.fai` to exercise the rendered tree.

### Testing pages that read route params

`routeParam('id')` only returns a value while a `Router` is actively
matching a `Route`. To test such a page, drive it through `App()`
after seeding the path:

```fai
setPathFromPlatform('/posts/my-slug')
let node = App()
```

Calling `PostDetailPage()` directly will get a null param and
usually trap on the unwrap.

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
- UFCS modifiers (`padding`, `background`, `fontSize`, …) chain
  directly off a `do...end` block. Both same-line and new-line
  chains parse:

  ```fai
  VStack do
      Label('hi')
  end.padding(12).background('#fafafa')

  VStack do
      Label('hi')
  end
    .padding(12)
    .background('#fafafa')
  ```

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
