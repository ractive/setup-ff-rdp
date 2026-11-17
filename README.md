# setup-ff-rdp

A composite GitHub Action that installs the prebuilt [`ff-rdp`](https://github.com/ractive/ff-rdp)
CLI on a runner and puts it on `PATH`. ff-rdp drives Firefox over its Remote
Debugging Protocol from the shell, so a job can launch headless Firefox, load
an unsigned WebExtension as a temporary add-on, open a page and assert on it.

The action is a thin wrapper around
[`ractive/release-workflows/setup`](https://github.com/ractive/release-workflows#installing-a-release-binary-setupactionyml),
the installer shared with [setup-hyalo](https://github.com/ractive/setup-hyalo):
pure `bash` + `curl`, plus the preinstalled `gh` CLI for provenance verification.

## Usage

Test a WebExtension against a page the job serves itself:

```yaml
name: Extension smoke test
on:
  pull_request:
jobs:
  smoke:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ractive/setup-ff-rdp@v1
      - name: Load the extension and check its content script ran
        run: |
          set -euo pipefail
          python3 -m http.server 8080 --bind 127.0.0.1 --directory test/pages &
          curl -fsS --retry 10 --retry-connrefused --retry-delay 1 -o /dev/null http://127.0.0.1:8080/fixture.html
          ff-rdp launch --headless --temp-profile --port 6011 \
            --temporary-addon ./extension http://127.0.0.1:8080/fixture.html
          trap 'ff-rdp --port 6011 stop' EXIT
          test "$(ff-rdp --port 6011 eval 'document.documentElement.dataset.myExt' --jq .results)" = '"ready"'
```

- Every ff-rdp command exits non-zero on failure and prints a JSON error, so
  `set -e` turns a bad manifest or a missing path into a failed job.
- `launch --temporary-addon` (ff-rdp v0.5.0 and later) installs the add-on
  before the page opens, so `document_start` content scripts run on the first
  load. Serve test pages over http yourself rather than depending on
  third-party sites.
- `stop` also removes the temporary profile. Give parallel jobs different ports.

[`.github/workflows/smoke.yml`](.github/workflows/smoke.yml) runs exactly this
against [`testdata/`](testdata) on every runner below.

## Firefox on the runner

ff-rdp looks for Firefox in `/Applications` (macOS), `Program Files`
(Windows) and on `PATH`; this action does not install Firefox.

__FIREFOX_TABLE__

Where it is missing, install it before `ff-rdp launch`, for example with
[`browser-actions/setup-firefox`](https://github.com/browser-actions/setup-firefox)
(pin it by commit SHA):

```yaml
      - uses: browser-actions/setup-firefox@0bc507ddf224827e3b1af68e014d5e42ab93e795 # v1.7.2
        with:
          firefox-version: latest
```

## Inputs

| Input                | Default               | Description |
| -------------------- | --------------------- | ----------- |
| `version`            | `latest`              | Release to install. `latest` tracks the newest release; pin a tag like `v0.5.0`. |
| `github-token`       | `${{ github.token }}` | Token for the release API (raises the anonymous rate limit) and the attestation lookup. |
| `verify-attestation` | `true`                | Verify the archive's build-provenance attestation before installing. See [Verification](#verification). |

## Outputs

| Output       | Description |
| ------------ | ----------- |
| `version`    | The resolved version tag that was installed (e.g. `v0.5.0`). |
| `provenance` | `verified`, `exempt` (no attestation was published for this release/target; warned) or `disabled`. See [Verification](#verification). |

```yaml
      - uses: ractive/setup-ff-rdp@v1
        id: ff-rdp
        with:
          version: v0.5.0
      - run: echo "installed $VERSION"
        env:
          VERSION: ${{ steps.ff-rdp.outputs.version }}
```

## Supported runners

| OS      | Architecture     | Release target                                     |
| ------- | ---------------- | -------------------------------------------------- |
| Linux   | x86_64           | `x86_64-unknown-linux-gnu`                         |
| Linux   | aarch64          | `aarch64-unknown-linux-musl` (no aarch64 gnu build) |
| macOS   | aarch64 (arm64)  | `aarch64-apple-darwin`; x86_64 macOS is unsupported |
| Windows | x86_64 / aarch64 | `*-pc-windows-msvc` (`.zip`); steps run in `bash`  |

Releases before v0.3.0 used unversioned archive names and cannot be installed
with this action.

## Verification

Every download is checked twice before it is installed:

1. **SHA256SUMS**: the archive's digest must match the release's
   `SHA256SUMS`. This catches corruption, but the sums file lives in the same
   release as the archive: anyone who can replace one can replace both.
2. **Build provenance**: `gh attestation verify` checks the archive's Sigstore
   attestation. It must have been built from `ractive/ff-rdp` **at the
   requested tag** (`--source-ref refs/tags/<version>`) by the shared
   `ractive/release-workflows/.github/workflows/release.yml` pipeline, on a
   GitHub-hosted runner. A rebuilt archive has no such attestation, and a
   genuine archive of another release swapped in carries the wrong tag;
   either way the step fails.

A missing attestation is a failure, not a warning, except where none was ever
published. That case emits a `::warning::` and installs after the checksum
check only:

- **`aarch64-unknown-linux-musl`** (Linux arm runners) **from v0.3.0 up to
  v0.4.4**: the musl archives are cross-built, and those releases attest no
  cross-built archive. From v0.5.0 every archive is attested, so `latest`
  must verify on every runner.

`gh` is preinstalled on all GitHub-hosted runners and uses `github-token`
(the workflow token by default, no extra `permissions:` needed). On a
self-hosted runner without `gh`, either install it or set
`verify-attestation: false`, which falls back to the checksum check alone.

The extracted binary is cached under `RUNNER_TOOL_CACHE` with a marker
recording how its provenance was established; a cache hit is reused only if
that marker satisfies the current run.

## Pinning

`@v1` follows the latest compatible release of the action. To pin exactly,
reference a full commit SHA:

```yaml
      - uses: ractive/setup-ff-rdp@<full-sha> # v1.0.0
```

Pinning the **action** (`@v1` / SHA) is independent of pinning the **binary**
(`version:` input). Pin both for fully reproducible CI.

## Versioning & release protocol

- The action is versioned independently of ff-rdp, so ff-rdp can release
  without retagging the action.
- Releases carry a full `vMAJOR.MINOR.PATCH` tag **and** a moving `vMAJOR`
  tag. Only move `vMAJOR` for backwards-compatible changes; a breaking change
  bumps it (`v2`).
- **After an ff-rdp release:** run this repo's `smoke` workflow
  (`gh workflow run smoke.yml -R ractive/setup-ff-rdp`) and check it passes:
  its `latest` jobs install the new release and drive Firefox with it.

## License

MIT, see [LICENSE](LICENSE).
