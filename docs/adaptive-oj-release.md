# adaptive-oj fork release

Independent crates.io lineage for the Solid-support fork while upstream
integration is pending. Workspace version stays `0.2.23`; this is a separate
package lineage, not a version bump. Do not publish crates, create tags, or
push from a docs-only change: the steps below are run explicitly by a human.

## Package map

Final release set: seven fork packages at `0.2.23` — CLI, cache, compiler,
resolver, server, env, config:

| was | now |
| --- | --- |
| `oj` | `adaptive-oj` |
| `oj_cache` | `adaptive-oj-cache` |
| `oj_compiler` | `adaptive-oj-compiler` |
| `oj_config` | `adaptive-oj-config` |
| `oj_env` | `adaptive-oj-env` |
| `oj_resolver` | `adaptive-oj-resolver` |
| `oj_server` | `adaptive-oj-server` |

Reused as-is from the registry at verified `0.2.22` (coherent upstream
JS/NAPI/process train; process hooks and NAPI exports stay on upstream
identities, never mixed with fork ones): `oj_graph`, `oj_js`, `oj_css`,
`oj_diag`, `oj_deno_napi`, `oj_deno_process`, `oj_deno_runtime`, and
`oj_deno_snapshots` (underscore names, `0.2.22`).

Already-published but unused: six earlier `adaptive-oj-*` names
(`adaptive-oj-graph`, `adaptive-oj-js`, `adaptive-oj-css`, `adaptive-oj-diag`,
`adaptive-oj-deno-napi`, `adaptive-oj-deno-runtime`) remain on crates.io
without yanking and keep their existing Trusted Publishing configs, but the
CLI release graph does not depend on them and release scripts never select
them. Their publishable local members stay in the workspace with
`publish = false`, still named `adaptive-oj-*`, so local `-p adaptive-oj-*`
test/build selectors keep working.

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

## One-time bootstrap (seven required names at 0.2.23)

Each new name's very first publish cannot use Trusted Publishing yet (OIDC
cannot create new crates); do it
manually with a scoped token, then every crate joins the release workflow.
Select the seven required fork crates explicitly in one Cargo 1.95+ invocation
with repeated `-p` flags — never `--workspace` (keep unused local members out
of the release). Cargo packages the selected set up front using its temporary
registry overlay, then schedules uploads in dependency order. Do not replace
this with seven separate publish commands: repeated packaging setup wastes
time, especially within the future OIDC token's 30-minute lifetime.

Bootstrap version convention: publish the seven required crates manually at `0.2.23`
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
        raise ValueError("invalid token configuration")
    token = vals[0]
except Exception:
    print("Unable to load CARGO_REGISTRY_TOKEN", file=sys.stderr)
    sys.exit(1)
env = {**os.environ, "CARGO_REGISTRY_TOKEN": token}
env["PATH"] = os.path.dirname(os.path.realpath(cargo)) + os.pathsep + env.get("PATH", "")
# Seven required fork crates; Cargo 1.95+ accepts repeated -p in one invocation.
# A partial success resumes by dropping the names already on crates.io
# (cargo refuses duplicates as an error, not a skip).
pkgs = ["adaptive-oj-env", "adaptive-oj-config", "adaptive-oj-cache",
        "adaptive-oj-compiler", "adaptive-oj-resolver", "adaptive-oj-server",
        "adaptive-oj"]
args = [cargo, "publish", "--locked", "--no-verify"]
for pkg in pkgs:
    args.extend(["-p", pkg])
rc = subprocess.call(args, env=env)
if rc != 0:
    print("publish failed; check crates.io and resume with only unpublished names", file=sys.stderr)
    sys.exit(rc)
EOF
```

Publication is not atomic: a failed upload or index-poll timeout can leave
some crates published. Check each selected name/version on crates.io before
retrying and remove successful names from `pkgs` (or the workflow's `-p` list).
The helper propagates Cargo's failure status; GitHub Actions' default Linux
shell uses `bash -e`, so the workflow also fails rather than silently continuing.

`--no-verify` skips the per-crate verification build (a standalone crate
build compiles `deno_v8` without the `v8` feature that only the full
workspace unifies; the same wall the manual fork publishes hit). It does not
prove the crates compile: the gate is the targeted local preflight checks,
not full CI. All of these have passed for this release (9 focused Rust tests,
5 JS unit tests, CLI `cargo check` plus per-crate package listings for the
seven required crates):

```sh
cargo metadata --locked --no-deps  # package identities, aliases, publish flags, no upstream leaks
cargo package --list --locked -p adaptive-oj  # repeat per required crate (seven listings self-contained)
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
the official API can configure each of the 7 required crates — no browser UI
required (it remains optional). The six already-published unused
`adaptive-oj-*` names keep their existing publisher configs; do not remove
them:

```sh
# List existing configs first, then create only the missing crates, one call each:
# POST /api/v1/trusted_publishing/github_configs
# {"github_config":{"crate":"<each adaptive-oj* name>","repository_owner":"david1gp","repository_name":"oj","workflow_filename":"release.yml","environment":null}}
```

From then on pushing a `v*` tag runs
`.github/workflows/release.yml`: it creates the GitHub release from the
`CHANGELOG.md` section, then publishes the seven required fork crates via
OIDC with no stored secret. The tag must equal the workspace version; the workflow's tag check
fails otherwise. Future releases: bump the workspace version first, then push
the matching tag (e.g. workspace `0.2.24` with tag `v0.2.24`); only the
one-time `v0.2.23` bootstrap tag skips the `publish` job.

## Snapshot pin note

`flake.nix` still pins the bundle from the upstream `snapshot-pins` release,
and the release workflow's staleness check fetches it from
`lovablelabs/oj`. That stays correct: the active release graph reuses
registry `oj_deno_snapshots` 0.2.22, so `tools/gen-snapshot-pin.sh` harvests
the upstream `package_id` (`#oj_deno_snapshots@0.2.22`) from the
`cargo build --release -p adaptive-oj` output, not the unused publish = false
local `adaptive-oj-deno-snapshots` member. For `v0.2.23` the deno
dependencies are unchanged, so the check is expected to report
not-stale and skip the harvest in seconds — no fork URL change needed.
