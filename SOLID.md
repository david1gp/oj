# SolidJS support (fork: david1gp/oj, branch `solid-support`)

Status: **dev module pipeline works via the Vite plugin host; Start SSR is not yet supported.**

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

## What does NOT work yet

- `GET /` on a Start app 404s (`oj: no such file: /index.html`). oj's plain
  dev server needs an `index.html`; the TanStack `dev-server` plugin
  middleware answers 404 inside oj's shim (it needs more of Vite's
  Environment API than the shim provides).
- `oj build` on a Start app fails the same way (`no index.html`), and even
  past that, `oj build` / `start_build` / `start_host` hardcode
  `@tanstack/react-start` (server-entry, server-rpc, client-entry,
  `assets/start/*`). A Solid Start build needs parallel
  `@tanstack/solid-start` entries plus `target: "solid"` router codegen.
- `is_tanstack_start_app` (`crates/oj_server/src/lib.rs`) only matches
  `@tanstack/react-start`, deliberately: routing a Solid Start app into the
  React Start pipeline would fail worse than the current 404. Use
  `is_tanstack_solid_start_app` (same file) to detect the Solid flavor.

This branch adds, without changing any React behavior:

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

## How to verify (no browser needed)

```sh
cargo build -p oj
(cd ~/personal/homepage && ../oj/target/debug/oj dev --port 5199 . &)
curl -s http://localhost:5199/src/client.tsx | grep -c RefreshReg   # expect 0
curl -s http://localhost:5199/src/client.tsx | grep -o '_\$createComponent' | head -1
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:<mw-port>/  # plugin mw: 404
```

Or the browser e2e for plain Solid apps: `node e2e/solid.mjs`
(same shape as `e2e/svelte.mjs`: render, click-counter reactivity, HMR
hot-swap without reload, plus a production preview render).
**Status: passes** (`SOLID E2E PASSED`: `[dev] solid OK (+hot swap,
no reload)`, `[prod] solid OK`).

## Roadmap to full Solid Start support

1. **Dev SSR shell**: implement enough of Vite's Environment API in the
   plugin-middleware shim (`plugin-host.mjs`, `plugin_mw.rs`) for
   `tanstack-start-core:dev-server` to handle `/`, or add a native
   Solid Start dev path mirroring `crates/oj/src/start_dev.rs` with
   `@tanstack/solid-start/server-entry` + a `client-entry-solid.tsx`
   (`solid-js/web` `hydrate` + `@tanstack/solid-start/client`).
2. **Framework aliases**: parameterize `framework_aliases`
   (`crates/oj/src/start_host.rs:653`) and `rewrite_server_fns` (`:206`)
   on the Start flavor.
3. **Build**: `assets/start/*` Solid variants (`target: "solid"` in
   `generate.mjs`, solid `client-entry`/`server-entry`), `start_build`
   dispatch in `main.rs:618`.
4. **Resolve**: confirm the `"solid"` export condition reaches oj's native
   resolver via the merged vite config (`merge_resolve_block` in
   `crates/oj_server/src/plugins/extract.rs` adopts `resolve.conditions`
   from plugin `config` hooks, which is how `vite-plugin-solid` sets it);
   add a test once a dep requiring it is identified.
