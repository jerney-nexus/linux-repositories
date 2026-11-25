# linux-repositories

Central publisher for the signed apt and dnf repositories served at
<https://packages.jerney.nexus>. Every application that ships `.deb` or `.rpm`
packages publishes through this repository, so no application needs its own
subdomain, signing key, or R2 credentials.

## How publishing works

1. An application release workflow mints a GitHub App token for this repository
   and sends a `repository_dispatch` event named `publish-linux-packages` with
   `client_payload.sources` set to `owner/repo@tag`.
2. `.github/workflows/publish.yml` checks that the source is listed in
   [allowed-sources.txt](allowed-sources.txt). Unlisted sources are rejected.
3. The workflow downloads the `*.deb` and `*.rpm` assets of that release,
   mirrors the current repositories from R2, adds the new packages, signs the
   metadata with the shared GPG key, and uploads the result back to R2.

Each downloaded package is verified against the SHA-256 digest GitHub recorded
for that release asset (`gh release view --json assets`). A missing digest or a
mismatch fails the run before anything is signed or uploaded. This check works
for every existing release, including older ones, because it needs no change in
the source repository. It proves the bytes are the ones GitHub stored for the
release; it does not prove which workflow built them.

Packages are stored under their metadata names, such as
`epikrise_<version>_<arch>.deb` and `epikrise-<version>-1.<arch>.rpm`, not the
build-target names of the release assets.

Dispatches are only accepted from the default branch, so `publish.yml` must
stay on `main`. Publish runs share the `linux-repository` concurrency group and
are never cancelled mid-upload.

## Adding an application

1. Open a reviewed PR that adds the `owner/repo` line to `allowed-sources.txt`.
2. Install the publisher GitHub App on the application's repository, or make
   sure the App has read access to its releases.
3. In the application's release workflow, mint a token with the App and dispatch
   `publish-linux-packages` to this repository, as Epikrise does in
   `jerney-nexus/epikrise` `.github/workflows/signed-release.yml`.

Packages from different applications share one signing key and one repository.
Two applications that publish the same package name and version overwrite each
other, so give each application distinct package names.

## Configuration

Environment `linux-repository`:

| Name | Kind | Purpose |
| --- | --- | --- |
| `LINUX_REPOSITORY_APP_ID` | variable | ID of the GitHub App used to read source releases |
| `LINUX_REPOSITORY_APP_PRIVATE_KEY` | secret | Private key of that App, used to mint read tokens for source releases |
| `LINUX_REPOSITORY_GPG_PRIVATE_KEY` | secret | Passphrase-less signing key, ASCII-armored |
| `LINUX_REPOSITORY_GPG_KEY_ID` | variable | Fingerprint of the signing key |
| `LINUX_REPOSITORY_R2_ENDPOINT` | secret | R2 S3 endpoint for the account |
| `LINUX_REPOSITORY_R2_ACCESS_KEY_ID` | secret | R2 access key with object read/write on the bucket |
| `LINUX_REPOSITORY_R2_SECRET_ACCESS_KEY` | secret | Secret of that R2 access key |
| `LINUX_REPOSITORY_R2_BUCKET` | variable | Bucket name, `linux-repositories` |

Configure required reviewers on the `linux-repository` environment so that a
publish waits for approval.

## Local checks

- `actionlint` on `.github/workflows/*.yml`
- `shellcheck scripts/*.sh`

Keep the builder in `scripts/build-linux-repositories.sh` compatible with the
packaging conventions in the Epikrise repository: `.deb` built with
`dpkg-deb`, `.rpm` built with `rpmbuild`, and signatures created with the
shared key.
