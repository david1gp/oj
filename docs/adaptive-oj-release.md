# adaptive-oj fork release

Independent crates.io lineage for the Solid-support fork while upstream
integration is pending. Workspace version stays `0.2.23`; this is a separate
package lineage, not a version bump. Do not publish crates, create tags, or
push from a docs-only change: the steps below are run explicitly by a human.

## Package map

All 15 publishable workspace packages rename `oj` / `oj_<suffix>` to
`adaptive-oj` / `adaptive-oj-<suffix>` (underscores become hyphens):

| was | now |
| --- | --- |
| `oj` | `adaptive-oj` |
| `oj_cache` | `adaptive-oj-cache` |
| `oj_compiler` | `adaptive-oj-compiler` |
| `oj_config` | `adaptive-oj-config` |
| `oj_css` | `adaptive-oj-css` |
| `oj_deno_napi` | `adaptive-oj-deno-napi` |
| `oj_deno_process` | `adaptive-oj-deno-process` |
| `oj_deno_runtime` | `adaptive-oj-deno-runtime` |
| `oj_deno_snapshots` | `adaptive-oj-deno-snapshots` |
| `oj_diag` | `adaptive-oj-diag` |
| `oj_env` | `adaptive-oj-env` |
| `oj_graph` | `adaptive-oj-graph` |
| `oj_js` | `adaptive-oj-js` |
| `oj_resolver` | `adaptive-oj-resolver` |
| `oj_server` | `adaptive-oj-server` |

Unchanged: the installed command stays `oj` (`[[bin]]` name), Rust library
targets and `use oj_*::` imports stay as-is, crate directory paths stay
`crates/oj_*`, and `oj_wasm` keeps its name with `publish = false`.

## Install

- Usable today, from a local checkout (no push or crates.io release needed):

  ```sh
  cargo install --path crates/oj --locked
  ```

- From git only after this branch is pushed (not usable from a local-only
  checkout):

  ```sh
  cargo install --git https://github.com/david1gp/oj --branch solid-support --locked adaptive-oj
  ```

- From crates.io only after the first successful publication:

  ```sh
  cargo install adaptive-oj --locked
  ```

## One-time bootstrap (all 15 new names at 0.2.23)

Each new name's very first publish cannot use Trusted Publishing yet (OIDC
cannot create new crates); do it
manually with a scoped token, then every crate joins the release workflow.
`cargo publish --workspace` resolves dependency order itself and skips
`publish = false` (`oj_wasm`), so one workspace invocation bootstraps every
new package. Use the installed toolchain's workspace publish support; no
token fallback in CI and no extra dependencies are needed.

Bootstrap version convention: publish all 15 crates manually at `0.2.23`
first, then push tag `v0.2.23`. That tag still creates the GitHub release and
still runs the snapshot-pin harvest, but the workflow's `publish` job skips
exactly `v0.2.23` (re-publishing would fail as duplicates).

Scoped token use only. Read exactly one variable from
`/home/david/leo_internal/dev-servers/david-server/env/env.conf`: never `source` or `eval` the
file (it may execute unrelated env code), never print the value, never run
with shell tracing (`set -x`), never pass the token as a command-line
argument, and keep it only in the environment of the single publish
subprocess (never persisted). The loader below requires exactly one exact
`CARGO_REGISTRY_TOKEN=` assignment with a nonempty value (optional `export`
prefix and single/double quoting supported); anything else fails closed:

```sh
python3 - /home/david/leo_internal/dev-servers/david-server/env/env.conf <<'EOF'
import os, re, shutil, subprocess, sys
pat = re.compile(r'^(?:export\s+)?CARGO_REGISTRY_TOKEN=(?:"([^"]*)"|\'([^\']*)\'|([^\s#]*))\s*(?:#.*)?$')
cargo = shutil.which("cargo") or os.path.expanduser("~/.cargo/bin/cargo")
try:
    vals = []
    for raw in open(sys.argv[1]).read().splitlines():
        m = pat.match(raw.strip())
        if m:
            vals.append(next(g for g in m.groups() if g is not None))
    if len(vals) != 1 or not vals[0]:
        raise ValueError("missing token")
    subprocess.run([cargo, "publish", "--workspace", "--locked", "--no-verify"],
                   env={**os.environ, "CARGO_REGISTRY_TOKEN": vals[0]}, check=True)
except Exception:
    print("CARGO_REGISTRY_TOKEN is not set", file=sys.stderr)
    sys.exit(1)
EOF
```

`--no-verify` skips the per-crate verification build (a standalone crate
build compiles `deno_v8` without the `v8` feature that only the full
workspace unifies; the same wall the manual fork publishes hit). It does not
prove the crates compile: the gate is the targeted local preflight checks,
not full CI. All of these have passed for this release (9 focused Rust tests,
5 JS unit tests, CLI `cargo check` plus all-15 package listings):

```sh
cargo metadata --locked --no-deps  # package identities, aliases, publish flags, no upstream leaks
cargo package --list --locked -p adaptive-oj  # repeat per crate / all 15 listings self-contained
cargo check --locked -p adaptive-oj
cargo test --locked -p adaptive-oj-server --lib  # focused relevant Rust tests only
node --test e2e/unit/start-client-transforms.test.mjs
```

Pushes to `solid-support`
intentionally run no fork CI — `ci.yml` push stays `main`-only so the fork
branch does not trigger the full expensive e2e suite — so do not claim full
CI green for the bootstrap; claim only the targeted gate above.

## Steady state: Trusted Publishing

OIDC still cannot create new crates, so the manual bootstrap above stays
required for each new name's very first publish. After the bootstrap, the
verified `david1gp` token already carries the trusted-publishing scope, so
the official API can configure each of the 15 new crates — no browser UI
required (it remains optional):

```sh
# List existing configs first, then create only the missing crates, one call each:
# POST /api/v1/trusted_publishing/github_configs
# {"github_config":{"crate":"<each adaptive-oj* name>","repository_owner":"david1gp","repository_name":"oj","workflow_filename":"release.yml","environment":null}}
```

From then on pushing a `v*` tag runs
`.github/workflows/release.yml`: it creates the GitHub release from the
`CHANGELOG.md` section, then publishes the workspace via OIDC with no stored
secret. The tag must equal the workspace version; the workflow's tag check
fails otherwise. Future releases: bump the workspace version first, then push
the matching tag (e.g. workspace `0.2.24` with tag `v0.2.24`); only the
one-time `v0.2.23` bootstrap tag skips the `publish` job.

## Snapshot pin note

`flake.nix` still pins the bundle from the upstream `snapshot-pins` release,
and the release workflow's staleness check fetches it from
`lovablelabs/oj`. That stays correct until the fork harvests its own pin;
when a fork release first attaches a fresh `tools/gen-snapshot-pin.sh`
bundle, point that fetch URL at `david1gp/oj`. For `v0.2.23` the deno
dependencies are unchanged, so the check is expected to report
not-stale and skip the harvest in seconds — no fork URL change needed.
