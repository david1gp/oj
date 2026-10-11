# SolidJS support (fork: david1gp/oj, branch `solid-support`)

Status: **plain Vite+Solid fully works; TanStack Solid Start dev boots with
SSR shell + client hydration; Solid Start production build works end to end.**

## What works

- Plain Vite + Solid apps: `oj dev` serves with `solid-refresh` HMR
  (hot-swap, no reload) and `oj build` + preview render. Proven by
  `e2e/solid.mjs` (`SOLID E2E PASSED`).
- TanStack Solid Start dev (`~/personal/homepage`): `oj dev` routes into the
  Start pipeline, generates the route tree (`target: "solid"`), bundles the
  Solid client entry, and serves SSR HTML shell + route head + client
  hydration with zero errors. Edit → rebuild → reload works.
- TanStack Solid Start production: `oj build` emits `server.mjs` (+ client
  assets) and the built server renders the full page correctly (titles in
  Vite order, content, hydration, no errors).
- React behavior unchanged: `e2e/start.mjs` (dev + prod) and `e2e/svelte.mjs`
  pass; Rust + JS suites green.

## What already works (verified against `~/personal/homepage`, a TanStack Solid Start app)

`oj dev` loads the app's `vite.config.ts` and runs its plugins, including
`solid[pre]` (babel-preset-solid + solid-refresh) and the TanStack router
code-splitter. Observed serving `~/personal/homepage` with stock oj:

- `GET /src/client.tsx` → babel-preset-solid output (`_$createComponent`,
  `solid-js/web` imports). No `$RefreshReg$` / `jsx-dev-runtime` pollution:
  oxc sees no JSX after the plugin transform, so React Refresh never fires.
  (Pinned by `solid_plugin_output_never_gains_react_refresh` in
  `crates/oj_compiler/src/compile.rs`.)
- `GET /@solid-refresh` → 200 (the plugin's `resolveId`/`load` runtime).
- Solid components compile with solid-refresh HMR instrumentation
  (`_$$registry` / `_$$component`) intact.
- TanStack router transforms apply (`$$splitComponentImporter`,
  `?tsr-split=component`).
- Tailwind v4 plugin, dep pre-bundling (`/@oj-deps/solid-js_web.mjs`) unaffected.

So for **plain Vite + Solid apps** (with an `index.html`), `oj dev` should
already serve and hot-update: the solid plugin owns the JSX transform, oj owns
serving/HMR transport, and the two compose through the normal plugin hooks.

## Known gap: dev SSR body

In `oj dev`, the SSR document has the full head (meta, route head, JSON-LD)
but an empty body; the client renders all content on hydration (no errors,
page interactive). Production SSR renders the full body. Suspected area:
the dev runner path (embedded engine + streaming SSR), not resolution
(all `#src` and bare imports resolve uniformly to single server builds) or
transform (solid SSR output verified) or the route tree (byte-identical to
Vite's). Debugging record: route `head` runs, route `component` never
runs, throwing components surface as clean empty 200s (error swallowing
worth fixing regardless), React Start renders fully under the same runner.

## Roadmap (remaining)

1. **Dev SSR body** (above): route components never execute in the dev
   runner although matching/head succeed. Prime suspect is the embedded
   engine's streaming SSR behavior (production Node renders the same code
   fully); needs engine-level observability (runner console forwarding +
   SSR error surfacing).
2. **SSR error surfacing**: render throws become clean empty 200s. Surface
   them (500 + message) the way Vite does.
3. **Resolve**: confirm the `"solid"` export condition reaches oj's native
   resolver via the merged vite config (`merge_resolve_block` in
   `crates/oj_server/src/plugins/extract.rs` adopts `resolve.conditions`
   from plugin `config` hooks, which is how `vite-plugin-solid` sets it);
   add a test once a dep requiring it is identified.

This branch adds, without changing any React behavior:

- `StartFramework` (`crates/oj_server/src/lib.rs`): `start_framework()`
  detection, `start_package()` / `router_target()` / entry filenames;
  `main.rs` dispatches dev + build on it; `start_dev`/`start_build` keep
  React wrappers (`*_with_framework` do the work).
- Framework-threaded Start pipeline: `framework_aliases`,
  `rewrite_server_fns`, `StartHost::new`, `StartEngine::new`,
  `script_env`/`start_script_env` (`OJ_START_PKG`, `OJ_ROUTER_TARGET`,
  `OJ_CLIENT_ENTRY`, `OJ_SERVER_ENTRY_FILE`), per-framework codegen and
  bundle cache keys (`route-tree-solid`, framework-salted bundle store).
- `assets/start/`: `client-entry-solid.tsx`, `server-entry-solid.tsx`,
  `start-pkg.mjs` + `solid-transform.mjs` (babel-preset-solid for bundles:
  dom/ssr + hydratable, rolldown JSX preserve, `solid`-first conditions);
  `build.mjs`, `bundle-client.mjs`, `cf-build.mjs`, `generate.mjs`
  (`target`, footer, fallbacks), `vite-plugin-bridge.mjs` read them.
- `is_tanstack_solid_start_app()` + tests (`adapter_tests.rs`).
- A `oj dev` startup note when a Solid Start app is detected, and a hinted
  `oj build` error, so the 404 is not a mystery.
- The compiler regression test above.
- **`vite:alias` fallback in the plugin `resolveId` chain
  (`plugin-host.mjs` `resolveIdFull`, test
  `e2e/unit/resolve-alias-fallback.test.mjs`):** when no `resolveId` hook
  claims an id, the merged config's `resolve.alias` (string finds and
  RegExps, including entries from plugin `config` hooks) rewrites it.
  Without this, `vite-plugin-solid`'s `/^solid-refresh$/` →
  `/@solid-refresh` alias never applied and every Solid component 500'd
  (`Failed to resolve import "solid-refresh"`). Plugin claims still win
  over aliases, as in Vite.
