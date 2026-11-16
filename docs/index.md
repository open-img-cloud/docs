# Open Image Cloud documentation

Open Image Cloud ([openimages.cloud](https://openimages.cloud)) publishes **signed, tested
qcow2 cloud images** for **OpenStack** and **Proxmox VE**, built in public CI from official
upstream releases. This documentation is for users, contributors and maintainers.

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

- [Getting started](getting-started.md): download an image, check it, import it into
  OpenStack Glance or Proxmox VE.
- [Verifying images](verifying-images.md): cosign keyless signatures, checksums and the
  provenance manifest.
- [Image contract and quality checks](image-contract.md): what every image guarantees and
  how it is tested.
- [Catalog index](catalog-index.md): `index.json`, `versions.json`, `MANIFEST.json` and the
  osism export, for automation.
- [oic-sync](oic-sync.md): keep Glance in sync with the catalog.
- [OpenStack service images](service-images.md): Octavia amphora, Manila service image,
  Trove guest image.

## For contributors and maintainers

- [Architecture](architecture.md): repositories, build pipeline, storage, CDN Worker,
  website, upstream watchers and release flow.
- [Contributing](contributing.md): reporting problems, requesting an image, onboarding a
  new image, changing the shared platform.
- [Security policy](https://github.com/open-img-cloud/docs/blob/main/SECURITY.md).

## License

This documentation is licensed under
[Creative Commons Attribution 4.0 International](https://github.com/open-img-cloud/docs/blob/main/LICENSE)
(CC-BY-4.0). Its sources are in [open-img-cloud/docs](https://github.com/open-img-cloud/docs).
