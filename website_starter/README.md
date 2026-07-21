# Melt Website Starter

A small multi-page website built with **Melt**, using a lightweight
Model-View-Controller structure and a fluent `Document` builder that
composes each page from HTML partials via **method chaining**.

## Structure

- **`main.melt`** — Entry point: imports `config/app.melt` and `routes.melt`, defines the `App` handler, calls `listen(app.port)`.
- **`config/app.melt`** — `AppConfig` / global `app`: `name`, `port`, `tagline`.
- **`core/route.melt`** — The `Route` class: `get(path, handler)` registers a route, `matchPath(pattern, actualPath)` matches `{param}` segments, `dispatch(path)` matches and invokes the right controller (see below).
- **`core/document.melt`** — The `Document` fluent builder (see below) and a shared `navDataFor(username, isLoggedIn)` helper for the navbar greeting.
- **`routes.melt`** — Imports `core/route.melt` and the controllers, then registers the route table with `Route.get(path, [Controller, 'method'])`.
- **`controllers/`** — One file per page (`home`, `about`, `contact`, `hello`) plus `not_found_controller.melt` for the 404 fallback. Each builds a page with `Document` and calls `setResponseBody(...)`.
- **`views/partials/`** — Reusable page fragments: `head.html` (doctype/head/`<body>`), `navbar.html`, `footer.html`, `foot.html` (`</body></html>`), plus `meta.html` and `social.html`, which `head.html` and `footer.html` pull in with `{{ include(...) }}` (see below).
- **`views/*.html`** — Per-page bodies: `home.html`, `about.html`, `contact.html`, `hello.html`, `404.html`.
- **`public/css/site.css`** — Stylesheet, served at `/css/site.css` via `servePublic(path)`.

## How to run

From the project root:

```bash
./build/melt examples/website_starter/main.melt
```

Then open:

- **http://localhost:8090/** — Home
- **http://localhost:8090/about** — About
- **http://localhost:8090/contact** — Contact
- **http://localhost:8090/hello/World** — Hello, with `World` captured as a route parameter (try changing it)
- Any other path — 404

## Laravel-style routing: `Route.get(path, [Controller, 'method'])`

The `Route` class itself lives in `core/route.melt`; `routes.melt` imports
it and registers routes the same way Laravel's `Route::get(...)` does,
just spelled with `.` instead of `::`:

```melt
Route.get('/', [HomeController, 'index']);
Route.get('/about', [AboutController, 'index']);
Route.get('/hello/{name}', [HelloController, 'greet']);
```

and `main.melt`'s `App.handle()` dispatches every request through it:

```melt
Route.dispatch(path);
```

Two melt features make this possible (see `examples/laravel_style_routes.melt`
for a smaller, standalone version):

- **`Class.method(args)`** — calling a method through a *class* (not an
  instance) invokes it with no `this` bound ("static" call), instead of
  the usual "unknown class property" error. `Route.get(...)` and
  `Route.dispatch(...)` are both plain `method`s on `Route`, just invoked
  without ever writing `Route()`. Melt classes are already first-class
  values, so `[HelloController, 'greet']` is simply the class value
  itself paired with a method-name string — no special "class reference"
  syntax needed.
- **`callMethod(controller, methodName, argsArray)`** — calls a method
  whose name is only known at runtime as a string. This is what lets
  `Route.dispatch` pull `[HelloController, 'greet']` out of the route
  table and actually invoke `.greet(name)` on a fresh
  `HelloController()`.

`Route.matchPath(pattern, actualPath)` splits both the route pattern and
the request path on `/` and compares them segment by segment; a
`{placeholder}` segment always matches and its value is collected (in the
order it appears in the pattern) into the `params` array that gets handed
to `callMethod` as positional arguments — which is why
`greet(name)` receives the right value for `/hello/{name}`.

