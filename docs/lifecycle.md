# Versions, retention and end of life

How long images stay available, how new versions arrive, and what happens when an upstream
release reaches its end of life.

## Versions and URLs

Each build of an image is a **version**, published under its own path:

```
https://images.openimages.cloud/<image>/<version>/<file>.qcow2   immutable, kept
https://images.openimages.cloud/<image>/latest/<file>.qcow2      moves to each new version
https://images.openimages.cloud/<image>/versions.json             every published version
```

- Use a **versioned URL** to pin an image (reproducible deployments, Terraform, audits): it
  always serves the same file, cached for a year.
- Use **`latest/`** to always get the newest version: it is cached for 5 minutes and moves at
  each release.
- `versions.json` lists every published version with its digests, and
  [`index.json`](catalog-index.md) the latest version of every image.

## Retention

**Every published version is kept, with no time limit.** Nothing is deleted when a newer
version is published or when the release reaches its end of life: a versioned URL that worked
once keeps working.

A version is not meant to change after publication; a fix ships as a new version. In the rare
case a version is republished (a broken publication), its qcow2, checksums, signature and
manifest are replaced together and the CDN cache is purged: always check a file against the
`MANIFEST.json` and the cosign bundle of the same version, as [Verifying images](verifying-images.md)
explains. Replaced copies are kept in the storage for 7 days, then expire.

If this policy ever has to change (storage cost, an upstream license), the change will be
announced on this page in advance.

## New versions

- An **upstream watcher** checks most images daily and opens a pull request when upstream
  publishes a new release or point release.
- **Merging is releasing**: the image is built, boot-tested on a real OpenStack cloud, signed
  and published, then `latest/` moves to it.
- The [catalog](https://openimages.cloud/images/), `index.json` and the image pages are rebuilt
  daily and after each release. The [status page](https://openimages.cloud/status/) shows when
  each image was last built and how its last release and watch runs went.
- Rolling distributions (Arch Linux, Gentoo) and images that follow upstream releases (Alpine,
  openSUSE Leap, NixOS) move to the next release in place; the other images have one catalog
  entry per major release (Debian 12 and 13, Ubuntu 22.04 and 24.04...).

## End of life

Support dates come from [endoflife.date](https://endoflife.date) at each catalog build and are
shown on each image page, on the status page and in `index.json` (`support` block).

| When | What happens |
|---|---|
| 6 months before the end of upstream security support | **EOL soon** badge on the catalog, the image page and the status page; `oic-sync --eol warn` (the default) warns |
| End of upstream security support | **EOL** badge; the image is moved to the **Archive** status with a banner pointing to the successor |
| After | Files stay downloadable (see Retention). Builds stop, unless upstream still ships extended-support updates, in which case they continue for existing users |

New deployments should move to the successor (for example Debian 12 to Debian 13). Automation
can enforce it: `oic-sync --eol skip` leaves end-of-life images out of a sync, `--eol fail`
stops on them. The [Ansible](ansible.md) role `glance_image` has the same choice
(`glance_image_eol`), and the [Terraform provider](terraform.md) exposes `end_of_life` and
`end_of_life_soon` to filter on.

## OpenStack series images

The Octavia amphora, Manila service image and Trove guest image are built per OpenStack series
(see [OpenStack service images](service-images.md)).

- Every series that is **maintained upstream** (from the OpenStack release team data) is
  published and offered in the catalog; `latest/` follows the newest series.
- A series that leaves upstream maintenance stays published unchanged, so existing deployments
  keep working, but it is no longer rebuilt; the catalog marks it "unmaintained upstream".
