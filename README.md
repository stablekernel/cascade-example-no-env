# cascade-example-no-env

A minimal, runnable example of [cascade](https://github.com/stablekernel/cascade)
driving a no-environment (library / CLI) pipeline that goes straight from a
trunk merge to a release candidate, with no deploy lane.

## What this demonstrates

This repository exercises the no-env topology: a library-style project that
declares no environments at all. cascade watches trunk, and on every qualifying
change it:

- runs a build callback (`build-lib`),
- generates a changelog with a Contributors section, and
- mints a release-candidate draft.

Because there are no environments, cascade emits no deploy job: orchestrate
runs the build and mints the RC prerelease directly, and the run contains no
per-environment deploy lane.

The feature slice shown here:

- **No environments.** `config.environments` is empty, so no deploy callbacks
  are generated and no deploy job runs. State lives under an implicit
  `prerelease` key rather than per-environment keys.
- **Direct-to-prerelease orchestrate.** A trunk merge under `src/` mints a
  `rel-*-rc.*` draft with populated prerelease state and nothing else.
- **`tag_prefix: rel-`.** Tags are `rel-<version>` (for example `rel-1.0.0-rc.0`)
  instead of the default `v` prefix.
- **Contributor changelog.** Release notes include inline contributor
  attribution.
- **State loop.** cascade records the candidate version and committer back into
  the manifest under `ci.state.prerelease`.

## Layout

| Path | Role |
|------|------|
| `.github/manifest.yaml` | The cascade manifest. Edit this, then regenerate. |
| `.github/workflows/build-lib.yaml` | Stub build callback (echo and sleep). |
| `.github/workflows/orchestrate.yaml` | Generated. Runs on trunk changes; mints the RC draft. |
| `.github/workflows/promote.yaml` | Generated. Publishes the prerelease into a final release. |
| `.github/actions/manage-release/` | Generated composite action used by the workflows. |
| `.github/workflows/scenario-suite.yaml` | Hand-written end-to-end check of the no-env orchestrate path. |

## Regenerating the workflows

The orchestrate and promote workflows and the `manage-release` action are
produced by cascade from the manifest. After editing `.github/manifest.yaml`,
regenerate them:

```sh
cascade generate-workflow --config .github/manifest.yaml --force
```

The cascade CLI version is pinned through the `cli_version` field in the
manifest, which the generated workflows reference when they install the CLI.

## The scenario suite

`scenario-suite.yaml` is a hand-written driver that runs the no-env orchestrate
path against this repository and asserts the outcome. It resets to a clean
slate, commits a change under `src/` to start orchestrate, asserts that a
`rel-*-rc.*` draft and tag appear with populated prerelease state, and asserts
that no deploy job ran in the orchestrate run. It registers the orchestrate run
in the fleet ledger and ends in the shared fleet reconcile gate.

It runs on a daily schedule and on manual dispatch. It needs a
`CASCADE_STATE_TOKEN` repository secret with permission to push to trunk and
manage releases.
