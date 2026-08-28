# Ollie Shop CLI Reference

The CLI binary is **`ollieshop`** (published as `@ollie-shop/cli`). All commands run in the developer's local project directory. Most agent-facing commands talk to Ollie's backend; the two interactive commands (`login`, `start`) drive the local UX.

## Conventions

- **Authentication** — every command except `login`, `help`, and `version` requires a logged-in session. Run `ollieshop login` first.
- **Project config** — most commands need an `ollie.json` (or `ollie.<stage>.json`) at the cwd containing `storeId` and optionally `versionId`. Use `ollieshop init` to create it.
- **Global flags** apply to every agent command:
  - `--output, -o json` — force JSON output (auto-set when stdout is piped).
  - `--dry-run` — validate arguments and print what would run, without mutating.
  - `--fields a,b,c` — limit output to these top-level fields.
  - `--data '{...}'` — pass the full payload as raw JSON instead of flags. Field names use camelCase.
  - `--stage, -s <name>` — load `ollie.<stage>.json` instead of `ollie.json` (e.g. `--stage dev`).
- **Discover any schema** — `ollieshop schema` lists every known resource. `ollieshop schema <resource>.<action>` prints the JSON Schema (e.g. `ollieshop schema store.create`). Useful when you need authoritative field names and constraints before scripting a mutation.
- **Git Bash on Windows** — argument values that begin with `/` get rewritten to Windows paths by MSYS path conversion. Use `--data '{...}'` (which carries the value inside the JSON string) to bypass this, or run the CLI from PowerShell / cmd.

---

## Authentication

### `ollieshop login`
Opens the browser to authenticate against Ollie's auth backend and stores credentials locally. Interactive (renders an Ink TUI).

Flags:
- `--port, -p <number>` — local callback port (default `7777`).

### `ollieshop whoami`
Prints the current user's email. Exits non-zero if not logged in.

---

## Local development

### `ollieshop init`
Writes an `ollie.json` (or `ollie.<stage>.json` with `--stage`) into the cwd with the chosen `storeId` and optional `versionId`. Run once per project per stage.

Flags:
- `--store-id <uuid>` (required) — store to associate with this project.
- `--version-id <uuid>` (optional) — pin a specific version.
- `--stage, -s <name>` (optional) — write `ollie.<stage>.json` instead of `ollie.json`.

Example:
```bash
ollieshop init --store-id 11111111-1111-1111-1111-111111111111 --version-id 22222222-2222-2222-2222-222222222222
```

### `ollieshop setup` (CLI ≥ 1.5.0)
Converts a project from the per-folder `meta.json` pattern to the root `ollie.json` `components` map. Both patterns are supported at runtime (see `references/component-anatomy.md` §Two registration patterns) — `setup` is the migration path when a project wants to switch from A to B. Scans `components/*/index.tsx`, reads each folder's `meta.json`, writes an entry (`"<meta.id>": { "path": "components/<name>", "slot": <meta.slot> }`) into `ollie.json` (or `ollie.<stage>.json` when `--stage` is passed), and **deletes the old `meta.json`**.

- Metas with no `id` (unlinked placeholders) are **skipped and left as-is**.
- `--dry-run` prints what would migrate without touching any files.
- Idempotent — running it again on an already-migrated project reports "No meta files to migrate."

```bash
ollieshop setup --dry-run -o json   # preview
ollieshop setup                     # migrate + remove meta.json files
```

Only run this if the project wants to switch registration patterns. Projects that prefer per-folder `meta.json` don't need it.

### `ollieshop start`
Boots the local dev server on **port 4000** and opens Ollie Studio (`https://admin.ollie.shop/studio`) in the browser pre-configured to consume the local components. Interactive TUI with hot reload (esbuild watch + manifest plugin). All component changes in `./components/<name>/` are picked up live.

**This is the primary day-to-day loop for component work**: iterate locally with Studio side-by-side, and ship from the Studio preview when you're happy (see "Deploying" below).

