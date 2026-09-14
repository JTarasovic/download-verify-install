# Download, Verify & Install Release

A public composite GitHub Action that downloads a release asset from any GitHub
repository, **verifies its build-provenance attestation and SHA256 checksum
before extraction**, and installs the executable into a fresh, runner-temporary
directory. Verification failures are fatal — there is no bypass or fallback.

- Supports **Linux and Windows** runners.
- Supports **`.tar.gz` / `.tgz` and `.zip`** archives.
- Requires **no checkout of the producer repository** and no credentials beyond
  a token that can read the producer's release assets and public attestations.

## Usage

```yaml
- name: Install gh
  id: install
  uses: JTarasovic/download-verify-install@v1
  with:
    source-repository: cli/cli
    tag: v2.100.0
    asset: gh_2.100.0_linux_amd64.tar.gz
    checksum-asset: gh_2.100.0_checksums.txt
    executable-path: gh_2.100.0_linux_amd64/bin/gh
    expected-signer-workflow: cli/cli/.github/workflows/deployment.yml
    add-to-path: true
    token: ${{ github.token }}

- run: "${{ steps.install.outputs.executable }} --version"
```

Pin to an immutable release for stronger supply-chain guarantees:

```yaml
uses: JTarasovic/download-verify-install@<commit-sha>   # or @v1.2.3
```

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `source-repository` | yes | | Producer repository, as `<owner>/<repo>`. |
| `tag` | yes | | Explicit release tag (no `latest` resolution). |
| `asset` | yes | | Exact asset filename to download (`.tar.gz`, `.tgz` or `.zip`). |
| `executable-path` | yes | | Path to the executable inside the extracted archive. |
| `checksum-asset` | no | `""` | Release asset containing SHA256 checksums. |
| `expected-sha256` | no | `""` | Explicit expected SHA256 hex digest of the asset. |
| `expected-signer-workflow` | no | `""` | Workflow that must have signed the attestation, as `[host/]<owner>/<repo>/<path/to/workflow>`. |
| `expected-source-ref` | no | `""` | Git ref the attestation must be tied to (e.g. `refs/tags/v1.2.3`). |
| `expected-source-commit` | no | `""` | Source commit SHA the attestation must be tied to. |
| `add-to-path` | no | `false` | Add the executable's directory to `PATH`. |
| `token` | yes | | Token with read access to the producer's releases and attestations. |

Provide **at least one** of `checksum-asset` or `expected-sha256`; if neither is
given, the action fails rather than skip checksum verification. The attestation
is always verified against `source-repository`; supplying
`expected-signer-workflow`, `expected-source-ref`, and/or
`expected-source-commit` tightens the enforced producer identity and is strongly
recommended.

## Outputs

| Output | Description |
| --- | --- |
| `executable` | Absolute path to the installed executable. |
| `install-dir` | Absolute path to the fresh installation directory (under `RUNNER_TEMP`). |

## Verification model

1. **Download** the asset (and optional checksum asset) with `gh release download`.
2. **Attestation** — `gh attestation verify` against `source-repository`, additionally
   enforcing `--signer-workflow`, `--source-ref`, and `--source-digest` when provided.
   A missing or non-matching attestation fails the run.
3. **Checksum** — the asset's computed SHA256 must match `expected-sha256`, or the
   entry for the asset in `checksum-asset`. A missing or mismatched checksum fails the run.
4. Only after both pass is the archive **extracted** into a fresh directory and the
   executable located. Nothing is extracted or executed before verification.

## Permissions

The `token` needs read access to the producer's release assets and its public
attestations. For releases in public repositories, the default job token is
sufficient:

```yaml
permissions:
  contents: read
```

## Supported runners

`ubuntu-*` and `windows-*` GitHub-hosted runners (and equivalent self-hosted
runners). Both provide the `gh` CLI, `tar`, and `sha256sum` used by the action.

## Local development

Tooling is pinned with [mise](https://mise.jdx.dev/); tasks lint the workflows and
run the CI jobs locally with [act](https://github.com/nektos/act) (needs Docker).

```bash
mise install          # actionlint, shellcheck, gh, act
mise run lint         # actionlint (+ shellcheck on embedded run: scripts)
mise run test:success # install a real attested cli/cli release, assert it runs
mise run test:reject  # assert bad checksum / missing attestation / wrong identity are rejected
mise run check        # lint + both test jobs
```

The test tasks hit the real GitHub API and reuse your `gh auth` token
automatically (`gh` is provided by mise); export `GITHUB_TOKEN` to override.

The Windows matrix leg only runs on GitHub-hosted runners (act uses Linux containers).

## Releasing (maintainers)

Every push to `main` publishes a release. Commit messages follow
[Conventional Commits](https://www.conventionalcommits.org/): a breaking change
bumps major, `feat` bumps minor, anything else bumps patch — so every push
produces a new immutable `vX.Y.Z` tag and moves the `vX` convenience tag.
Consumers may pin by `vX`, `vX.Y.Z`, or commit SHA.

## License

[MIT](./LICENSE)
