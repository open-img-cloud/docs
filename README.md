<p align="center">
  <a href="https://openimages.cloud">
    <img src="https://openimages.cloud/img/og.png" alt="openimages.cloud: signed, boot-tested cloud images for OpenStack and Proxmox" width="720">
  </a>
</p>

<h1 align="center">Open Image Cloud documentation</h1>

<p align="center">
  <a href="https://github.com/open-img-cloud/docs/actions/workflows/links.yml"><img alt="Links" src="https://github.com/open-img-cloud/docs/actions/workflows/links.yml/badge.svg"></a>
  <a href="LICENSE"><img alt="License CC-BY-4.0" src="https://img.shields.io/badge/license-CC--BY--4.0-555"></a>
</p>

<p align="center">
  <a href="https://openimages.cloud"><b>Website</b></a> ·
  <a href="https://openimages.cloud/images/">Catalog</a> ·
  <a href="docs/getting-started.md">Getting started</a> ·
  <a href="https://openimages.cloud/verify/">Verify</a> ·
  <a href="https://github.com/open-img-cloud/docs/issues">Report an issue</a>
</p>

Open Image Cloud ([openimages.cloud](https://openimages.cloud)) publishes **signed, tested
qcow2 cloud images** for **OpenStack** and **Proxmox VE**, built in public CI from official
upstream releases. This repository holds the project documentation for users, contributors and
maintainers.

The catalog itself (images, current versions, sizes, download links) lives on the website and in
the machine-readable index. This documentation never copies it, because it changes every day:

| What | Where |
|---|---|
| Catalog, one page per image | <https://openimages.cloud/images/> (image pages: `https://openimages.cloud/images/<id>/`) |
| Verification guide | <https://openimages.cloud/verify/> |
| How images are tested | <https://openimages.cloud/quality/> |
| Why the project exists | <https://openimages.cloud/about/> |
| Image registry | <https://images.openimages.cloud> (files under `/<id>/<version>/` and `/<id>/latest/`) |
| Machine-readable catalog | <https://images.openimages.cloud/index.json> (schema `oic-index/v1`) |
| osism openstack-image-manager export | <https://images.openimages.cloud/osism/images.yml> |
| Source code | <https://github.com/open-img-cloud> (one repository per image, plus the platform repositories) |

## For users

- [Getting started](docs/getting-started.md): download an image, check it, import it into
  OpenStack Glance or Proxmox VE.
- [Verifying images](docs/verifying-images.md): cosign keyless signatures, checksums and the
  provenance manifest.
- [Image contract and quality checks](docs/image-contract.md): what every image guarantees and
  how it is tested.
- [Catalog index](docs/catalog-index.md): `index.json`, `versions.json`, `MANIFEST.json` and the
  osism export, for automation.
- [oic-sync](docs/oic-sync.md): keep Glance in sync with the catalog.
- [OpenStack service images](docs/service-images.md): Octavia amphora, Manila service image,
  Trove guest image.

## For contributors and maintainers

- [Architecture](docs/architecture.md): repositories, build pipeline, storage, CDN Worker,
  website, upstream watchers and release flow.
- [Contributing](docs/contributing.md): reporting problems, requesting an image, onboarding a
  new image, changing the shared platform.
- [Security policy](SECURITY.md).

## License

This documentation is licensed under
[Creative Commons Attribution 4.0 International](LICENSE) (CC-BY-4.0). The code in the other
repositories keeps its own license: GPL-2.0 for the image repositories, the shared
workflows and `images-router`, Apache-2.0 for the website and `oic-sync` (see each repository). Each image keeps its
upstream license. Distribution names and logos belong to their owners; the project is not
affiliated with them.
