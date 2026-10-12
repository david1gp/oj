# Adaptive oj fork release

## Goal
Complete an independent crates.io release of the Solid-support fork as `adaptive-oj` while upstream integration is pending. Preserve the installed executable name `oj` and existing Rust import names. The user has authorized committing, pushing, publishing, creating the release tag, and verifying the install autonomously. Do not start development servers.

## Decisions
- Checkout: `/home/david/adaptive/oj`, branch `solid-support`.
- `adaptive-oj` was absent from crates.io when checked. Internal names must also be checked before release.
- Retain workspace version `0.2.23`; this is a separate package lineage.
- Rename all 15 publishable workspace packages to `adaptive-oj` / `adaptive-oj-*`. Leave nonpublishable `oj_wasm` unchanged.
- Preserve internal dependency keys and actual existing library target names through explicit package aliases and library names. No Rust source import renaming.
- Point workspace repository metadata at `https://github.com/david1gp/oj`.
- Load `CARGO_REGISTRY_TOKEN` from `/home/david/leo_internal/dev-servers/david-server/env/env.conf` only for scoped release commands, without printing or persisting its value. Future CI may use crates.io Trusted Publishing configured for the fork and each new crate.
- Use existing project tooling. Do not add dependencies or run the full workspace test suite or e2e tests.

## Tasks
1. Complete — Rename publishable package identities and internal package aliases, preserve library/binary identities, update workspace metadata and lockfile. Audit resulting dependency graph using targeted metadata checks. Scope: manifests and Cargo.lock only.
2. Complete — Align existing CI/release configuration and document install, bootstrap publishing, and Trusted Publishing for the fork. Update affected Cargo package selectors in scripts/documentation and e2e harness commands; avoid unrelated rewrites. Scope: workflows, `tools/gen-snapshot-pin.sh`, README/development docs, e2e package selectors, and concise release documentation / SOLID.md installation guidance.
3. Complete — Independently review manifest/lockfile/workflow changes and perform targeted packaging checks feasible before the new dependencies are on crates.io. Validate no upstream internal package identities leak into registry dependencies. Report limitations without publishing or running broad builds.
4. Complete — Remove CLI compile-time reliance on sibling crate source files. Expose the exact existing embedded Start assets from their owning `oj_server` crate and consume them from the CLI without changing script names, bytes, iteration order, or runtime behavior. Verify via focused unit checks and targeted compilation; no full workspace tests or e2e.
5. Complete — Align the excluded fuzz manifest's internal dependency package aliases; preserve existing Rust import keys and verify lightweight fuzz metadata.
6. Complete — Perform release preflight on the existing changes, confirm publish authentication, and ensure packaged files are self-contained. Use targeted tests and checks, not broad e2e runs. Correct concrete release defects only.
7. Complete — Commit the Solid-support release changes in logical conventional commits and push `solid-support`. Use the commits skill. No PR is required for the fork release.
8. In progress — Bootstrap publish the 15 new packages at `0.2.23` with the external scoped token, verifying registry results and resuming safely after any partial success. Do not expose or persist credentials.
9. Pending — Push the matching `v0.2.23` tag, ensure the GitHub release exists and relevant Actions jobs succeed, and verify the crates.io package installs an `oj` executable. Configure future Trusted Publishing if available with existing access; accurately document any external account setup remaining.

## Verification approach
- Locate Cargo through existing development tooling if it is not on PATH.
- Run metadata with `--locked --no-deps` and validate package identities, preserved targets, aliases, publish flags, and dependency versions/paths.
- Use package listing and other lightweight package checks rather than full compilation. A dry-run publishing command may depend on unpublished registry dependencies; assess that before running it and do not confuse this with a complete release verification.
- Keep credentials secret; report presence only.

## Current context
Read-only exploration completed. Workspace contains 16 members; `oj_wasm` alone is nonpublishable. Solid support changes multiple internal crates, so replacing only the CLI package identity would not deliver the fork implementation through crates.io.
The existing release workflow uses workspace publishing and OIDC. Retain this authentication approach; document manual first-publish bootstrap rather than adding a token-secret fallback. Keep selectors for the unchanged `oj_wasm` package intact.
Implementation and targeted release preflight are complete. Four logical commits have been pushed to `origin/solid-support`; the external token is valid for `david1gp` and supports the official Trusted Publishing configuration API. Bootstrap publication is in progress using Cargo 1.95 workspace publishing. The plan's status update is the only intended worktree change and must not block publishing the committed package files. No fork `v0.2.23` tag or release exists yet.