**A scoping gotcha that will bite you if you extend this router:** melt
has no per-call variable scope — every `let` in every method writes to one
shared global variable map. `Route.dispatch`'s route-loop counter and
`Route.matchPath`'s segment-loop counter *must* have different names
(`routeIndex` vs `segIndex` here), because `dispatch` calls `matchPath`
from inside its own loop. If both had been called `i`, `matchPath`'s
`let i = 0; while (i < ...)` would silently overwrite `dispatch`'s loop
index every time a route's segment count happened to match the request's
(this is exactly the bug this file had until it was caught by testing —
`/about` and `/contact` both fell through to the 404 handler because
matching `/` first corrupted the outer loop's counter). There's no
compiler warning for this; when adding more nested method calls, give
their local variables names that won't collide with the caller's.

## The `Document` builder: composing a page with method chaining

Each controller builds its page as one chained expression — every
`append()` renders a partial with `renderView(path, data)`, appends it,
and returns `this`, so the next `.append()` can run right after it:

```melt
let page = Document()
    .append("views/partials/head.html", ["title" :=> "Home — " + app.name])
    .append("views/partials/navbar.html", navDataFor("Alex", true))
    .append("views/home.html", ["heading" :=> app.name, "tagline" :=> app.tagline])
    .append("views/partials/footer.html", ["year" :=> 2026, "siteName" :=> app.name])
    .append("views/partials/foot.html", objectCreate())
    .build();
setResponseBody(page);
```

`Document` itself is just:

```melt
class Document {
    method init() {
        this.html = "";
    }
    method append(viewPath, viewData) {
        this.html = this.html + renderView(viewPath, viewData);
        return this;
    }
    method build() {
        return this.html;
    }
}
```

Notes on writing your own chained builders like this in Melt:

- **Every method call needs matching arity.** Melt methods require the
  exact number of arguments declared — there are no optional/default
  parameters. That's why `append` always takes `(viewPath, viewData)`;
  pass `objectCreate()` (an empty object) for partials that need no data.
- **Map literals use `[ "key" :=> value ]`**, not `{ "key": value }`.
- **Avoid reusing a caller's variable name as a method's parameter name.**
  Melt has a single flat variable namespace (no per-call lexical scope) for
  class methods, so if a method parameter has the same name as a variable
  the *caller* passes in as a later argument in the same call, the
  parameter binding can clobber it before that argument is evaluated. Pick
  parameter names that won't collide with typical caller variables (e.g.
  `viewPath`/`viewData` rather than `path`/`data`).

## Blade-like templates

Templates are plain HTML with placeholders, rendered by the built-in
`renderView(path, data)`:

- `{{ name }}` — escaped output.
- `{!! name !!}` — raw output.

`data` is any object (a class instance or a `[ "k" :=> v ]` map literal);
its fields become the placeholder values.

## Template-level includes: `{{ include("file.html") }}`

Besides composing whole page sections in melt code with `Document`, a
template can pull in a smaller fragment directly in its own HTML:

```html
<head>
  <title>{{ title }}</title>
  {{ include("meta.html") }}
</head>
```

`head.html` uses this to include `meta.html` (Open Graph / description
meta tags), and `footer.html` includes `social.html` (the footer links).
A few things to know:

- **Resolution**: `{{ include("meta.html") }}` in `views/partials/head.html`
  looks for `views/partials/meta.html` first (relative to the including
  file), falling back to the usual view-path resolution if not found there
  — so `{{ include("views/partials/meta.html") }}` also works from anywhere.
- **Shared data**: an include sees the same data object passed to the page
  that (directly or indirectly) included it — `meta.html`'s `{{ tagline }}`
  is filled from the same data map passed to `head.html`.
- **Includes nest**: an included file can itself `{{ include(...) }}`
  another one (there's a depth guard against accidental cycles).
- **The include target must be a literal string.** `{{ include(pathVar) }}`
  is not supported — only `{{ include("...") }}` / `{{ include('...') }}`
  with the path written directly in the template.
- If you ever need the literal text `{{ include(...) }}` to show up on a
  page (like this README's sibling, `views/about.html`, does to explain
  the feature), write it with HTML entities — `&#123;&#123; include(...)
  &#125;&#125;` — otherwise the engine will try to actually include it.