- **Dep-aware imports in plugin-loaded source (`plugin_routes.rs`
  `dep_pre_resolver`):** `serve_plugin_id` / `serve_plugin_load_fallback`
  now resolve bare imports through the optimized dep map like app modules
  do (previously `|_| None`). Without this, the `solid-refresh` runtime
  imported `solid-js` from raw `/node_modules/.../dev.js` while the app
  used the prebundled `/@oj-deps/solid-js.mjs` — two Solid instances
  (the "multiple instances" warning) and silently dead HMR registries:
  updates applied (`hot updated`) but never re-rendered.

## How to verify

```sh
cargo build -p oj
# plain Solid app e2e (render, reactivity, HMR hot-swap, prod preview):
node e2e/solid.mjs
# Solid Start dev on the homepage:
(cd ~/personal/homepage && ../oj/target/debug/oj dev --port 5199 . &)
curl -s http://localhost:5199/ | head -c 300   # SSR shell + head
# Solid Start production build:
(cd ~/personal/homepage && ../oj/target/debug/oj build . --out /tmp/oj-solid-dist)
(cd /tmp/oj-solid-dist && node server.mjs)     # serves fully-rendered page
```

## Roadmap (remaining)

1. **Dev SSR body** (above): route components never execute in the dev
   runner although matching/head succeed. Prime suspect is the embedded
   engine's streaming SSR behavior (production Node renders the same code
   fully); needs engine-level observability (runner console forwarding +
   SSR error surfacing).
2. **SSR error surfacing**: render throws become clean empty 200s. Surface
   them (500 + message) the way Vite does.
3. **Resolve**: confirm the `"solid"` export condition reaches oj's native
   resolver via the merged vite config (`merge_resolve_block` in
   `crates/oj_server/src/plugins/extract.rs` adopts `resolve.conditions`
   from plugin `config` hooks, which is how `vite-plugin-solid` sets it);
   add a test once a dep requiring it is identified.