Behavior:
- Reads `ollie.json` (or `ollie.<stage>.json`).
- Discovers components by globbing `./components/*/index.tsx`.
- Builds them to `node_modules/.ollie/build/`.
- Serves them at `http://localhost:4000/<name>/index.js` and `http://localhost:4000/bundle?path=/<name>/index.js`.
- Studio iframes the local checkout and receives build events via SSE on `/esbuild`.

Flags:
- `--stage, -s <name>` — load `ollie.<stage>.json`.

Interactive keys: `q` quit, `r` rebuild, `o` re-open Studio.

If a component folder isn't linked to a database record — via either `meta.json` (pattern A) or the root `ollie.json` `components` map (pattern B) — `start` still builds it but flags it as **unlinked**. It can render in Studio but won't be placed in checkout slots until a real component is created via `component create` and the returned UUID is written where the project expects it. (Under pattern B, a component owned by a different stage's config shows up as `otherStage` in the manifest.)

---

## Stores

### `ollieshop store create`
Create a store under the authenticated user's organization (or a specific org with `--org`).

Flags (or use `--data '{...}'` with camelCase keys):
- `--name, -n <string>` (required) — store display name.
- `--platform, -p <vtex|shopify|vnda|custom>` (required) — vendor.
- `--platform-store-id <string>` (required) — platform-specific store identifier (e.g. VTEX account name).
- `--logo <url>` (optional).
- `--settings <json-string>` (optional) — raw JSON settings string.
- `--org <uuid>` (optional) — target organization; defaults to the user's default org.

```bash
ollieshop store create --name "My Store" --platform vtex --platform-store-id mystore
ollieshop store create --data '{"name":"My Store","platform":"vtex","platformStoreId":"mystore"}' -o json
```

### `ollieshop store list` (alias: `store ls`)
Flags:
- `--org <uuid>` (optional) — list stores in a specific org.

---

## Versions

A **version** is a snapshot of components + functions for a store. The CLI does not yet expose update/activate as separate commands; flip `active` / `default` on create.

### `ollieshop version create`
Flags:
- `--store-id <uuid>` (required).
- `--name, -n <string>` (required) — e.g. `v1`, `main`.
- `--active` (optional) — flag.
- `--default` (optional) — make this the default version.
- `--template <string>` (optional, nullable) — template name; defaults to the store's default template.

```bash
ollieshop version create --store-id <uuid> --name v1 --active
```

### `ollieshop version list` (alias: `version ls`)
- `--store-id <uuid>` (required).

---

## Components

### `ollieshop component create`
Registers a component in the database under a version. Code is uploaded later (see "Deploying" below).

Flags:
- `--version-id <uuid>` (required).
- `--name, -n <string>` (required) — component name (matches the folder name in `./components/<name>/`).
- `--slot, -s <string>` (required) — target slot id (see `references/slots-catalog.md`).
- `--active` (optional, default `true`) — accepts `--active` or `--active=false`.
- `--props <json>` (optional, default `null`) — initial props object, seeded at create time (see below). Must be a JSON object or `null`; `--props false` and `--props [1,2]` are rejected.

```bash
ollieshop component create --version-id <uuid> --name FreeShippingBar --slot cart_summary_header --props '{"threshold":199}'
```

After creation, link the returned `id` to the local folder. Which file you write to depends on the project's registration pattern (see `references/component-anatomy.md`):

- **Pattern A** — write `./components/<name>/meta.json` as `{ "id": "<uuid>" }`.
- **Pattern B** — add `"<uuid>": { "path": "components/<name>", "slot": "<slot-id>" }` to the `components` map in the root `ollie.json`.

If you're switching an existing project from A to B, `ollieshop setup` writes those entries for you.

#### Seeding `props` at create time

`component.create` sets a component's initial `props` — confirm with `ollieshop schema component.create`, which shows `props` as an optional `object | null`. To change props on an *existing* component, use `ollieshop component update --id <uuid> --props '{…}'` (or the Admin/Studio UI).

`ollieshop deploy` ships **code only** (the bundle), never props. Props are **not** carried in `ollie.json` (a component entry is just `{ path, slot }`) — the source of truth for props is the DB component record (set at create, edited later in the Admin). Studio pulls those record props when it renders the preview.

Full flow to ship a **new** component *with* props in one go:

```bash
# 1) register the component record (id + initial props)
ollieshop component create --data '{
  "versionId": "<version-uuid>",
  "name": "CouponSelector",
  "slot": "cart_coupon_form",
  "active": true,
  "props": { "coupons": [{ "code": "WELCOME10", "label": "Welcome 10%" }] }
}' -o json
# -> { "data": { "id": "<component-uuid>" } }

# 2) link that id to the local folder — one of:
#    A) write ./components/CouponSelector/meta.json  ("id": "<component-uuid>")
#    B) add "<component-uuid>": { "path": "components/CouponSelector", "slot": "cart_coupon_form" }
#       to ./ollie.json -> components

# 3) upload the code bundle
ollieshop deploy --component-id <component-uuid> --name CouponSelector --wait -o json
```

Verify the stored props afterwards with `ollieshop component list --store-id <uuid> --version-id <uuid> -o json` (each record echoes its `props`).

### `ollieshop component update`
Flags — `--id` plus at least one field to change; an update with nothing to set is rejected.
- `--id <uuid>` (required) — the component record to change.
- `--name, -n <string>` (optional).
- `--slot, -s <slot-id>` (optional) — moves the component to another slot.
- `--active true|false` (optional).
- `--props <json>` (optional) — replaces the stored props, and must be a JSON object. `--props null` clears them.
- `--version-id <uuid>` (optional) — moves the component to another version.

```bash
# fix a slot that was set wrong at create time
ollieshop component update --id <component-uuid> --slot cart_coupon_form -o json

# replace the stored props
ollieshop component update --id <component-uuid> --props '{"coupons":[]}' -o json
```

### `ollieshop component list` (alias: `component ls`)
Flags:
- `--store-id <uuid>` (required).
- `--version-id <uuid>` (optional) — filter to one version.

---

## Functions

Hub functions are HTTP middleware that run on the request or response of a target endpoint (excluding PCI-protected destinations — see `references/hub-functions.md` once that file is added).

A **trigger** is an object with two non-empty string properties:
- `url` — absolute http(s) URL or relative path starting with `/`. Identifies what endpoint the function attaches to.
- `expression` — a JSONata-like match expression evaluated against the incoming request/response. Example: `method in ["POST"]`. Conditions can be chained with `and`, e.g. `method in ["POST"] and headers."x-store" = "pharma"`.

The trigger is optional. If omitted, the function is created without one and won't run until a trigger is set via `function update` or the admin UI.

The **invocation** decides which phase the handler runs in:
- `response` (**default**) — the handler runs after the destination replies. It receives the upstream response and rewrites that. This is the common case.
- `request` — the handler runs before the call reaches the destination. It receives `res: null` and rewrites the outgoing request.

What must never happen is the column being left **null**: the hub's function query filters on `invocation is not null` (alongside `active`, `urn` and `trigger`), so a null-invocation function is never even loaded. It sits in listings looking healthy and never executes. The CLI therefore always writes a value — passing `--invocation` explicitly is still the right habit whenever the function is a `request` one.

### `ollieshop function create`
Flags:
- `--version-id <uuid>` (required).
- `--name, -n <string>` (required).
- `--invocation <request|response>` (optional, default `response`) — the phase the handler runs in. Pass it explicitly for `request` functions.
- `--trigger-url <url>` (optional) — the trigger URL. **Must be paired with `--trigger-expression`** — passing one without the other is rejected.
- `--trigger-expression <string>` (optional) — the match expression.
- `--active` (optional, default `false`) — accepted bare or as `--active true|false`. Any other value is rejected rather than ignored.
- `--on-error <throw|skip>` (optional, default `throw`) — what to do if the function throws.
- `--priority <integer>` (optional, default `0`) — execution order among siblings.

```bash
# without trigger (configure later via function update)
ollieshop function create --version-id <uuid> --name validatePrescription \
  --invocation request --on-error throw

# with trigger via flags
ollieshop function create --version-id <uuid> --name validatePrescription \
  --invocation request \
  --trigger-url https://mystore.vtexcommercestable.com.br/api/checkout/pub/orderForm \
  --trigger-expression 'method in ["POST"]' \
  --on-error throw --priority 10

# with trigger via --data (bypasses Git Bash path conversion on Windows)
ollieshop function create --data '{
  "versionId": "<uuid>",
  "name": "validatePrescription",
  "invocation": "request",
  "trigger": { "url": "/api/checkout/pub/orderForm", "expression": "method in [\"POST\"]" },
  "onError": "throw",
  "priority": 10
}'
```

### `ollieshop function update`
Patches an existing function record. At least one field flag is required; anything you don't pass is left untouched.

Flags:
- `--function-id <uuid>` (required; `--id` is accepted as an alias).
- `--name, -n <string>`, `--active <true|false>`, `--invocation <request|response>`, `--on-error <throw|skip>`, `--priority <integer>`, `--version-id <uuid>` (all optional).
- `--trigger-url <url>` + `--trigger-expression <string>` (optional) — same pairing rule as `create`. To *clear* a trigger, pass `--data '{"id": "<uuid>", "trigger": null}'`.

```bash
# fix a record that was created without a phase
ollieshop function update --function-id <uuid> --invocation request

# repoint the trigger and activate it
ollieshop function update --function-id <uuid> \
  --trigger-url '/api/checkout/pub/orderForm/:id/attachments/paymentData' \
  --trigger-expression 'method in ["POST"]' \
  --active true
```

There is deliberately **no `function delete`**, matching `component` — records are removed from the admin.

### `ollieshop function list` (alias: `function ls`)
Flags:
- `--store-id <uuid>` (required).
- `--version-id <uuid>` (optional).

Worth checking `invocation` in the output on any function created before CLI 1.12 or through older tooling: a `null` there is a function that will never run, and `function update --invocation` is now the fix.

---

## Deploying

There are two recommended paths to ship code to a store, **depending on resource type**:

### What actually ships — and how to trim it with `.ollieignore`

Read this before the first deploy. Everything in this section is about the CLI's bundler, which is what the Studio deploy button (it fetches `/bundle` from the local CLI) and `ollieshop deploy` both go through — for components **and** functions alike. It does not apply only if you upload a hand-built zip through the admin form, where `.ollieignore` has no say in what ships.

**The bundle is the whole project root, not just the component or function being deployed.**

Everything under the root rides along unless something drops it. What the CLI drops on its own is narrow: `node_modules/`, `.git/`, `dist/`, `build/`, `.next/`, and `*.d.ts` / `*.js.map` / `*.tsbuildinfo`, at any depth. Loose files sitting *directly* in the root are opted in by extension (sources, styles, images, fonts); **inside any subdirectory there is no extension filter at all** — every file is bundled. Nothing is dropped for being a dotfile, `.env` included.

The two ways this bites:

- **Silent weight.** A folder of QA screenshots, a HAR capture, design exports, a `docs/` tree, an unrelated `vtex-apps/` checkout — none of it is referenced by the component and all of it uploads. The Builder rejects the bundle over **10 MB** with `File size exceeds the maximum allowed size.`, which names no file, so the cause is not obvious from the error.
- **Secrets.** A root `.env` reaches the Builder unless excluded.

Fix both with a `.ollieignore` in the project root — gitignore syntax, matched against paths relative to the bundle root:

```gitignore
# QA / feedback assets
input/
qa-feedback-*/
docs/
*.png
*.har

# Unrelated sources
vtex-apps/

# Agent + editor scratch
.claude/
.vscode/
.turbo/
.cache/
coverage/

# Secrets
.env
.env.*
```

Patterns read exactly as in a `.gitignore`: write `public/screenshots/`, not `screenshots/`. Only the root file is read — nested `.ollieignore` files are ignored.

Three things it cannot do:

1. **Re-include a built-in exclusion.** The list above is applied before user patterns, so a `!` negation only works against your own patterns.
2. **Re-include a file under an excluded directory** — same restriction git has. `public/` followed by `!public/keep.png` keeps nothing; use `public/*` + `!public/keep.png`.
3. **Drop `package.json` or the generated entry point** (`index.ts` / `index.tsx`). The builder needs both; a pattern matching them warns and is otherwise ignored.

Verify a pattern before uploading — `--dry-run` reports the bundle size, the ignore file in effect, and what was excluded:

```bash
ollieshop deploy --component-id <uuid> --name FreeShippingBar --dry-run -o json
```

```json
{"dryRun": true, "action": "deploy", "data": {
  "bundleSizeBytes": 184320, "ignoreFile": ".ollieignore",
  "excludedCount": 12, "excluded": ["input/", "docs/"]
}}
```

`excluded` caps at 50 paths (`excludedTruncated: true` marks a longer list). Warnings go to **stderr**, so `-o json` on stdout still parses. Projects scaffolded by `create-ollie-shop` ≥ 1.4.0 already ship an `.ollieignore` covering the local caches and tooling directories; older projects have none until someone adds one.

### Components — deploy from Studio preview (recommended)
1. Run `ollieshop start` in the project directory.
2. Studio opens in the browser, iframing the live checkout with your local component bundled in.
3. Iterate. Hot reload re-bundles on every save.
4. When you're happy with the result, **deploy from Studio's preview UI** — it triggers a build of the local bundle and promotes it to the target version.

This is the path you should suggest first to any developer doing component work. It removes the manual zip/upload step and keeps the deploy gesture next to the visual validation.

### Functions — deploy from the CLI
`ollieshop deploy --function-id` works end to end. The bundler reads from `functions/<name>/` and generates an `index.ts` entry point that re-exports your `handler`, so the full loop is:

1. Put the function in `functions/<name>/`, exporting `handler` of type `CustomFunction` (shape documented in `references/hub-functions.md`). `@ollie-shop/functions` comes from the Lambda layer at build time, so keep it a normal `dependencies` entry — don't bundle it.
2. Create the record: `ollieshop function create --version-id <uuid> --name <label> --invocation request|response`.
3. Deploy the code: `ollieshop deploy --function-id <uuid> --name <folder> --wait`.
4. Set or correct the trigger with `ollieshop function update` once you know the exact endpoint.

```bash
ollieshop deploy --function-id <uuid> --name auchan-promissory-keeper --wait
# {"success":true,"data":{"id":"FunctionBuildProject-…","status":"SUCCEEDED","resource":{"type":"function"}}}
```

`--dry-run` is the cheap way to inspect what gets zipped before spending a build.

The admin's upload dropzone (single file, max **10 MB**) remains available and is still the only way to delete a function record.

### `ollieshop deploy` (advanced / CI)
Bundles a resource from the local cwd and uploads it to the builder directly. Exactly one of `--component-id` or `--function-id` is required. For components, **prefer Studio preview for day-to-day work**; for functions this is the primary path.

> **`deploy` ships code only — never `props`.** To set a component's default props from the CLI, pass them to `component create`; to change them later, use `component update --props` (see "Seeding `props` at create time" above). `ollie.json` component entries carry only `{ path, slot }`, so nothing prop-related is pushed on deploy.

Flags:
- `--component-id <uuid>` **or** `--function-id <uuid>` (one required, never both).
- `--name, -n <string>` (required) — local folder name to bundle (e.g. `FreeShippingBar`).
- `--wait` (optional) — poll the build until it reaches a terminal status.
- `--timeout <seconds>` (optional, default `300`) — only used with `--wait`. Must be a whole number of seconds, at least 1; anything else is rejected rather than silently polling forever or not at all. The same holds for `timeout` (and the boolean `wait`) inside `--data`.

```bash
ollieshop deploy --component-id <uuid> --name FreeShippingBar --wait
```

```bash
ollieshop deploy --function-id <uuid> --name auchan-promissory-keeper --wait
```

Exit code is non-zero if the build ends in any terminal status other than `SUCCEEDED`.

> **Bundle size is expected to exceed the folder you deployed.** For a function too: the bundle is the whole project root (see above), not just `functions/<name>/`, so a three-file function can still zip to a couple hundred KB. That is intentional — it's what lets a function import shared code from `utils/`, `lib/`, and friends. Trim it with `.ollieignore` if it matters.

---

## Builds

### `ollieshop status`
Fetches or polls a build status by id (the `id` returned by `deploy`).

Flags:
- `--build-id <string>` (required).
- `--wait` (optional) — poll until terminal status.
- `--timeout <seconds>` (optional, default `300`).

```bash
ollieshop status --build-id <build-id> --wait -o json
```

---

## Business rules

Business rules are **scoped to a store** and can optionally link to versions, components, and functions. They live in Ollie's backend and are used by the admin UI / agents to track which rules each customization implements. Currently in **alpha** — surface area is small but stable enough to script against.

### `ollieshop business-rule list` (alias: `business-rule ls`)
Flags (all optional):
- `--store-id <uuid>` — filter by store.
- `--version-id <uuid>` — filter by version.
- `--code-updated true|false` — filter by whether the rule's code has been updated since the last sync.

### `ollieshop business-rule get`
Flags:
- `--id <uuid>` (required).

### `ollieshop business-rule update`
Flags (or use `--data '{...}'` with camelCase keys):
- `--id <uuid>` (required).
- `--content, -c <string>` (required) — the rule body.
- `--versions-ids <json-array>` (optional) — linked versions, e.g. `'["<uuid>"]'`.
- `--components-ids <json-array>` (optional).
- `--functions-ids <json-array>` (optional).

```bash
ollieshop business-rule list --store-id <uuid> --code-updated true
ollieshop business-rule update --id <uuid> --content "Free shipping for orders over R$ 199" \
  --versions-ids '["<uuid>"]' --components-ids '["<uuid>"]'
ollieshop business-rule update --data '{"id":"<uuid>","content":"...","versionsIds":["<uuid>"]}'
```

---

## Schema introspection

### `ollieshop schema`
Lists every schema known to the CLI (e.g. `store`, `store.create`, `version.create`, `component.create`, `function.create`, `deploy.component`, `deploy.function`).

`deploy` has no subcommand, so its two entries are the two mutually exclusive shapes it accepts: `deploy.component` (takes `componentId`) and `deploy.function` (takes `functionId`). Passing both is rejected.

`deploy --data` also still accepts a legacy `{ resourceId, type }` shape, where `type` defaults to `component` when omitted or empty. Anything other than `component` or `function` is now rejected — `"Function"` used to be read as `component`, bundling a function from `components/`. It is kept for callers that predate `componentId`/`functionId` and is deliberately not in the schema — prefer the two shapes above for anything new.

### `ollieshop schema <resource>[.<action>]`
Prints the JSON Schema for that resource (or all actions on it). Use this whenever you need authoritative field names and constraints before scripting a mutation.

```bash
ollieshop schema                          # list all
ollieshop schema store.create             # one action
ollieshop schema function                 # all actions on function
```

---

## Other

### `ollieshop help`
Renders the same TUI help shown when running with no arguments.

### `ollieshop version`
Prints the installed CLI version.

---

## Typical end-to-end flow

```bash
ollieshop login
ollieshop store create --name "Pharma" --platform vtex --platform-store-id pharma
ollieshop version create --store-id <store-id> --name v1 --active
ollieshop init --store-id <store-id> --version-id <version-id>

# inside the project dir
ollieshop component create --version-id <version-id> --name FreeShippingBar --slot cart_summary_header
# link the returned UUID to ./components/FreeShippingBar/ — either
#   A) ./components/FreeShippingBar/meta.json  ("id": "<uuid>"), or
#   B) an entry in ./ollie.json -> components: "<uuid>": { "path": "...", "slot": "..." }

ollieshop start                 # iterate locally + deploy from Studio preview when ready
```

For functions, create the record via `function create` (remember `--invocation`), put the code in `./functions/<name>/`, then ship it with `ollieshop deploy --function-id <uuid> --name <name> --wait`.
