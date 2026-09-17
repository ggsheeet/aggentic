# GG → Go

Personal design patterns and project structure for greenfield Go web apps.
Use this as the checklist when spinning up a new boilerplate.

---

## Stack defaults

| Layer | Choice | Notes |
|-------|--------|--------|
| Language | Newest local Go (currently 1.26.x) | Match Dockerfile base image (`golang:1.26-alpine`) |
| Module path | Bare name matching the repo | Not `github.com/...` unless you need it |
| HTTP | Stdlib `net/http` + Go 1.22+ mux patterns | No chi/gin/echo by default |
| Templates | [templ](https://templ.guide) | Generated `*_templ.go` is **gitignored**; always `templ generate` |
| Frontend interactivity | HTMX 4 (no extensions) | Vendored under `public/js/`, not CDN |
| CSS | Hand-written: `reset.css` → `global.css` → `components.css` | No Tailwind/Bootstrap |
| Hot reload | Air | Dev Docker runs Air; prod runs a static binary |
| Env files | `godotenv` + `.env.dev` / `.env.prod` | Load `.env.dev` only when `ENVIRONMENT=development` |
| Logger | Custom colored `CustomLogger` | Not `slog` / zap by default |
| Containers | Multi-stage Dockerfile + `compose.dev.yml` / `compose.prod.yml` | App-only until a DB is needed |
| Database | Optional | When added: `database/sql` + sqlc + embed migrations. Engine is per-app. |

Keep third-party deps minimal. Prefer stdlib. Typical early deps: `templ`, `godotenv`. Add sqlc as a `go.mod` tool (like templ) only when you have queries.

---

## Top-level layout

```
.
├── main.go                 # Boot: env load, mux, middleware, ListenAndServe
├── middleware.go           # SecurityMiddleware chain (package main)
├── logger.go               # CustomLogger (package main)
├── static_dev.go           # //go:build dev  — os.DirFS("public")
├── static_prod.go          # //go:build !dev — //go:embed public
├── env/env.go              # PORT, ENVIRONMENT, APP_BASE_URL (+ Sync after godotenv)
├── app/                    # HTTP routes & handlers (web UI)
├── database/               # Optional: Open, Ping, Close, embed migrations, sqlc
│   ├── migrations/         # Numbered *.sql applied on boot
│   └── sqlc/               # Generated (committed; unlike *_templ.go)
├── sql/                    # Optional: sqlc inputs
│   ├── schema/schema.sql   # Schema snapshot (keep in sync with migrations)
│   └── queries/            # Query files
├── sqlc.yaml               # Optional: with sqlc
├── helpers/                # RenderPage, form validation, small shared utils
├── templates/
│   ├── component/          # Reusable templ fragments (as the app grows)
│   └── view/<feature>/     # Page-level .templ files
├── public/
│   ├── css/reset.css
│   ├── css/global.css      # Tokens, base elements, shared utilities
│   ├── css/components.css  # Layout, forms, modals (no page stylesheets)
│   └── js/htmx.min.js      # HTMX 4, vendored; no extension files
├── .air.toml
├── Dockerfile              # builder → dev → production
├── compose.dev.yml
├── compose.prod.yml
├── Makefile
├── .env.dev / .env.prod    # gitignored
├── go.mod / go.sum
└── README.md
```

**Omit until needed:** `api/`, `auth/`, `database/` + `sql/` + sqlc, i18n/`locales/`, Delve, a DB container in compose.

Larger apps add packages at the root (`api`, `auth`, `database`, …) and inject a shared `Logger` interface into each package via deps. Small boilerplates can keep logger in `package main` only.

---

## Static assets: `static_dev.go` / `static_prod.go`

This is non-negotiable.

| File | Build tag | Behavior |
|------|-----------|----------|
| `static_dev.go` | `dev` | `os.DirFS("public")` — live disk files |
| `static_prod.go` | `!dev` | `//go:embed public` + `fs.Sub` — baked into binary |

Shared API:

- `var publicFS fs.FS`
- `func public() http.Handler` → `http.FileServer(http.FS(publicFS))`

Wire in `main` / router:

```text
GET /public/  →  http.StripPrefix("/public/", public())
```

Asset URLs in HTML are always absolute: `/public/css/...`, `/public/js/...`.

**Air must build with `-tags dev`.** Prod Docker / `make build` use `-tags '!dev'` (or equivalent). Without the dev tag, you silently ship the embed path in local reload.

If you add i18n, mirror the same split for `locales/` (disk vs embed). Drop locales entirely until you need them.

---

## Boot sequence (`main.go`)

1. If `ENVIRONMENT == "development"` → `godotenv.Load(".env.dev")` then `env.Sync()` (package-level env vars are set at `init`; Sync refreshes after dotenv).
2. If the app has a DB: `database.Open(...)` — ping, migrate, fail boot on error — then pass the handle into `app` deps.
3. Build mux, register `app` routes, attach static handler.
4. Wrap with `SecurityMiddleware(...)`.
5. `ListenAndServe(":"+port, handler)` with `PORT` default `8080`.
6. Log with `CustomLogger` (`Starting server...`, fatal on listen error).

Compose injects env via `env_file` before process start; godotenv still helps host-side `make dev`.

Minimal env surface for a new app:

```env
PORT=8080
ENVIRONMENT=development   # or production
APP_BASE_URL=http://localhost:8080
```

Connection string / file path vars are app-specific; add them when a DB exists.

---

## HTTP / routing

- Stdlib mux with method+path patterns: `"GET /{$}"`, `"GET /login"`, `"POST /login"`.
- Keep web handlers in `app/` (`Router`, feature handlers).
- Thin handlers: parse → validate → render templ / redirect.
- Prefer `http.StatusSeeOther` for POST→GET redirects.

A typical entry: `GET /` redirects to `GET /login`.

---

## Database (when needed)

Not part of a hello-world stub. When persistence shows up, keep the **libraries and layout** stable; pick the **engine** per project (SQLite file, Postgres, …).

Prefer:

| Piece | Choice |
|-------|--------|
| API | stdlib `database/sql` (no GORM / Ent by default) |
| Queries | [sqlc](https://sqlc.dev) — `go tool sqlc generate`; commit `database/sqlc/` |
| Schema changes | Numbered `*.sql` **embedded** and applied on boot |
| Driver | Whatever the engine needs; keep `CGO_ENABLED=0` in Docker when the driver is pure Go |

Layout (same folders regardless of engine):

```
database/            # Open, Ping, Close, migrate; exposes sqlc.Queries
database/migrations/ # 000001_….sql — runtime source of truth
database/sqlc/       # generated; committed
sql/schema/          # snapshot for sqlc (keep in sync with migrations)
sql/queries/         # sqlc query files
sqlc.yaml
```

Prod image is binary-only: embed migrations (and sqlc output is compiled in). Do not copy a live database file into image layers. Add a compose DB **service** only if the engine is a server; a file-backed engine uses a path + volume.

New table = new migration + update `sql/schema` + query file + `sqlc generate`.

---

## Security middleware

A **security-only** chain (not language/auth gates):

- Request ID (`X-Request-ID` + context key)
- CORS (from `APP_BASE_URL`)
- Timeouts
- Body size limit
- Rate limit (stdlib token bucket or `golang.org/x/time/rate`)
- Security headers (CSP tuned to self-hosted assets — no Google Fonts if using system fonts)
- Content-Type checks on POST/PUT
- Request logging (skip `/public/`)

Keep middleware in `package main` for small apps. Grow into a dedicated package only if it gets heavy.

---

## templ organization

```
templates/view/<feature>/<page>.templ   → package <feature>
templates/component/*.templ             → package component
```

- Generated files sit beside sources: `login_templ.go` next to `login.templ`.
- **Gitignore `*_templ.go`.** CI and Docker must run `templ generate` before `go build`.
- Prefer `go tool templ` (tool directive in `go.mod`) so the CLI version tracks the module.
- Render via a tiny helper:

```text
helpers.RenderPage(w, r, pageComponent)  → page.Render(r.Context(), w)
```

Extract a shared **head component** (`templates/component/head.templ`) on day one, even for a stub. It owns the `htmx-config` meta tag, the HTMX script, and the base stylesheet links; pages pass a title and any page-local script as children. Asset links are always absolute `/public/...` paths.

---

## HTMX 4 + form validation

Behaviors this pattern depends on:

1. **4xx/5xx responses are swapped by default.** `htmx.config.noSwap` defaults to `[204, 304]`. Error bodies reach the DOM natively — no extensions.
2. **Attribute inheritance is explicit.** Nothing is inherited from a parent unless the parent uses the `:inherited` modifier (`hx-target:inherited="#foo"`). Put `hx-*` on the element that makes the request.
3. **`<hx-partial>` is the mechanism for multi-target and per-field error slots.**

Current names: `hx-disable` (submit-button locking), colon-namespaced events (`htmx:after:settle`, `htmx:after:request`), `htmx.config.includeIndicatorCSS`.

### Scripts

Vendor exactly one file: `/public/js/htmx.min.js` (HTMX 4). No extension files.

Load it from the **shared head component**, not per page:

```html
<meta name="htmx-config" content='{"defaultTimeout": 0, "includeIndicatorCSS": false}'>
<script type="text/javascript" src="/public/js/htmx.min.js" defer></script>
```

Configure via the `htmx-config` meta tag, not JS. Do **not** set `implicitInheritance` or `noSwap` — leave HTMX 4 defaults in place.

### Form markup

```html
<form id="auth_form" class="form_container"
      hx-post="/login"
      hx-target="#auth_form"
      hx-swap="innerHTML"
      hx-indicator="#spinner"
      hx-disable="#form_submit_btn">
```

- No `hx-ext`. No extension attributes.
- Use `hx-disable` for submit-button locking.
- Where a 4xx must **not** swap the main target, use `hx-status:4xx="swap:none"` / `hx-status:5xx="swap:none"` on the element.

### Empty error slots in the form

Always render placeholders (even when empty). They are the swap targets:

```html
<div id="email_error" role="alert" class="helper_message error text_xs"></div>
<div id="general_error" role="alert" class="badge_message error text_sm"></div>
```

Inputs keep `aria-describedby="email_error"`. Do **not** put `aria-invalid` in the template — see below.

### Validation result types (`helpers`)

```text
FormValidationResult { Errors []FormValidationError; Valid bool }
FormValidationError  { Field string; Message string }  // Message = stable i18n key
GetFormErrorMessage(result, field) string
```

Slice of `{Field, Message}`. `Message` is a stable key resolved at render time (`i18n.T(ctx, "errors."+key)` / a plain-English `FormErrorText` helper in stubs).

### Failure path

Centralize the write in one helper so every handler behaves identically:

```go
func RenderErrors(w http.ResponseWriter, r *http.Request, errors templ.Component, customErr error) error {
	w.Header().Set("HX-Reswap", "none")
	return RenderComponent(w, r, errors, customErr)
}
```

1. Return a **4xx**. Use `400` via a named error constant (`HTMX_FORM_ERROR = HTMXError("ERR_FORM_ERROR", "Errors handled by form", http.StatusBadRequest)`). `422` is equally valid; pick one and be consistent.
2. Keep `HX-Reswap: none`. A response containing only `<hx-partial>` tags leaves the main target untouched; the header makes the intent explicit and protects against a stray non-partial node.
3. Render an errors templ that emits **only `<hx-partial>` nodes**, one per field slot:

```html
{{ emailError := helpers.GetFormErrorMessage(validationResult, "email") }}
<hx-partial hx-target="#email_error">
  if emailError != "" {
    { helpers.FormErrorText(ctx, emailError) }
  }
</hx-partial>
```

Each form gets its own `XFormErrors(validationResult helpers.FormValidationResult)` component. Reserve `hx-swap-oob="true"` for the rare case of replacing a whole region (e.g. swapping a form header + button row on OTP step change) — not for field errors.

### `aria-invalid` is set client-side

`<hx-partial>` replaces a slot's *contents*, so attributes on the slot itself are not updated. Split the concern:

- **Visual state** is pure CSS off `:not(:empty)` (see CSS conventions) so error chrome survives with JS disabled.
- **`aria-invalid` on the control** is for assistive tech, and needs a small post-settle pass:

```js
document.addEventListener('htmx:after:settle', () => {
	setupFormErrorDetection();   // reads [role="alert"] slots, toggles aria-invalid on the matching input
});
```

Keep it in the page's module (`auth.js` / `admin.js` / `forms.js`). Because the styling does not depend on it, this pass is additive — if it fails to load, the UI still reads as invalid.

### Success path

- HTMX request → `HX-Redirect` header + `200`.
- Non-HTMX → `http.Redirect(..., http.StatusSeeOther)`.
- Stay-on-page success → `HX-Trigger` (for snackbars/refresh hooks) + a normal 200 fragment. No `HX-Reswap: none`.

---

## CSS conventions

### Load order

1. `reset.css` (element reset; `button { all: unset; … }`, `line-height: 1.15`, `outline: transparent` rather than `outline: 0`)
2. `global.css` (design tokens on `:root`, base `html`/`body`, shared utilities: `.text_*`, `label`/`.label`, `.helper_message`, `.badge_message`, `.custom_cta`, HTMX/spinner)
3. `components.css` (layout, modal, `custom_header`, `custom_card` + `card_header`/`card_body`, form_container, field_section / field_wrapper / single_column)

### Writing style (not BEM)

- **snake_case** block/class names: `custom_modal`, `custom_header`, `form_container`, `field_wrapper`, `custom_cta`
- **Stack modifiers on the same element** — e.g. `.layout_wrapper.centered`, `.main_container.page_wide`, `.form_container.danger`, `.custom_header.has_action`, `.custom_header.custom_header_sm`, `.custom_cta.compact`, `.field_wrapper.single_column`
- **Do not** nest component selectors under a page wrapper (no `.login_page .form_container`)
- Tokens as CSS variables: `--clr-*`, `--text-*`, `--br-*`, `--input-field-height`
- Prefer solid surfaces/borders for new UIs; avoid glassy gradient input chrome unless the product needs it
- System font stack is fine for boilerplate; Outfit/local fonts when branding demands it
- Skip Alpine / Lucide / extra JS frameworks until a feature needs them

### Typical shared pieces in `global.css`

- `:root` color + type tokens
- `.text_med` / `.text_semi`; size utilities `.text_normal` / `.text_sm` / `.text_xs`
- `label` / `.label` (global, not nested under `.form_container`)
- `.helper_message` (+ `.error`, `.warning`, `:empty` hide, fade-in) and `.helper_message_floating` (position-only, `.over` / `.under`)
- `.badge_message` (+ `.error`, `.warning`) for opaque general-error banners
- `.custom_cta` button + color variants; expose `--spinner-size` / `--spinner-stroke` per size
- `.htmx-indicator` / `.spinner` / `@keyframes`

Layout lives in `components.css`; personalization is a second class on the **same** element, not a page parent.

Shared layout primitives (never nested under `.main_content`):

- `.custom_header` — page/section titles. Stack `.has_action` when there is a trailing `.multi_action` row; wrap the title+description in a child `<div>` so the action cluster can sit on the opposite side. `.custom_header_sm` / `.custom_header_xs` for in-form headings.
- `.custom_card` — info surfaces only (not forms). Optional `.card_header` / `.card_body` / `.no_border` / `.clickable`.
- `.text_row` / `.text_list` / `.text_column` / `.text_and_icon` — small text stacks, not page wrappers.
- `.icon_btn` / `.relative` — tiny utilities. Skip `.radial_progress` until a feature needs a ring.

### Error styling

Error display is **content-driven**. Visibility comes from `:empty`, and input chrome comes from two selectors — a CSS-only path that works with JS disabled, plus the `aria-invalid` the post-settle pass sets:

```css
/* helper text hides itself when the slot is empty — no display toggling by attribute */
.helper_message {
  color: var(--clr-text-dim);
  &:empty { display: none; }
  &.error { color: var(--clr-error); }
}

/* red chrome: content-driven (no JS needed) OR aria-invalid (set post-settle for AT) */
.field_wrapper:has(.helper_message:not(:empty)) :is(input, textarea, select),
:is(input, textarea, select)[aria-invalid="true"],
.file_input[aria-invalid="true"] { border-color: var(--clr-error-alt); }

/* autofill needs its own override */
:is(input:-webkit-autofill, input:autofill)[aria-invalid="true"] {
  border-color: var(--clr-error-alt) !important;
}
```

Do **not** key anything off `aria-invalid` on the helper text. Visual state belongs to `:not(:empty)`; `aria-invalid` belongs on the control and exists for assistive tech.

### HTMX indicator CSS

Built-in indicator styles are disabled via `includeIndicatorCSS: false`, so own them:

```css
.htmx-indicator { display: none; }
.htmx-request { display: flex !important; }
.htmx-indicator.htmx-request ~ * { display: none; }   /* hide sibling label while loading */
.spinner { width: var(--spinner-size, 1.25rem); stroke-width: var(--spinner-stroke, 5); }
```

The sibling-combinator rule works for any CTA content without needing a wrapper class on the label.

### Tokens

- Grayscale is derived, not hand-picked hex: `oklch(<L>% var(--neutral-c) var(--neutral-h))` with `--neutral-c` / `--neutral-h` on `:root`. Changing the two knobs retints the whole ramp.
- Name text tokens by **surface**, not by size alone, when a value diverges per area (`--text-title-home` / `--text-title-admin`, not one `--text-title`).
- Keep `--input-field-height`, `--bw-*`, `--br-*`, `--form-*` grid tokens.

### Modern CSS in use

Nesting (`&`), `:has()`, `@starting-style` + `transition-behavior: allow-discrete` for `dialog` open/close fades, and `container-type: inline-size` where a component must scale to its slot. Not used: `@layer`, `@property`, `light-dark()`.

---

## Logger

`logger.go` shape:

- `NewCustomLogger(prefix, out)` wrapping `log.Logger`
- Methods: `Info`, `Warning`, `Error`, `Debug` (colored levels; Warning/Error/Debug include caller file:line)
- Package-level `var logger = NewCustomLogger("", nil)` in `main`

For multi-package apps, define a small `Logger` interface in each package and inject from `main`. Don’t invent a new logging stack for greenfield apps.

---

## Air

Tracked `.air.toml` expectations:

| Setting | Value |
|---------|--------|
| `cmd` | `go build -tags dev -o ./tmp/main .` |
| `pre_cmd` | `["go tool templ generate"]` (+ `"go tool sqlc generate"` once sqlc exists) |
| `include_ext` | include `templ`, `css`, `js`, `go` (add `sql` with sqlc) |
| `tmp_dir` | `tmp` |
| Delve / sqlc | Omit until needed |

`-tags dev` and `templ generate` are required. When using sqlc, commit generated Go and still regenerate on `.sql` changes.

---

## Docker

### Dockerfile stages

1. **builder** — `go mod download`, `go tool templ generate`,  
   `CGO_ENABLED=0 GOOS=linux go build -tags '!dev' -ldflags="-w -s" -o <bin> .`  
   (drop `CGO_ENABLED=0` only if the chosen DB driver requires CGO)
2. **dev** — install Air, copy source, `CMD ["air", "-c", ".air.toml"]`, expose app port
3. **production** — `alpine` (or similar) + CA certs, `COPY --from=builder` binary only, `CMD ["/bin"]`

Prod image does **not** need the `public/` tree on disk; embed handles it. Same for SQL: embed migrations; do not ship a writable database file in the image.

### Compose

- `compose.dev.yml` → `build.target: dev`, mount `./:/app`, `env_file: .env.dev`, port 8080
- `compose.prod.yml` → `build.target: production`, `env_file: .env.prod`, **no** source mount required
- Name services/images after the project
- Add a DB service (and volumes / `depends_on`) only when the engine is a separate server

---

## Makefile (tracked)

Useful targets:

```text
templ / generate  → go tool templ generate  (+ sqlc generate when sqlc exists)
build             → generate + go build -tags '!dev' -o bin/<name> .
dev               → ENVIRONMENT=development air -c .air.toml
docker-dev        → docker compose -f compose.dev.yml up --build
docker-prod       → docker compose -f compose.prod.yml up --build
```

---

## .gitignore essentials

```text
bin/
tmp/
*_templ.go
.env
.env.*
!.env.example   # optional
build-errors.log
```

Track `.air.toml` and `Makefile`. Do **not** commit real secrets in `.env.*`.
If the engine is file-backed, also ignore the data dir / `*.db*` — not `database/sqlc/`.

---

## Growth path (when the app stops being a stub)

Add in this order as real needs appear:

1. **Real auth** — replace login stub; cookies/sessions; keep the same HTMX validation UX
2. **`api/` package** — JSON handlers if you need a separate API surface
3. **DB** — when you need persistence: `database/sql` + sqlc + embed migrations; pick the engine then
4. **i18n** — `locales/` + embed/disk split like static
5. **Richer middleware** — language redirect, auth gates, admin IP allowlists
6. **Components** — extract `templates/component/` (head, CTAs, shared forms)
7. **Delve** — only if you debug inside Docker regularly

Do not pull the full surface into a new repo on day one.

---

## New project checklist

- [ ] `go mod init <repo-name>` on newest local Go
- [ ] `main.go`, `logger.go`, security `middleware.go`
- [ ] `static_dev.go` / `static_prod.go` + `/public/` route
- [ ] `env/` + `.env.dev` / `.env.prod` + godotenv in development
- [ ] `app` router + at least one templ page
- [ ] `helpers.RenderPage` + form validation helpers if forms exist
- [ ] Vendored HTMX **4** (single file) loaded from a shared head component + `htmx-config` meta; no extensions, no `hx-ext`
- [ ] `reset.css` + `global.css` + `components.css` (snake_case, stacked modifiers; no page stylesheets)
- [ ] `.air.toml` with `-tags dev` + `templ generate`
- [ ] Multi-stage Dockerfile + compose dev/prod
- [ ] Makefile + `.gitignore` (`*_templ.go`, `tmp/`, `bin/`, `.env*`)
- [ ] Smoke: `GET` page 200, validation 4xx renders `<hx-partial>` field errors without replacing the form, prod build embeds static

---

## Anti-patterns

1. Air without `-tags dev` → `static_dev.go` never compiles in.
2. Air without `templ generate` → `.templ` edits don’t rebuild UI.
3. HTMX extensions or attributes that are not part of this stack: `hx-ext`, `hx-target-error`, `hx-disabled-elt`, old event names like `htmx:afterSettle`, or relying on `hx-target` inheritance from a parent without `:inherited`. These fail **silently**.
4. Replacing the whole form on validation error instead of `<hx-partial>` field slots + `HX-Reswap: none`.
5. Setting `htmx.config.noSwap = [204, 304, '4xx', '5xx']` — that drops error bodies so field slots never update.
6. Styling errors off `aria-invalid` on the helper text: `<hx-partial>` swaps slot *contents*, so template-set attributes on the slot never update. Drive visuals from `:empty` / `:not(:empty)`, and put `aria-invalid` on the control from a post-settle JS pass — never make the visual state depend on that JS.
7. CDN fonts/scripts when prod binary should be self-contained.
8. Pulling DB/auth/i18n/sqlc into a hello-world boilerplate.
9. Ignoring generated `*_templ.go` in git **and** forgetting to generate in Docker/CI.
10. Baking a writable database file into image layers (if the engine is file-backed).
