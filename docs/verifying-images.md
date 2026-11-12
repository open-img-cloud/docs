# Verifying images

The website guide is <https://openimages.cloud/verify/>; this page gives the same procedure and
the reasoning behind it.

## What a signature proves

Images are signed with [Sigstore](https://docs.sigstore.dev/) cosign in **keyless** mode. During
the release, GitHub Actions issues a short-lived OIDC token for the workflow that runs; the
Sigstore certificate authority (Fulcio) binds the signature to that workflow identity, and the
signing event is recorded in the public Rekor transparency log. A successful verification shows
that:

- the file is byte for byte what the build produced;
- it was signed by one of the reusable build workflows of
  [`open-img-cloud/.github`](https://github.com/open-img-cloud/.github), not by a person or by
  another repository;
- the signing event is publicly logged.

There is no key to download or rotate.

## Identity and issuer

Certificate identity (regular expression, `--certificate-identity-regexp`):

```text
^https://github.com/open-img-cloud/\.github/\.github/workflows/build-(libguestfs|dib|nix-flake|freebsd)-image\.yml@
```

OIDC issuer (`--certificate-oidc-issuer`):

```text
https://token.actions.githubusercontent.com
```

Bundle format: Sigstore bundle, published as `<file>.bundle` next to the image (pass
`--new-bundle-format`).

The four alternatives match the four build engines: libguestfs (customized upstream cloud
images), diskimage-builder (`dib`), Nix flakes, and FreeBSD builds. In a published bundle the
certificate subject is, for example,
`https://github.com/open-img-cloud/.github/.github/workflows/build-libguestfs-image.yml@refs/heads/main`.

The identity is the **reusable build workflow** in `open-img-cloud/.github`, not the image
repository: keyless signing records the workflow that actually ran the signing step. To tie a file
to a specific image repository commit, read its manifest (below).

These values are also published in `index.json` (`signing.certificate_identity_regexp` and
`signing.certificate_oidc_issuer`), which tools should read instead of hardcoding them. They are
defined in `oic-website/src/lib/images.ts` and in section A of the
[image standards](https://github.com/open-img-cloud/.github/blob/main/docs/IMAGE_STANDARDS.md).

## Steps

1. Install cosign 2.x or later: see the
   [installation guide](https://docs.sigstore.dev/cosign/system_config/installation/).
2. Download the image and its bundle from the same versioned directory (the image page on
   <https://openimages.cloud/images/> lists the exact URLs):

   ```sh
   URL=https://images.openimages.cloud/<id>/<version>/<file>
   curl -fLO "$URL"
   curl -fLO "$URL.bundle"
   ```

3. Verify:

   ```sh
   cosign verify-blob <file> \
     --bundle <file>.bundle --new-bundle-format \
     --certificate-identity-regexp '^https://github.com/open-img-cloud/\.github/\.github/workflows/build-(libguestfs|dib|nix-flake|freebsd)-image\.yml@' \
     --certificate-oidc-issuer https://token.actions.githubusercontent.com
   ```

   The output must be `Verified OK`.

## Provenance manifest

Each version directory has a `MANIFEST.json` (or `MANIFEST-<variant>.json`). It records the
version, sha256 and sha512, sizes, firmware, architecture, the upstream URL the image was built
from, the image repository commit, the CI run URL and the builder container image and digest:

```sh
curl -fLO https://images.openimages.cloud/<id>/<version>/MANIFEST.json
jq '{version, sha256, built_at, commit, build_url, builder}' MANIFEST.json
```

Open `build_url` to read the full build log, and `commit` in the image repository to see the
exact build scripts. The manifest is not signed itself: trust it after checking that its
`sha256` equals the digest of the file you verified with cosign. Field list:
[catalog index](catalog-index.md#manifestjson).

## Checksums only

Without cosign you can still detect a corrupted download:

```sh
curl -fsSL https://images.openimages.cloud/<id>/<version>/<file>.sha256 | sha256sum -c -
```

A checksum fetched from the same server proves nothing about who built the file: prefer the
signature.

## Automated verification

- [oic-sync](oic-sync.md) verifies the cosign bundle of every artifact against the sha256 of the
  index, with the identity and issuer from the index, before importing it.
- The [osism export](catalog-index.md#osism-openstack-image-manager-export) carries sha256
  checksums only: openstack-image-manager does not verify cosign signatures.
